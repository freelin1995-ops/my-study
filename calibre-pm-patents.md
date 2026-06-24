# Calibre Pattern Matching 专利群

## Siemens EDA (Mentor Graphics) 直接持有的 PM 专利

| 专利号 | 名称 | 优先权日 | 发明人 | 核心创新 |
|--------|------|---------|--------|---------|
| US8326018B2 | Fast pattern matching | 2010-05-29 | Oberdan Otto, Mark C. Simmons | **锚边加速**：选择 pattern 中最稀有的边作为锚，逐级拒绝（extent → 边计数 → 锚边 → secondary edge → 全匹配）+ 搜索窗口网格分区 |
| US10311199B2 | Pattern matching using edge-driven dissected rectangles | 2016-07-01 | Jea Woo Park, Robert A. Todd | **EDDR PM 核心**：DRC 边操作（L/W/S/A）将多边形切割为矩形集，origin + reference 向量空间消除 8 方向迭代，bin-search grid O(n) 加速 |
| US10496783B2 | Context-aware pattern matching for layout processing | 2017-04-19 | Sherif Hany Riad Mohammed Mousa 等 | **上下文感知匹配**：引入电路拓扑连通性 + 多层 proximity range + 加权评分（Jaccard similarity），支持 dummy fill / multi-patterning / OPC 模型校准 |
| US10796070B2 | Layout pattern similarity determination based on binary turning function signatures | 2018-07-19 | Navin Srivastava, Hanzhong Xu, John E. Hershberger | **BTF 相似度**：将多边形顶点编码为二进制转向函数（左转/右转），循环移位 + 位序列反转 → 最小/最大二进制数作为规范签名，支持**无先验容差的模糊匹配** |
| US8683394B2 | Pattern matching optical proximity correction | 2012-01-31 | 未标注 | **PM + OPC 混合**：用 PM 定位重复 pattern 阵列 → 分成 core/boundary → core 复用预计算 OPC 结果 → boundary 单独跑 OPC |
| US20150067621A1 | Logic-driven layout pattern analysis | 2012-09-05 | 未标注 | **逻辑驱动的 PM**：将电路逻辑连接与版图几何结合，在逻辑等价区域内进行模式匹配 |
| US8677300B2 | Canonical signature generation for layout design data | — | — | **规范签名**：版图数据的规范形式生成，与 Niewczas 轮廓签名相关 |
| US9378327B2 | Canonical forms of layout patterns | — | — | **规范形式**：pattern 的标准化表示 |
| US10089432B2 | Rule-check waiver | — | — | **基于 pattern 的 DRC waiver** |

### 专利间引用关系

```
US8326018 (锚边, 2010)
  ├── US10311199 (EDDR PM, 2016)  ── 引用 US8326018
  ├── US20150067621 (逻辑驱动 PM, 2012)
  └── 被 US10235492 (Globalfoundries XOR 密度, 2017) 引用

US10311199 (EDDR PM, 2016)
  ├── US10496783 (上下文感知, 2018)
  └── 引用 US7818707 (Gennari, 2006)
```

## Frank Gennari 专利家族（Mentor → Cadence）

这些专利的发明人是 Frank Gennari（最初在 Mentor，后来转到 Cadence），专利最后 assignee 是 Cadence：

| 专利号 | 名称 | 优先权日 | 核心创新 |
|--------|------|---------|---------|
| US7818707B1 | Fast pattern matching | 2006 | **图像化 pattern 匹配**：支持 match factor（匹配度因子），可对匹配结果打分 |
| US8516406B1 | Smart pattern capturing and layout fixing | 2010 | **fuzzy pattern replacement**：识别 pattern → 自动替换 → 版图自动修复，替换 pattern 可根据上下文动态调整 |
| US8429582B1 | (同家族) | 2010 | **double patterning 分解上下文**：fuzzy replacement 中考虑多 patterning 着色约束 |

## 学术根源

| 工作 | 年份 | 关联 |
|------|------|------|
| Niewczas, ISPD 1998 / TCAD 1999 | 1998/1999 | 轮廓签名等距不变匹配，Calibre PM 的理论开端 |
| Park, Todd, Song, TCAD 2016 | 2016 | EDDR PM 论文（对应 US10311199） |

## 总结

Calibre PM 专利群的技术演进：

1. **1998/1999** — Niewczas 轮廓签名（理论基础，非专利）
2. **2006-2010** — Gennari 图像化匹配 + fuzzy replacement（后在 Cadence 落地为 Pegasus LPA）
3. **2010** — US8326018 锚边加速（Mentor 首个 PM 工程专利）
4. **2016** — US10311199 EDDR PM 核心（矩形切割 + 向量空间 + bin-grid，当前 Calibre PM 的主体算法）
5. **2018** — US10496783 上下文感知（从几何匹配扩展到语义匹配）
6. **2019** — US10796070 BTF 相似度（二进制转向函数，无先验容差的模糊匹配）

核心专利（EDDR PM）的发明人 Park 来自 Samsung，但申请时 assignee 是 Mentor Graphics——反映了 Samsung 与 Mentor 在 PM 技术上的合作或技术转让。
