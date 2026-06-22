# Calibre Pattern Matching 核心算法原文翻译与解析

本文档将 6 篇核心文献中的算法原文逐段翻译为中文，并附关键技术解析。

---

## 1. 严格匹配 (Exact Matching)

### 1.1 Niewczas 轮廓签名法 (1998 ISPD / 1999 TCAD)

#### 原文 (ISPD 1998, §3 Pattern Matching Algorithm)

> **3.1 The signature**
>
> Let us choose an arbitrary vertex V out of n vertices of a contour K. We construct our signature as a sequence of angles and edge lengths built by following the vertices along the contour.

**翻译：** 从轮廓 K 的 n 个顶点中任选一个顶点 V。我们的签名构造方法为：沿着轮廓遍历顶点，构建一个**角度和边长交替出现的序列**。

---

> **Definition 4.** For a given full vertex sequence A={Vi}i=0,..,n-1, by the angle-segment sequence of contour K we will understand the sequence:
>
> S = {(αi, Li)} i=0,..,n-1
>
> where, αi is a magnitude of internal angle of vertex Vi and Li is the length of the line segment connecting Vi with the vertex V(i+1)mod n.

**翻译：定义 4.** 对给定的完整顶点序列 A = {Vi}，轮廓 K 的**角-边序列**定义为：
S = {(αi, Li)} 
其中 αi 是顶点 Vi 的内角大小，Li 是连接 Vi 和 V(i+1) 的线段长度。

**解析：** 这是 Niewczas 算法的核心数据结构。将一个多边形轮廓编码为一系列 (内角, 边长) 元组。关键洞察：**角度和边长在等距变换（平移/旋转/镜像）下保持不变**，因此这个序列可以作为轮廓的"指纹"。

---

> **Observation.** Let us consider a given contour K and let S be any of this contour's angle-segment sequences in a form given by (1). S uniquely identifies the contour equivalence class to which K belongs.

**翻译：观察结论。** 给定轮廓 K，令 S 是它的任意一个角-边序列。S **唯一确定** K 所属的轮廓等价类。

**解析：** 这是算法的正确性基础。证明思路：给定 S，可以通过三角形全等法重建多边形——前三个顶点 V0V1V2 确定一个三角形（两边+夹角），后续每个顶点通过 angle+length 递推确定，这些操作在等距变换下保序。

---

> **Definition 6.** The contour signature consists of the lexicographically smallest direct angle-segment sequence and lexicographically smallest indirect angle-segment sequence.

**翻译：定义 6.** 轮廓签名由**字典序最小的正向角-边序列**和**字典序最小的逆向角-边序列**组成。

**解析：** 因为从不同起点遍历会得到不同的序列，需要选一个"标准形"作为签名。字典序排序规则：先比角度，角度相同再比边长。为什么需要两个序列？因为镜像翻转后正向序列会变成逆向序列（图 2 直观展示），所以需要同时存储两个方向的序列才能识别镜像等价。

---

> **3.2 Signature comparison**
>
> Let us assume that two contours P (pattern) and C (candidate) are given. P and C are equivalent if and only if one of the following is true:
>   • minimal direct angle-segment sequence of P is equal to the minimal direct angle-segment sequence of C
>   or
>   • minimal indirect angle-segment sequence of P is equal to the minimal direct angle-segment sequence of C.

**翻译：3.2 签名比较。** 给定两个轮廓 P (pattern) 和 C (candidate)，P 和 C 等价当且仅当以下之一成立：
- P 的最小正向序列 == C 的最小正向序列
- **或者** P 的最小逆向序列 == C 的最小正向序列

**解析：** 匹配规则的精髓。只需对候选 C 计算一次最小正向序列，然后分别与 P 的正向和逆向序列比较即可。当匹配到逆向序列时，说明 C 是 P 的镜像。

---

> **3.3 Chain matching**
>
> The chain matching procedure differs from contour matching only at the stage of signature construction. An open chain is represented by exactly two vertex sequences: a direct one and an indirect one. Thus, the smallest direct angle-segment sequence starts at the beginning of the chain and the smallest indirect one starts at the and of the chain.

**翻译：3.3 链匹配。** 链匹配与轮廓匹配的**唯一区别**在于签名构造阶段。一条开放链正好用两个顶点序列表示：正向和逆向。最小正向序列从链起点开始，最小逆向序列从链终点开始。

**解析：** Chain（子链）是轮廓的部分边序列，用于描述 OPC 边界特征或走线路径。Niewczas 发现很多设计中的小尺度特征（如 2μm 间隔内的顶点链）重复度极高，用链匹配可以进一步压缩数据。

---

> **4.3 Constraint of transformations of interest**
>
> In our current implementation, we decided to limit the possible contour instance transformations to arbitrary translations, the reflection with respect to the Y axis (traditionally called mirror X or MX), and rotations by an angle which is a multiple of π/2. This is because all coordinates are snapped to a grid. Hence, a rotating a polygon may in general have slightly different edge lengths and angle magnitudes than before rotation.

**翻译：4.3 变换限制。** 当前实现中，我们限制了可能的轮廓变换为：任意平移、Y 轴镜像（即 MX）、以及 π/2 整数倍的旋转。因为所有坐标都**吸附到网格上**，旋转一个多边形通常会导致边长度和角度大小发生微小的变化。

**解析：** 这是 Niewczas 对实际工程的妥协——理论上签名支持任意角度，但网格化坐标使得非 90° 旋转会引入舍入误差。限制到 8 种变换（4 个旋转 x 2 个镜像）后，变换矩阵只包含 0 和 ±1，3 bit 即可编码。

---

> **4.4 The case of π/4 geometry**
>
> As shown in Figure 5, we can use a very limited number of angle codes to represent angle magnitudes for the π/4 geometry. Now, the contour signature can be stored in a more efficient way since the angle codes require only 3 bits.

**翻译：4.4 π/4 几何的特殊处理。** 如图 5 所示，π/4 几何的角只需 7 种编码（45° 倍数），每个角度只需 **3 bit**。曼哈顿几何（90° 倍数）更少，只需 1 bit。

**解析：** 这是针对 IC 版图特点的优化——大多数版图是曼哈顿或 45° 线，不需要存储完整浮点角度。

---

#### TCAD 1999 版补充

> **Definition 5 (Ordering relation).** S1 is smaller than S2, if and only if one of the three following conditions is true:
> 1) n < m;
> 2) n = m and there exists i for which αi < βi and αk = βk for all k < i;
> 3) n = m and αi = βi for all i and there exists j for which aj < bj and am = bm for all m < j.

**翻译：定义 5（序关系）。** S1 < S2 当且仅当以下之一成立：
1) 顶点数 n < m
2) 顶点数相等，存在第一个 i 使 αi < βi
3) 顶点数和角度序列全等，存在第一个 j 使边长 aj < bj

**解析：** 字典序的具体定义。顶点数优先，然后是角度序列，最后是边长序列。注意角度比边长优先级高——Niewczas 认为角度是更本质的几何特征。

---

### 1.2 US8326018 锚边排序加速 (2008 Mentor Graphics)

#### 原文 (§ Brief Summary / Fig. 3 Flowchart)

> Initially, an edge in the reference pattern is selected as an anchor edge. The anchor edge may be selected, for example, as the edge in the library pattern having the fewest number of matching edges in the region of the layout design being searched (i.e., the search window area).

**翻译：** 首先，在参考 pattern 中选择一条**锚边 (anchor edge)**。锚边的选择依据是：在搜索窗口区域中，**匹配数最少的边**被选为锚边。

**解析：** 核心思路——选择选择性最高的边作为锚点。如果选择锚边后搜索窗口中找到 0 个匹配，则整个搜索窗口直接跳过（无匹配），这是最快速的拒绝。

---

> After the anchor edge is selected, an edge in the search window area matching the anchor edge (i.e., an anchor matching edge) is selected. A search portion of the reference pattern then is compared with the region of the search window area corresponding to the selected anchor matching edge.

**翻译：** 选定锚边后，在搜索窗口中找与其匹配的锚匹配边。然后将参考 pattern 的搜索部分与锚匹配边对应的搜索窗口区域进行比较。

---

> With various implementations of the invention, the anchor selection module first sorts edges (or layout edges) in a search window area by endpoint type (concave or convex) and then by length (e.g. longest to shortest). For each anchor edge candidate (any real edge in the reference pattern), the anchor selection module may determine which and how many layout edges in the tile match it.

**翻译：** 锚边选择过程：先将搜索窗口中的边按**端点类型（凹/凸）**排序，再按**长度**排序。对每条锚边候选，统计搜索窗口中有多少条与之匹配的边。

**解析：** 排序的目的是利用二分查找快速定位匹配边——同类型+同长度的边是连续存储的。匹配包含 8 种方向变体（4 旋转 x 2 镜像），都在同一组中。

---

> If there are variable edges in the library pattern, all the fixed vertices may be matched first. If successful, each variable vertex in the library pattern can move around in a rectangle determined by its allowable range.

**翻译：** 如果 library pattern 中包含了**可变边 (variable edges)**，则首先匹配所有固定顶点。如果成功，每个可变顶点可以在其允许范围内移动。

**解析：** 这是该专利中**模糊匹配的雏形**——允许边在一定范围内偏移，是后来 EDDR PM 模糊匹配的前身。不过这里的"可变"仍然需要用户显式声明范围，不是自动的几何相似度计算。

---

### 1.3 EDDR PM 边驱动矩形切割 + 向量空间 (Park 2016 TCAD / US10311199)

#### Algorithm 1: 暴力扫描线版

```
Algorithm 1 EDDR PM (Pattern Match Using Edge Driven Dissected Rectangle)
 1: procedure EDDR-PM(P1, P2, ...., Pn, nonMem, PDB)
 2:    P1 = set of origin members in a layout
 3:    P2 = set of the first reference members in a layout
 4:    P3..n = set of all the other reference members in a layout
 5:    nonMem = set of non-member polygons in a layout
 6:    PDB = a pattern description database.
 7:    while !empty in P1 do
 8:        Find a reference member p2 in P2 by searching the vector distance
 9:        between P1's center and P2's center.(PDB has this info.)
10:         if found then
11:             if the found p2 a valid reference member then
12:                  Create a bounding box using d1, d2, d3, and d4 determined by
13:                  vector info between P1 and the found P2's center.
14:             else
15:                  No match. continue to next p1 in P1
16:             if nonMem exists inside the bounding box then
17:                  No match. continue to next p1 in P1
18:             Find other members in P3...Pn inside the bounding box.
19:             for each member inside the bounding box do
20:                  n = number of each member inside bounding box
21:                  m = number of each member described in PDB
22:                  if n != m then
23:                      No match. continue to next p1 in P1
24:                  if valid member == false then
25:                      No match. continue to next p1 in P1
26:             Matching pattern found at this point.
27:             Output the bounding box to indicate the match.
28:             continue to next p1 in P1
29:         else
30:             No match. continue to next p1 in P1
```

**翻译与解析：**

```
算法 1 EDDR PM (边驱动矩形切割模式匹配)
第 1-6 行：输入参数
  P1 = 所有"原点成员"（pattern 的基准矩形）
  P2 = 所有"第一参考成员"（相对于 P1 定位的第一个参考矩形）
  P3..n = 其他参考成员
  nonMem = 非成员多边形（bounding box 中不应出现的形状）
  PDB = pattern 描述数据库

第 7-9 行：对每个 P1 候选，用向量（角度+距离）找对应的 P2
  → 关键优化：通过向量空间消除了 8 方向暴力枚举
  → 原文：Vector pointing from the center point of the origin rectangle
    member P1 to the center point of the first reference rectangle
    member P2 may be used to address the issue

第 12-13 行：用向量信息创建 bounding box
  d1,d2,d3,d4 是 P1 中心到 bounding box 四个边的距离
  注意：仅靠距离不能唯一确定 bounding box（8 种方向）
  但加上 P1→P2 的向量后就能唯一确定

第 16-17 行：快速拒绝——如果 bounding box 内有非成员多边形，直接跳过
  原文：If there is a non-member polygon inside the bounding box,
  it is immediately classified as no match (Fig. 7)

第 19-25 行：对 bounding box 内的每个成员，检查数量和有效性
  数量不匹配 → 立即拒绝（Fig. 8）
  valid member 检查 → 验证矩形的 Len1/Len2/Width 是否符合 PDB

第 26-27 行：所有检查通过 → 匹配成功！
```

**解析：** Algorithm 1 的核心创新是**向量空间加速**。传统方法需要 8 次独立迭代来尝试每种方向（0°/90°/180°/270° × 镜像/非镜像），而 EDDR PM 通过 P1→P2 的向量（角度+距离）可以一步定位到正确方向。但算法 1 使用扫描线搜索，复杂度 O(n²)，不适用于大规模 layout。

---

#### Algorithm 2: Bin-Search Grid 优化版

```
Algorithm 2 EDDR PM (with Bin-Search Grid)
 1: procedure EDDR-PM(P1, P2, ...., Pn, nonMem, PDB)
 2:    Inputs (P1..n, nonMem, and PDB) are the same as Algorithm 1.
 3:    ADD_BIN for each member from P1 to Pn.
 4:    LOCATE_BIN for all the origin members of P1 and get bin_counts
 5:    for i = 1 → bin_counts for pi in P1 bins do
 6:        LOCATE_BIN a reference member, p2, in P2 bins by searching
 7:        the vector distance between P1's center and P2's center.
 8:        if found then
 9:            Same process as Algorithm 1 to decide match or no match
10:             .....
11:             LOCATE_BIN for other members inside the bounding box.
12:             .....
13:             Same process as Algorithm 1 to decide match or no match
14:             .....
15:         else
16:             No match. continue
```

**翻译与解析：**

```
第 3 行：将所有成员矩形加入 bin grid（二维网格索引）
第 4 行：在网格中定位所有 P1（原点成员）
第 6-7 行：用向量信息在 P2 的网格中快速定位参考成员
第 11 行：在网格中定位其他成员
```

**解析：** Algorithm 2 用 **bin-search grid** 代替扫描线，将复杂度从 O(n²) 降至 O(nk)，其中 k 是搜索框重叠的网格元素数。因为 pattern bounding box 相对于全芯片很小，通常 k=1 或 k=9（3x3 网格），所以实际复杂度为 O(n)。

原文复杂度分析：
> Let n denote total number of P1 and let m denote total number of all members in a layout. Since Algorithm 1 visits all origin members in P1, it is O(n) for the while loop. The scan-line search to find other members at the step (8) and step (18) inside the while loop requires m times topological check per each loop. Therefore, the complexity becomes O(nm). Because of m >= n, we can say that O(n(n+c)) where c is a constant, and it becomes O(n^2).

> Those two steps in Algorithm 1 have been replaced with LOCATE_BIN for Algorithm 2. Since LOCATE_BIN's time complexity is O(k) where k is the total number of grid elements overlapped by a search box, Algorithm 2 has O(nk). Because k << n or k = 1 in general in our pattern match process, it is O(n) in practice.

---

## 2. 模糊匹配 (Fuzzy Matching)

### 2.1 Park EDDR PM 模糊匹配 (2016 TCAD §III-E)

#### 原文关键段落

> **E. Fuzzy Pattern Match**
>
> With our simple approach to pattern match, we could see another benefit of EDDR PM when it comes to fuzzy pattern matching. Fig. 10 illustrates this. Because we can use not only == but also other relational operators (>, >=, <, <=) for pattern description, we can describe a pattern in a fuzzy way and do a fuzzy pattern match.

**翻译：E. 模糊匹配。** 我们的方法在模糊匹配方面有额外优势。因为 pattern 描述不仅可以用 ==，还可以用关系运算符 (>, >=, <, <=)，所以可以用模糊方式描述 pattern 并进行模糊匹配。

**解析：** EDDR PM 的模糊匹配本质上是**参数区间匹配**——将 LENGTH/WIDTH/SPACE 约束从 == 扩展到 >=、<= 等。这不是几何形状的模糊相似度，而是 DRC 参数范围的模糊化。

---

> In this case, the vector space information is no longer valid. We can use the number of each member inside bounding box for fuzzy pattern match. Therefore, we skip validation checks at the step (24) and (11) of Algorithm 1 for members that are derived from relational operators except == operator. It also must either have at least one member rectangle created by only == operator inside the bounding box or have a configuration where origin member rectangle's center point is unchanging.

**翻译：** 这种情况下，向量空间信息不再有效。我们可以用 bounding box 内各成员的数量来做模糊匹配。因此，对于使用关系运算符（非 ==）的成员，跳过 Algorithm 1 中第 24 和 11 行的校验检查。同时，bounding box 内必须至少有一个用 == 创建的成员矩形，或者原点成员矩形的中心点保持不变。

**解析：** 为什么向量信息不再有效？因为使用了 >= 等关系运算后，矩形的实际位置可以浮动，P1→P2 的向量不再是确定值。解决方法：
1. 至少有一个成员用 == 定义（提供定位锚点）
2. 或者原点成员中心不变（提供基准点）
3. 对模糊成员跳过 valid member 检查

---

#### Algorithm 3: EDDR PM Fuzzy Match

```
Algorithm 3 EDDR PM Fuzzy Match
 1: procedure EDDR-PM(P1, P2, ...., Pn, nonMem, PDB)
 2:    Inputs (P1..n, nonMem, and PDB) are the same as Algorithm 1.
 3:    ADD_BIN for each member from P1 to Pn.
 4:    LOCATE_BIN for all the origin members of P1 and get bin_counts
 5:    for i = 1 → bin_counts for pi in P1 bins do
 6:        LOCATE_BIN a reference member, p2, in P2 bins by searching
 7:        the vector distance between P1's center and P2's center.
 8:        if found then
 9:            if the found p2(reference member) is fuzzy member then
10:                 Create one of 8 bounding boxes using 8 different d1, d2, d3,
11:                 and d4 sets stored in PDB.
12:                 Iterate 8 times from step(19) to step(35).
13:             else
14:                 if the found p2 a valid reference member then
15:                      Create a bounding box using d1, d2, d3, and d4 determined by
16:                      vector info between P1 and the found P2's center.
17:                 else
18:                      No match. continue to next p1 in P1
19:             if nonMem exists inside the bounding box then
20:                 No match. continue to next p1 in P1
21:             LOCATE_BIN for other members in P3...Pn
22:             inside the bounding box.
23:             for each member inside the bounding box do
24:                 n = number of each member inside bounding box
25:                 m = number of each member described in PDB
26:                 if n != m then
27:                      No match. continue to next p1 in P1
28:                 if member is fuzzy member then
29:                      skip member validation check
30:                 else
31:                      if valid member == false then
32:                          No match. continue to next p1 in P1
33:             Matching pattern found at this point.
34:             Output the bounding box to indicate the match.
35:             continue to next p1 in P1
36:         else
37:             No match. continue to next p1 in P1
```

**翻译与解析：**

```
第 9-12 行：如果 P2 是模糊成员
  → 无法用单一向量确定 bounding box
  → 创建 8 组不同的 d1,d2,d3,d4（对应 8 种方向）
  → 迭代 8 次尝试所有方向

第 28-29 行：对模糊成员跳过 valid member 检查
  → valid member 检查验证矩形的精确 Len1/Len2/Width
  → 模糊成员的尺寸在区间内变化，所以跳过此检查

关键差异（与 Algorithm 1 的比较）：
  算法 1: 用 P1→P2 向量一步确定方向和 bounding box
  算法 3: 如果 P2 是模糊成员，向量不再确定 → 退回 8 次迭代
```

**核心权衡：** 模糊匹配失去了向量空间带来的加速优势，在最坏情况下退回 8 次迭代。但如果原点成员 P1 是用 == 定义的（精确尺寸+位置），则 P1 本身提供唯一锚点，仍然可以避免部分迭代。

---

> Fig. 11 shows fuzzy match examples in details. (b) is for exact match where we have four member rectangles, P1, P2, P3, and P4. (e) uses relational operators to perform fuzzy match.

**翻译：** 图 11 详细展示了模糊匹配示例。图(b)是精确匹配（4 个成员都用 ==），图(e)是模糊匹配（P3 的 Len1 >= 0.235, P4 的 Len1 >= 1.251）。

**解析：** 具体示例：
- 精确版 (b): P1{Len1==1.142, Len2==0.240, Width==1.325}, P2{Len1==1.142, Len2==0.667, Width==0.265}...
- 模糊版 (e): P1{Len1>=1.142, Len2==0.240, Width==1.325}, P3{Len1>=0.235, Len2==0.235, Width==1.011}...
  → P1 的 Len1 从 == 改为 >=，P3 完全改为 >=
  → 意味着 P1 的宽度可以大于 1.142，P3 的尺寸可以更大

---

> If there is only one member with its unchanging center point, we have to iterate 8 times to cover all the 8 orientations, which is the case of Fig. 12.

**翻译：** 如果只有一个成员的中心点不变，就必须迭代 8 次来覆盖所有方向（图 12 的情况）。

**解析：** 图 12 展示了一个更复杂的模糊匹配场景，其中引入了 **Space 成员**（间距矩形）——它不是物理多边形，而是两个多边形之间的间距。通过 Space >= 0.238 的描述，可以匹配不同间距的 pattern。

---

### 2.2 US8429582B1 "Don't Care Areas" 模糊替换 (Cadence 2010)

> Some embodiments are directed at articles of manufacture embodying a sequence of instructions for implementing these processes... the methods or system comprise the processes or modules for implementing fuzzy pattern replacement in a layout of an electronic circuit design... the methods or systems comprise the processes or modules of identifying a first pattern from within the layout, identifying one or more second pattern based at least on one or more don't care areas of the first pattern, and performing the fuzzy replacement by replacing the first pattern with at least one of the plurality of second patterns in the layout.

**翻译：** 一些实施方式涉及实现模糊 pattern 替换的方法和系统：从版图中识别第一个 pattern → 基于第一个 pattern 的一个或多个 **don't care 区域**识别第二个 pattern → 执行模糊替换。

**解析：** 这是 Cadence 模糊匹配专利的核心概念。不同于 EDDR PM 的参数区间匹配，这里的"模糊"是通过**允许局部忽略**（don't care areas）实现的。即在 pattern 中标记某些区域为"不在乎"——这些区域内的几何可以任意变化而不影响匹配。这与 Partial Match 有本质不同：Partial Match 是完全忽略未指定的多边形，而 Don't Care 是有选择地忽略指定区域。

---

### 2.3 US20180307791 上下文感知匹配 (Mentor 2018)

> **Aspects of the disclosed technology relate to techniques of context-aware pattern matching and processing. In one aspect, there is a method comprising: receiving a circuit design... analyzing the circuit design to identity circuit components of interest; extracting reference layout patterns that are associated with the circuit components of interest; performing pattern matching to identify layout patterns that match the reference layout patterns; processing the layout patterns; and reporting results.**

**翻译：** 上下文感知 pattern 匹配方法：接收电路设计 → 分析电路设计识别感兴趣的电路组件 → 提取与这些组件关联的参考 layout pattern → 执行 pattern 匹配 → 处理匹配结果 → 报告。

**解析：** 这是 PM 从纯几何匹配到语义匹配的跃迁。传统 PM 只关心"形状是否一样"，而上下文感知 PM 先通过电路网表分析找到关键组件（如敏感 net、螺旋电感），然后**自动提取**这些组件周围的环境作为 pattern。这使得模糊匹配不再是参数范围，而是"与这个已知 hotspot 具有相同电路上下文的区域"。

---

> The extracting may comprise: determining preliminary reference layout patterns that are associated with the circuit components of interest; and compressing the preliminary reference layout patterns to generate reference layout patterns. The pattern matching may comprise fuzzy pattern matching.

**翻译：** 提取过程：确定与目标组件关联的初步参考 layout pattern → **压缩**这些初步 pattern 生成最终参考 pattern。匹配可以包含模糊匹配。

**解析：** "压缩"是关键——从同一类组件的多个实例中提取公共几何特征，形成泛化的 pattern 描述。例如，从 10 个不同尺寸的电感中提取出"螺旋电感的公共几何特征"，然后模糊匹配所有类似的螺旋结构。

---

> The processing may comprises adding dummy fills to the layout patterns. The dummy fills may be added based on orientations of the layout patterns. Alternatively or additionally, the processing may comprise performing layout decomposition for multiple-patterning.

**翻译：** 处理步骤可以包括：添加 dummy fill（基于 pattern 方向）、执行 multi-patterning 版图分解。

**解析：** 上下文感知 PM 不限于检测 hotspot，还扩展到 DFM 修复。例如识别螺旋电感后，根据其方向添加对称的 dummy fill 以保持 Q 因子不变。

---

## 3. 各算法复杂度对比

| 算法 | 时间复杂度 | 空间复杂度 | 核心加速手段 |
|------|-----------|-----------|-------------|
| Niewczas 轮廓签名 | O(n) 签名构造 + O(1) 字符串比较 | O(n) 每 pattern | 字典序最小签名简化比较 |
| US8326018 锚边 | O(e) 锚边选择 + O(m) 逐边比较 | O(1) | 选择最少匹配的边做剪枝 |
| EDDR PM Algo1 扫描线 | O(n²) | O(n) | 向量空间消除 8 方向迭代 |
| EDDR PM Algo2 bin-grid | O(n) 实际 | O(n) | bin-grid 索引 |
| EDDR PM Fuzzy Algo3 | O(8n) 最坏 | O(n) | 退回到 8 次迭代 |
| 上下文感知 US20180307791 | N/A (依赖底层 PM) | N/A | 电路分析预筛选 |

## 4. 关键结论

1. **严格匹配的进化路线：** Niewczas 的符号匹配（角-边字符串）→ 锚边选择剪枝 → EDDR 矩形切割 + 向量空间 → 每次进化都让匹配更快、更精确

2. **模糊匹配的三种实现方式：**
   - **DRC 参数区间**（EDDR PM）：用 >=、<= 代替 ==，最简单但仅支持一维容差
   - **Don't Care Areas**（Cadence US8429582B1）：标记不关注区域，允许局部几何变化
   - **上下文感知**（Mentor US20180307791）：从电路语义层面做模糊匹配，最强大但也最复杂

3. **EDDR PM 的向量空间是最大创新：** 将 8 次迭代降为 1 次，从 O(8n) 到 O(n)。代价是一旦进入模糊匹配（关系运算符），向量信息失效，退回 8 次迭代。
