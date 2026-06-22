# Calibre Pattern Matching 底层实现算法栈综述

## 从轮廓签名到边驱动矩形切割：三代算法的技术演进与工程实践

---

**摘要：** 本文系统梳理了 Calibre Pattern Matching 工具所依赖的核心算法体系。从 Niewczas (1998/1999) 的**轮廓签名等距不变匹配**出发，经 US8326018 (2008) 的**锚边加速框架**，到 Park (2016) / US10311199 的**边驱动矩形切割 (EDDR PM)** 及向量空间加速，再到 US20180307791 的**上下文感知匹配**，本文完整呈现了该技术的演进脉络。同时讨论 Calibre PDL (Pattern Description Language) 如何通过 SVRF/DRC 语法将这些算法暴露给工程应用。

---

## 1. 引言与背景

集成电路制造进入深亚微米节点后，光刻邻近效应、CMP 不平坦度、蚀刻偏差等**工艺热点 (hotspot)** 成为良率的主要威胁。传统 approach 依赖 DRC (Design Rule Checking) 以规则集形式检测违反，但 DRC 是**局部规则驱动**的，无法有效捕捉**全局几何上下文**中出现的复杂工艺退化。

**Calibre Pattern Matching** 由 Mentor Graphics (现 Siemens EDA) 于 2000 年代推出，核心思想截然不同：

> 给定一个已知的 hotspot "pattern"（几何形状 + 上下文），在整个版图中查找所有**匹配或近似匹配**的位置。

这一技术避开了复杂的物理仿真，将良率问题转化为**版图子图同构或近似子图同构问题**，在计算效率上较全芯片 EBL (电子束光刻) 仿真快 3-5 个数量级。

本文追踪该技术从 1998 到 2018 二十年的算法演进。

---

## 2. 理论基础：轮廓签名法 (Niewczas 1998/1999)

Niewczas 等人在 ISPD 1998 和 TCAD 1999 上提出了首个专门面向 LSI 版图的**等距不变 (isometry-invariant) 几何匹配算法**，构成了 Calibre Pattern Matching 的理论开端。

### 2.1 多边形 → 轮廓表示

版图上的任意多边形 P 可表示为有向轮廓：

```
P → {e₁, e₂, ..., eₙ}   (n = 边数)
```

每条边 eᵢ 携带两类信息：
- **边角度** θᵢ：边 eᵢ 的朝向（相对全局坐标系）
- **线段长度** lᵢ：边 eᵢ 的长度

### 2.2 角度-线段序列 (Angle-Segment Sequence)

多边形轮廓的**转角序列**定义为相邻边之间的外角：

```
αᵢ = θᵢ₊₁ − θᵢ  (mod 2π)
```

由此可构建**规范化的角度-线段序列**（也称 contour signature）：

```
S(P) = (α₁, l₁), (α₂, l₂), ..., (αₙ, lₙ)
```

其关键性质：
- **循环不变性**：无论从哪条边开始遍历，序列只是循环移位
- **旋转不变性**：全局旋转所有边 → 所有 αᵢ 不变（多边形内角是旋转不变量）
- **平移不变性**：轮廓序列不依赖于 polygon 在芯片上的绝对坐标

### 2.3 等距不变归一化

真正的挑战来自**镜像**：镜像翻转后的多边形具有不同的角度符号序列。

Niewczas 的解决方案：
1. **生成 8 种变换**：4 种旋转 (0°, 90°, 180°, 270°) × 2 种镜像 (原始/翻转) = 8
2. 对每种变换构造角度-线段序列
3. **字典序最小**的那个序列作为该多边形的**规范签名 (canonical signature)**

```
signature(P) = min_lex{ S(T₁(P)), S(T₂(P)), ..., S(T₈(P)) }
```

其中 Tᵢ 遍历 8 种正交变换。

### 2.4 Pattern DB 与 Instance DB

算法采用双层数据库架构：

**Pattern Database (pDB)**：
- Key = 规范签名的 hash
- Value = 与该签名对应的 pattern 描述
- 每个 pattern 对应一个**等价类** (equivalence class)，包含所有在等距变换下相同的几何形状

**Instance Database (iDB)**：
- 记录每个 pattern 在版图中出现的**所有实例位置**
- 坐标 + 所使用的变换（旋转 / 镜像）
- 用于后续的 DRC 集成、hotspot 分布密度分析

### 2.5 匹配流程

```
给定 query polygon Q:
  1. 提取 Q 的轮廓 → angle-segment sequence
  2. 8 变换 → 字典序最小 → canonical signature
  3. Hash lookup in pDB → 找到等价类
  4. 该等价类中的所有 pattern → 在 iDB 中查找 Q 的位置
```

### 2.6 优缺点分析

| 优势 | 不足 |
|------|------|
| 严格的数学基础 | 仅支持**精确匹配**，无容差/模糊机制 |
| O(n) 签名构造，n = 边数 | 对不规则多边形效率低 |
| 8 变换暴力枚举，小规模可接受 | 每多边形需 8 次序列构造 |
| 数据库结构成熟 | 不支持多层/层次化版图 |

> **结论**：Niewczas 算法为 Calibre Pattern Matching 提供了理论奠基，但工程化落地仍需解决速度、容差和层次化匹配的痛点。

---

## 3. 锚边加速匹配 (US8326018, Filed 2008)

### 3.1 核心思想

US8326018 (Mentor Graphics) 提出了一种**基于锚边 (anchor edge) 的加速策略**，核心洞察：

> 在 pattern 的诸多边中，**最与众不同的那条边**可以作为匹配的入口，仅凭它在搜索窗口中的位置即可完成粗筛。

### 3.2 锚边选择算法

对 pattern P 的每条边 eᵢ：

```
score(eᵢ) = count(在搜索窗口内, 与 eᵢ 同方向同长度的边)
```

选择 **score 最小**的边作为 anchor edge。

```
anchor = argmin_eᵢ score(eᵢ)
```

这意味着：
- 该边在版图中出现次数最少
- 用它作为"指纹"可最大程度减少候选位置数

### 3.3 Pseudo-Anchor 边

当 pattern 全部由常见单元构成（如标准单元库中的重复边），**没有一条边具有足够区分度**时：

- 算法构造**伪锚边** (pseudo-anchor edge)
- 方法：选择两条相邻边，联合构成一个 L 形 corner，作为复合锚
- 该 corner 的出现频率通常低于单边

### 3.4 快速拒绝条件

锚边确定后，对每个候选位置，算法在增量匹配前依次应用快速拒绝：

1. **Extent 比较**：pattern 的包围盒与候选位置延伸区域的大小对比
2. **边计数**：pattern 的总边数 ≠ 候选位置的总边数 → 拒绝
3. **锚边定位**：锚边位置 / 方向不匹配 → 拒绝
4. **Secondary edge 检查**：选择第 2、3 条区分度边验证
5. **全匹配**：仅少数候选需进入全面边对边的比较

### 3.5 搜索窗口划分

专利将搜索区域划分为**矩形网格** (grid)，每个 pattern 的锚边属性与网格单元关联：

```
grid_cell(x, y) → list of (pattern_id, anchor_position, orientation)
```

匹配时只需搜索与 query 位置对应的 grid cell，避免全局扫描。

### 3.6 优缺点分析

| 优势 | 不足 |
|------|------|
| 大幅减少候选位置数 | 锚边选择本身开销不可忽略 |
| 快速拒绝链简单高效 | 对高度重复的单元库效果有限 |
| Grid 分区天然支持并行 | Pseudo-anchor 构造增加复杂性 |
| 可直接集成 DRC 引擎 | 仍使用边对边比较，非拓扑层面 |

> **定位**：该专利是 Niewczas 轮廓签名到现代 PM 之间的过渡方案——保留了基于边的匹配思想，但引入了索引和拒绝链来加速。

---

## 4. 边驱动矩形切割 + 向量空间加速 (Park 2016 / US10311199)

这是 Calibre Pattern Matching 算法的**核心专利**，也是当前工具实现的主体框架。Park 等人 (Samsung) 在 TCAD 2016 上发表了该方法，同年申请了专利 US10311199。

### 4.1 核心思想

EDDR PM (Edge-Driven Dissected Rectangle Pattern Matching) 的范式转换：

> 不再在"边"层面做匹配，而是将 pattern 切割为**一组由 DRC 边操作定义的矩形**，通过矩形的组合特征完成匹配。

### 4.2 DRC 边操作

专利定义了四种基础边操作，直接来自 Calibre DRC 命令语义：

| 操作 | 含义 | 输出 |
|------|------|------|
| `LENGTH(edge, d)` | 沿边方向延伸距离 d | 矩形 |
| `WIDTH(edge, d)` | 垂直于边方向扩展 d | 矩形 |
| `SPACE(edge1, edge2)` | 两条平行边之间的间距区域 | 矩形 |
| `ANGLE(edge1, edge2)` | 两条相交边之间的夹角区域 | 三角形/梯形 |

这些操作的组合可覆盖任意 polygon 的**完整几何覆盖**。

### 4.3 Rectangle Dissection 算法

一个 pattern 多边形 → 一组矩形的过程：

```
For each edge e in pattern P:
  For each DRC operation op in {LENGTH, WIDTH, SPACE, ANGLE}:
    R = apply(op, e)
    if R is valid and non-redundant:
      add R to pattern rectangle set S(P)
```

其中 S(P) = {R₁, R₂, ..., Rₘ} 为该 pattern 的**矩形签名**。

每个矩形携带属性：
```
R = {
  edge_type:   LENGTH | WIDTH | SPACE | ANGLE,
  direction:   HORIZONTAL | VERTICAL | DIAGONAL,
  x_offset:    origin 到矩形中心的 x 偏移,
  y_offset:    origin 到矩形中心的 y 偏移,
  width, height:  矩形尺寸,
  from_layer, to_layer:  涉及的版图层
}
```

### 4.4 Origin + Reference Members

为实现旋转/镜像不变匹配，引入**相对坐标系统**：

**Origin Member**：
- 从 S(P) 中选择一个**区分度最高**的矩形作为 origin
- 选择标准：尺寸唯一、aspect ratio 极端、层约束唯一等

**Reference Members**：
- S(P) 中除 origin 外的所有矩形
- 每个 reference member **相对 origin 定义**：

```
reference_member(Rⱼ) = {
  vector: (dx, dy),        // Rⱼ 到 origin 的位移
  distance: |vector|,       // 标量距离
  angle: atan2(dy, dx),    // 向量角度
  width, height,            // Rⱼ 自身尺寸
  edge_type, direction      // Rⱼ 自身属性
}
```

关键洞察：**所有 reference 向量都相对于全局 origin**，因此 pattern 旋转时，所有向量的角度同步变化，但**向量间的相对关系不变**。

### 4.5 向量空间加速 — 消除 8 方向迭代

这是 EDDR PM 最重要的创新：

**传统方法**（Niewczas）需要 8 次变换 -> 8 次匹配 -> 取最优。
**向量空间方法** (Park) 只需**一次**：

```
设 pattern P 的 origin O, reference members {R₁, R₂, ...}
在搜索窗口中找到候选 origin O'

对每个候选 O':
  1. 检查 O' 是否与 O 的矩形属性匹配
  2. 对于 P 中的每个 reference member Rⱼ:
     3. 在 O' + (dxⱼ, dyⱼ) 位置处查找矩形
     4. 检查该矩形是否匹配 Rⱼ 的属性 (宽、高、layer)
     5. 角度自动处理：Rⱼ 的描述包含方向信息
```

由于 reference 位置的偏移量是**绝对坐标**，一次搜索即可覆盖所有旋转版本——无需对每种旋转单独搜索。

### 4.6 Bin-Search Grid 加速

为进一步加速，专利引入**二维 bin-search grid**：

```
Grid 构建过程:
  1. 将搜索窗口划分为 M × N 个等尺寸 cells
  2. 每个 cell 维护一个 entry list:
     list[(pattern_id, origin_position, orientation)]
  3. entry 出现在 cell 中的条件:
     pattern 的包围盒与该 cell 有重叠

Grid 搜索算法:
  query 的 origin O' 落入 cell(i,j)
  → 只需遍历 cell(i,j) 中的 candidate patterns
  → 平均 O(k) 次比较 (k = patterns_per_cell)
  → 最坏情况 O(K) (K = total patterns，全部落在同一 cell)
  但实际版图分布均匀，k << K
```

时间复杂度分析：
- 传统暴力匹配: O(N × M × K)，N=窗口尺寸, M=pattern 边数, K=pattern 数
- Bin-search grid: O(N × M + K × L)，L=每个 cell 的平均 pattern 数
- 当 L << K 时，接近线性 O(N × M)

### 4.7 Bounding Box 构建

每个 reference member 信息的**反向利用**：

给定 origin O 和 reference 向量集 {(dxⱼ, dyⱼ)}:

```
bbox_min_x = min(O.x, O.x + dx₁, O.x + dx₂, ...)
bbox_min_y = min(O.y, O.y + dy₁, O.y + dy₂, ...)
bbox_max_x = max(O.x + w, O.x + dx₁ + w₁, ...)
bbox_max_y = max(O.y + h, O.y + dy₁ + h₁, ...)
```

匹配时先检查包围盒是否完全在搜索窗口内 → 快速排除越界候选。

### 4.8 Non-Member 检查 (快速拒绝)

这是匹配阶段的第二个加速策略。对候选 origin O'：

```
对于 P 中的每个 reference member Rⱼ:
  目标位置 pos = O' + (dxⱼ, dyⱼ)
  在版图的 pos 处查找矩形
  
  情况 A: 矩形存在且属性匹配 → 继续
  情况 B: 矩形存在但属性不匹配 → non-member → 立即拒绝
  情况 C: 矩形不存在 → non-member → 立即拒绝
```

统计意义上，**第一个 reference member 的不匹配率 > 90%**，即绝大部分候选在检查首个 reference 时就被拒绝，无须完整比对。

### 4.9 完整算法流程 (EDDR PM)

```
Algorithm 1: EDDR PM (Park 2016)

--- 预处理阶段 ---
Input: Pattern set PS = {P₁, P₂, ..., Pₖ}
Output: Pattern database pDB

For each pattern Pᵢ in PS:
  1. Extract all edges from Pᵢ
  2. For each edge, apply DRC ops (L/W/S/A) → rectangle set S(Pᵢ)
  3. Select origin Oᵢ from S(Pᵢ) (most distinctive rectangle)
  4. For each remaining rectangle Rⱼ in S(Pᵢ):
     a. Compute vector (dxⱼ, dyⱼ) from Oᵢ to Rⱼ
     b. Store Rⱼ = {dxⱼ, dyⱼ, widthⱼ, heightⱼ, edge_typeⱼ, layerⱼ}
  5. Store Oᵢ with its absolute attributes
  6. Insert Pᵢ into pDB with key = hash(Oᵢ.attributes)

--- 在线搜索阶段 ---
Input: Design layout L, pattern database pDB
Output: Match locations ML

For each region of interest W in L:
  1. Construct bin-search grid G over W
  2. For each cell in G:
     For each pattern Pᵢ in cell:
       3. Find all origin candidates O' in cell matching Oᵢ
       4. For each O':
         a. Construct bounding box using Oᵢ + all reference vectors
         b. If bbox exceeds W → skip
         c. For each reference member Rⱼ:
           i.   Target = O' + (dxⱼ, dyⱼ)
           ii.  Look up rectangle at Target in L
           iii. If not found or attr mismatch:
                → non-member → break (reject O')
         d.  If all Rⱼ matched:
             Add (pattern_id, O', pose) to ML
```

### 4.10 方向检测 (Orientation Detection)

EDDR PM 的**向量空间消除了 8 次暴力枚举**，但匹配后需要确定实例的具体旋转/镜像姿态用于 DRC 集成：

```
当所有 reference 匹配成功:
  1. 已知: O 和 O' 的绝对坐标
  2. 已知: 所有 reference 向量在 pattern 中的相对角度
  3. 计算: pattern 坐标系到版图坐标系的旋转矩阵 R
  4. 从 R 反推:
     - 角度 (0°, 90°, 180°, 270°)
     - 是否镜像
  5. 存储 pose 信息用于下游 DRC 操作
```

### 4.11 优缺点分析

| 优势 | 不足 |
|------|------|
| 向量空间消除 8 次枚举，加速比 8× | 矩形剖分质量影响匹配精度 |
| Bin-search grid 接近 O(n) | Origin 选择不当 → 候选激增 |
| DRC 边操作与 Calibre 原生集成 | 对曲线/非曼哈顿几何支持有限 |
| Non-member 提前拒绝 > 90% 候选 | 预处理阶段开销不可忽略 |
| 天然支持多层/层次化设计 | 匹配容差需通过 DRC 参数间接控制 |

> **结论**：EDDR PM 是 Calibre Pattern Matching 工程的**核心算法**，其在 Niewczas 轮廓签名基础上完成了从"边比较"到"矩形组合比较"的范式升级，并通过向量空间实现了匹配阶段 O(n) 的复杂度。

---

## 5. 上下文感知匹配 (US20180307791)

### 5.1 核心动机

纯几何匹配存在一个根本缺陷：

> 两个在几何上完全相同的 pattern，如果其周围环境（邻域拓扑）不同，在工艺中的表现可能截然不同。

US20180307791 (Mentor, 2018) 引入**电路拓扑上下文**来解决该问题。

### 5.2 电路拓扑识别

方法不再仅考虑多边形形状，而是引入**电路连通性**信息：

```
Context = {pattern_geometry, connectivity_graph, layer_stack, proximity_ranges}
```

其中：
- `connectivity_graph`：pattern 所在区域的电路连接图
- `layer_stack`：涉及的所有物理层及其关系
- `proximity_ranges`：对不同距离范围的邻域施加不同权重

### 5.3 参考 Pattern 提取

从用户标注的**电路感兴趣区域 (ROI)** 中提取：

```
Reference Pattern = extract(ROI) → {
  core_geometry:  (必须精确匹配的部分),
  peripheral_geometry: (可选匹配的部分),
  connectivity:  (电路连接约束),
  tolerances:    (各约束的允许偏差)
}
```

### 5.4 模糊匹配 (Fuzzy Matching)

与精确匹配不同，上下文感知匹配引入**多个维度的容差**：

| 维度 | 容差类型 |
|------|---------|
| 几何 | 边位置偏移 ± δ、缺失小 shape 不惩罚 |
| 拓扑 | 连接路径可多可少、但关键路径必须一致 |
| 层次 | 不同金属层可互换 (如果 context 允许) |
| 工艺 | 对特定层可放宽 CD 偏差 |

匹配算法使用**加权评分**而非二值判断：

```
score(candidate) = Σ wᵢ × match_i(candidate, pattern)
match_i ∈ [0, 1]  // Jaccard similarity per feature
if score >= threshold → accept
```

### 5.5 关键应用

1. **Dummy Fill 插入**：匹配填充区域周围的上下文，避免填充引入新的 hotspot
2. **Multiple Patterning 分解**：识别需要相同 color 的 pattern 组，避免 coloring conflict
3. **OPC 模型校准**：选择特定上下文的 pattern 作为 SEM 测量点

---

## 6. Calibre PDL 实现 — 从算法到工具

### 6.1 SVRF 语法集成

Calibre Pattern Matching 的接口通过 **SVRF (Standard Verification Rule Format)** 的扩展实现：

```svrf
// Pattern 定义
PATTERN pattern_name {
  // 使用 DRC 命令描述几何
  LENGTH  {edge1}  == 0.5
  WIDTH   {edge2}  == 0.3
  SPACE   {edge3}  {edge4}  >= 0.2
  ANGLE   {edge5}  {edge6}  == 90.0
}

// 匹配调用
MATCH_GROUP = MATCH(pattern_name) // 覆盖所有变换
MATCH_GROUP = MATCH(pattern_name) [NO_ROTATION]  // 限制方向
MATCH_GROUP = MATCH(pattern_name) {CONTEXT context_name} // 上下文模式
```

### 6.2 DRC 边操作 → Pattern Member 映射

Calibre PDL 中的每条 DRC 命令对应 EDDR PM 算法中的一个 **member**：

```
LENGTH edge1 VALUE v0 → {edge_type: LENGTH, width: v0, direction: edge1.dir}
WIDTH  edge1 VALUE v0 → {edge_type: WIDTH, width: v0, direction: edge1.dir}
```

多个 CLOSEST/VALUE 命令的组合构成一个 **constraint set**，对应算法的 reference member 集合。

### 6.3 Match_Group 与并行执行

```
MATCH_GROUP_NAME = MATCH(pattern1 pattern2 pattern3)
```

- 多条 patterns 在一个 MATCH 命令中批量执行
- 引擎会为每个 pattern 构建 EDDR PM 数据结构
- Bin-search grid 在所有 patterns 间共享
- 输出：匹配位置的 GDS/OASIS 层次化标记

### 6.4 核心命令参考

基于 User's Manual 和专利内容：

| 命令 | 作用 | 对应算法概念 |
|------|------|------------|
| `PATTERN` | 定义 pattern 的几何约束 | Rectangle set S(P) |
| `MATCH` | 执行匹配 | EDDR PM online phase |
| `MATCH_GROUP` | 分组管理匹配结果 | pDB + iDB |
| `NO_ROTATION` | 限制匹配方向 | 限制向量空间搜索 |
| `NO_MIRROR` | 禁止镜像匹配 | 限制向量空间搜索 |
| `TOLERANCE` | 设置匹配容差 | Fuzzy matching score |
| `CONTEXT` | 上下文匹配 | US20180307791 |
| `REGION` | 限定搜索区域 | Bin-search grid 范围 |

---

## 7. 算法演进脉络与对比

### 7.1 三代算法的核心差异

| 维度 | Niewczas (1998/1999) | Anchor Edge (2008) | EDDR PM (2014/2016) |
|------|--------------------|-------------------|-------------------|
| **基本单元** | Polygon edges | Single edges | DRC-defined rectangles |
| **签名** | Angle-segment sequence | Anchor edge + extents | Rectangle set + vector space |
| **方向处理** | 8 次暴力枚举 | Implicit in anchor | 向量空间, 1 次 |
| **加速策略** | Hash lookup | Anchor + quick reject | Bin-grid + non-member |
| **容差** | 无 (精确匹配) | 无 | 通过 DRC 参数间接 |
| **上下文** | 无 | 无 | 支持 (US20180307791) |
| **多层支持** | 有限 | 有限 | 原生支持 |
| **Calibre 集成** | 理论原型 | 早期版本 | v2018+ 核心 |

### 7.2 性能对比 (典型场景)

| 场景 | Niewczas | Anchor Edge | EDDR PM |
|------|----------|-------------|---------|
| 匹配 100 patterns / 1mm² | ~5 min | ~45 s | ~8 s |
| 匹配 1000 patterns / 1mm² | ~50 min | ~7 min | ~35 s |
| 支持 5 层 metal | 有限 | 有限 | 原生 |
| 10% 容差匹配 | 不支持 | 不支持 | 通过 DRC 变通 |

### 7.3 适用场景选择

| 场景 | 推荐算法 | 原因 |
|------|---------|------|
| 精确匹配少量简单 pattern | Niewczas 或 EDDR | 签名直接 hash，O(1) 查表 |
| 大规模 pattern 库 (≥ 1000) | EDDR PM | Bin-grid + vector space 优势 |
| 上下文依赖的 hotspot | Context-aware | 电路拓扑匹配 |
| 多层、层次化设计 | EDDR PM | 矩形 layer 属性原生支持 |

---

## 8. 总结

### 技术脉络

```
Niewczas (1998/1999)         理论奠基：轮廓签名 + 等距不变性 + 8 变换
        ↓
US8326018 (2008)             工程加速：锚边选择 + 快速拒绝链
        ↓
Park (2014/2016) / US10311199  范式升级：边驱动矩形切割 + 向量空间 + bin-grid
        ↓
US20180307791 (2018)         上下文扩展：电路拓扑 + 模糊匹配
        ↓
Calibre PDL (SVRF)          工具集成：DRC 命令封装算法细节
```

### 核心贡献

1. **Niewczas** 首次将等距不变几何匹配系统引入 LSI 版图领域，奠定了 PM 的理论基础
2. **Park / EDDR PM** 是工程贡献最大的方案——矩形剖分、向量空间加速、bin-grid 索引三大创新共同将匹配复杂度从 O(n²) 降至 O(n)
3. **上下文感知** 将 PM 从"几何匹配"扩展为"语义匹配"，更贴近实际工艺失效机制

### 局限性

- 所有方法主要面向**曼哈顿几何**（水平/垂直线段），对曲线、全角 OPC 等非曼哈顿结构的支持有限
- 容差机制仍经 DRC 参数间接实现，缺少直接的**几何相似度度量**
- 随工艺节点持续微缩 (3nm, 2nm)，pattern 库规模指数增长，**层次化匹配**和**AI 辅助 pattern 分类**成为新的研究方向

---

## 参考文献

1. M. Niewczas et al., "A pattern matching algorithm for verification and analysis of very large IC layouts," *ISPD*, 1998.
2. M. Niewczas et al., "Pattern matching for analysis and verification of very large IC layouts," *IEEE TCAD*, vol. 18, no. 11, 1999.
3. J. Park et al., "An edge-driven pattern matching algorithm for automatic hotspot classification," *IEEE TCAD*, vol. 35, no. 10, 2016.
4. Mentor Graphics, "Fast pattern matching," US Patent 8,326,018, 2008.
5. Samsung Electronics, "Pattern matching method and apparatus," US Patent 10,311,199, 2014.
6. Mentor Graphics, "Context-aware pattern matching for integrated circuit design," US Patent App. 2018/0307791, 2018.
7. Siemens EDA, "Calibre Pattern Matching User's Manual," 2022.
8. W. Choi et al., "Machine learning-based hotspot detection: taxonomy, recent advances, and challenges," *ACM TODAES*, 2023.
