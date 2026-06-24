# Calibre 产品系列概览

Calibre 是 Siemens EDA（原 Mentor Graphics）的芯片设计后端全流程平台，覆盖从物理验证、寄生提取、可制造性设计、分辨率增强技术到掩模数据准备的所有环节。本文档基于 Calibre 2025.1 版用户手册整理，标\* 的产品有独立用户手册。

---

## 一、物理验证（Physical Verification）

物理验证是 Calibre 的核心起点，确保版图在几何和电气上符合代工厂要求。

### Calibre nmDRC \*
**解决问题：** 版图是否符合代工厂的设计规则（最小间距、最小宽度、最小面积等）。

物理验证中最核心的一环。nmDRC 读取代工厂提供的规则文件（SVRF/TVF 格式），对版图执行数百到数千条几何规则检查，输出违规标记到 RVE 中供设计者调试。层次化引擎是其关键优势——只处理有变化的部分，大幅缩短全芯片运行时间。

使用场景：所有芯片的签核（Sign-off）前必须通过 DRC clean。

### Calibre LVS (Layout vs. Schematic) \*
**解决问题：** 画出来的版图是否与电路图一致。

将版图中提取出的器件和连接网表与源电路网表进行比对，报告短路、断路、器件尺寸/类型不匹配等问题。LVS 也是连接 DRC 和寄生提取的桥梁——只有 LVS clean 的设计才能进入后续流程。

使用场景：模拟/数字全定制设计的签核步骤。

### Calibre PERC \*
**解决问题：** 芯片是否存在 ESD 损伤、电压越限、浮栅等可靠性隐患。

PERC 超越传统 ERC 的功能边界，基于拓扑连接关系和电压传播进行分析。典型检查包括：IO PAD 是否有 ESD 保护器件、核心电路是否暴露在高压下、栅极是否浮空、电源域交叉是否安全。支持布局后提取实际寄生电阻进行 IR Drop 感知的 PERC 分析。

使用场景：汽车电子、航空、工业级芯片等对可靠性要求极高的设计。

### Calibre 3DStack \*
**解决问题：** 3D-IC 堆叠设计（多个 Die + Interposer）如何在跨芯片层面做 DRC/LVS。

支持多芯片堆叠、TSV、Micro-bump、Interposer 互联的物理验证。核心功能包括：跨芯片 DRC（如 TSV 间距、对准标记检查）、跨芯片 LVS（Die-to-Die 连接一致性）、天线效应检查、悬空网络检测。支持 3Dblox 格式作为芯片级组装描述。

使用场景：HBM、Chiplet、SoIC 等 3D-IC 封装设计。

### Calibre DESIGNrev \*
**解决问题：** 如何快速查看、测量和修改版图数据。

全功能的版图查看与编辑器。支持 GDSII/OASIS/LEF/DEF 格式，提供 Cell 浏览器、层浏览器、量尺、Tcl 脚本宏扩展等功能。可作为验证结果的图形化前端，直接高亮显示 DRC/LVS 错误位置。

使用场景：版图 debug、物理分析、小范围工程修改（ECO）。

### Calibre RVE (Results Viewing Environment) \*
**解决问题：** DRC/LVS/PERC 结果太多，如何快速定位和理解。

统一的物理验证结果浏览器。以树形结构组织违规列表，支持按层/按规则/按 Cell 分组过滤。可高亮回标到版图编辑器，查看详细描述和修复建议。DRC、LVS、PERC、LFD 的结果都共用一套 RVE 界面。

使用场景：日常验证 debug。

### Calibre Interactive \*
**解决问题：** 如何从版图编辑器方便地启动验证流程。

Calibre Interactive 是嵌入到 Virtuoso、Laker 等版图编辑器中的 GUI 界面，设计者无需离开编辑环境即可配置和运行 DRC/LVS/PEX/PERC 等验证任务，并直接在版图上查看回标结果。

使用场景：设计-验证-修改的快速迭代。

### Calibre RealTime（Digital / Custom / APRISA）\*
**解决问题：** 画版图时能否即时反馈 DRC 违规，避免后期大量返工。

将 DRC 引擎嵌入到 EDA 布局布线工具中，在设计过程中实时增量检查。支持 Cadence Innovus/Virtuoso、Synopsys ICC2、SpringSoft Laker、Siemens APRISA 等主流工具。只在修改区域做增量 DRC，性能足以支持交互响应。

使用场景：布局布线阶段的在版 DRC。

### Calibre Auto-Waivers \*
**解决问题：** 大批量已知可接受的 DRC 违规能否自动豁免，避免重复人工确认。

基于规则或位置匹配，自动豁免满足条件的 DRC 违规并生成报告。支持 waive 生命周期管理，追踪每个 waive 的工程师、时间和原因。

使用场景：成熟工艺 / IP 复用中的 DRC 违规管理。

### Calibre eqDRC \*
**解决问题：** 传统 DRC 只看几何，但有些电气上等价的布局差异不应报错。

eqDRC 引入 DFM 属性（长度、宽度、面积、周长、间距等）作为检查要素，支持非几何上下文的规则检查。例如检查不同宽度走线的电流承载能力是否超出工艺限制。

使用场景：先进工艺中结合电气行为的物理规则检查。

### Calibre YieldServer \*
**解决问题：** 如何从大批量 DRC/LFD/CMP 结果中分析出系统性良率风险。

统一的良率分析平台，支持 Tcl 脚本化的大规模数据后处理、热点聚类、报告生成。配合 CMPAnalyzer 和 LFD 使用。

---

## 二、寄生参数提取（Parasitic Extraction）

从版图几何中提取电阻、电容等物理参数，用于后仿真是连接物理设计和电路仿真的桥梁。

### Calibre xACT \*
**解决问题：** 版图中的互联线会引入寄生 R、C，如何精确提取出来用于后仿真。

3D 场求解器精度级别的寄生提取工具。支持晶体管级（Flat）、门级（Gate-Level）、层次化、混合信号等多种提取模式，输出 DSPF/SPEF 等标准寄生网表。还支持：
- 3D 精确提取（xACT 3D）
- 电感提取
- TSV 耦合提取
- 多工艺角/多温度同时提取
- 对提取结果的可视化调试（xACTView）

使用场景：所有需要签核后仿真的设计。

### Calibre xRC \*
**解决问题：** 针对数字流程的快速寄生提取。

xACT 之前的主力产品，主要面向全数字流程，支持 LEF/DEF 数字流程，用于 STA（静态时序分析）需要的寄生参数文件生成。

### Calibre xCalibrat \*
**解决问题：** 提取模型中的介电常数、厚度等工艺参数如何校准。

通过与测试结构实测数据对比，自动反推提取模型参数，确保仿真结果与硅片实测一致。支持批量校准和交叉验证。

使用场景：工艺开发阶段的 PDK 建模。

---

## 三、可制造性设计（DFM）

在版图设计阶段预测和修复制造工艺偏差，提升良率。

### Calibre LFD (Litho-Friendly Design) \*
**解决问题：** 光刻工艺窗口有限，版图在工艺偏差下打印出来什么样。

全芯片光刻工艺仿真工具。核心能力：
- **PV-Band（工艺窗口带）**：模拟焦距/剂量变化条件下的版图成像偏差范围
- **DVI（设计可变性指数）**：量化每个图形的工艺敏感性
- **内置检查**：线端缩短、最小间距/宽度在工艺变化下的失效风险
- **DNN Flow**：基于深度神经网络加速仿真

输出结果以热力图和 RDB 形式展示，设计者可据此调整版图。

使用场景：先进节点（7nm 以下）的物理验证签核。

### Calibre CMPAnalyzer \*
**解决问题：** 化学机械抛光后金属厚度不均匀会影响性能和良率。

仿真 CMP 研磨过程，预测芯片表面各处的铜厚度分布，识别厚度/凹陷/侵蚀超标的 hotspot。支持预填充和后填充分析，以及热点自动聚类。可导出厚度图给寄生提取工具作修正。

使用场景：后端互联层的平坦化分析。

### Calibre Pattern Matching \*
**解决问题：** 已知有问题的版图形状，如何快速在新版图中找到类似问题。

不需要运行完整 DRC，而是用已知问题的拓扑图样（Pattern）库在新版图中做匹配搜索。支持三种匹配模式：
- **TEM（Topological Edge Match）**：边沿拓扑匹配
- **BCM（Barcode Match）**：条形码模式匹配
- **XBAR（Crossbar）**：交叉条匹配

匹配结果可以按相似度排序，也支持聚类以减少重复结果。

使用场景：工艺迁移、IP 复用时的已知热点检查。

### Calibre Multi-Patterning \*
**解决问题：** 先进工艺一层掩模画不下，如何自动拆成多层。

自动将一层版图分解为多张掩模版，支持 LELE（Light-Etch-Light-Etch）、LFLE（Litho-Freeze-Litho-Etch）、SADP（Self-Aligned Double Patterning）、SAQP（Self-Aligned Quadruple Patterning）。提供冲突检测、自动/手动 Stitch、Seed Anchor 等机制，输出拆分后的每层掩模数据。

使用场景：7nm/5nm/3nm 等需要多重图形的节点。

### Calibre LPE (Local Printability Enhancement) \*
**解决问题：** 流片后发现的少数光刻热点，能否在不重新做 OPC 的前提下修复。

LPE 对版图中的局部光刻热点做针对性修复。基于仿真结果识别需要修正的区域，自动应用局部 OPC 校正或图形优化，避免全局重新 OPC 的成本。修复后必须重新验证（通过 LFD 或 OPCverify）。

使用场景：流片前最后一轮热点修复。

---

## 四、分辨率增强技术（RET）

在光刻极限下，通过修改掩模图形来提高成像质量。

### Calibre OPCpro \*
**解决问题：** 光刻衍射导致版图变形失真，如何提前在掩模上反向补偿。

模型驱动的 OPC 工具。核心能力：
- **Fragmentation**：将图形边缘切分为小段，每段独立计算校正量
- **Model-based Correction**：基于光学/光刻胶模型计算每段的移动方向和距离
- **EPE (Edge Placement Error) 控制**：确保校正后边缘位置误差在容忍范围内
- **Tag 系统**：不同图形片段可赋予不同校正策略

使用场景：所有先进节点的掩模制作前置步骤。

### Calibre nmOPC \*
**解决问题：** 全芯片 OPC 如何跑得快。

OPCpro 的层次化引擎，利用设计层次结构加速 OPC 计算，只对层次之间有差异的部分做校正。

### Calibre OPCverify \*
**解决问题：** OPC 校正后的掩模版图在工艺窗口下能否正确成像。

用经过校准的光学模型对 OPC 校正后的结果做全芯片仿真验证，检查校正是否引入新的 Pinching/Bridging 等问题。

使用场景：OPC 后签核检查。

### Calibre nmSRAF \*
**解决问题：** 孤立图形成像质量差，如何在掩模上添加辅助图形。

自动生成亚分辨率辅助图形（Sub-Resolution Assist Features），帮助提高孤立图形的成像质量而又不会被真正打印到晶圆上。支持规则的、基于仿真的和模型优化的 SRAF。

### Calibre SMO (Source-Mask Optimization) \*
**解决问题：** 照明光源形状和掩模图形能否一起优化来获得最佳工艺窗口。

SMO 同时优化照明系统的光瞳形状和掩模图形，比单独 OPC 获得更宽的工艺窗口。具体模块包括：
- **pxSMO**：像素级光源-掩模联合优化
- **RET Selection**：SMO 流程管理与结果对比
- **Parametric Explorer**：参数化优化探索

使用场景：最先进节点（5nm 以下）的关键层光刻优化。

### Calibre pxOPC \*
**解决问题：** 传统基于片段的 OPC 是否还有进一步提升空间。

基于像素级的 OPC 校正方法，在传统片段 OPC 基础上进一步精细化，适用于对精度要求极高的层。

### Calibre WORKbench \*
**解决问题：** RET 流程调试和优化需要一套图形化环境。

Calibre 的 RET 开发平台。提供光学模型调试、OPC 参数调优、仿真结果分析等功能。包含 LITHOview 作为专门的仿真可视化工具，查看光刻 aerial image、resist profile、OPC 前/后对比等。

使用场景：代工厂 RET recipe 开发和模型校准工程师。

---

## 五、掩模数据准备（MDP）

将验证并修正后的版图转换为掩模写入机能理解的格式。

### Calibre FRACTURE \*
**解决问题：** 版图格式掩模写入机不认识，如何转换成写入机格式。

将 GDSII/OASIS 格式的版图数据离散化为掩模写入机（Writer）的指令格式（MEBES、JEOL、NuFlare VSB、Hitachi、Micronic 等）。支持层次压平、图块变换、镜像/旋转、子场（Sub-field）处理。

使用场景：每颗芯片制造前必须做的数据格式转换。

### Calibre MDPverify \*
**解决问题：** 离散化后的数据有没有错误。

将 Fracture 后的结果与原始 GDS/OASIS 逐图形比对，检测是否出现图形缺失、移位、变形等问题。

### Calibre MDPmerge \*
**解决问题：** 多张掩模的 Job Deck 如何合并。

将多个 FRACTURE 输出合并为一个 Job Deck，支持多 Writer/多层的统一管理。

### Calibre MDPstat
**解决问题：** 掩模数据质量如何。

对 Fracture 后的数据做统计：图形数、数据量、密度分布、最小图形尺寸等，用于预判掩模制造难度。

### Calibre MASKOPT \*
**解决问题：** 掩模上的图形能否进一步优化以改善可制造性。

对掩模图形做规则驱动的优化，如增加辅助图形、平滑锐角、填充密度控制等。

### Calibre MPCpro
**解决问题：** 掩模制造过程本身也有工艺偏差，如何补偿。

Mask Process Correction——补偿掩模写入（EBL）和刻蚀过程的效应，确保掩模上的图形与预期一致。支持 Curvilinear（曲线）MPC。

---

## 六、基础平台与辅助工具

### Calibre DESIGNrev \*
除了用作布局查看器，DESIGNrev 还提供一套完整的 Tcl 脚本 API，支持版图自动化操作：几何图形生成/编辑、层次操作、数据格式转换、宏录制和执行。可以通过 `calibredrv -shell` 以无图形界面模式运行。

### Calibre CalScope
基于 Web 的 Calibre 运行管理和仪表盘工具，用于查看运行状态、资源占用和历史记录。

### Calibre Query Server
Tcl 驱动的版图数据库查询服务，可以按层 / Cell / 坐标等条件检索设计数据。

### Calibre MAPI
Calibre 的编程接口（API），允许第三方工具集成 Calibre 的各类功能。

### Calibre DRA (Dynamic Resource Allocator) \*
**解决问题：** 大规模 Calibre 作业如何高效利用计算集群资源。

DRA 根据作业的内存需求和 CPU 需求自动分配 LSF/SGE/OpenLava 集群资源，支持内存限制和优先级控制。

### CalCM (Calibre Cluster Manager) \*
跨节点资源管理组件，统一管理多台服务器的 Calibre 作业分发和监控。

---

## 产品命名速记

站在使用者角度看，Calibre 产品可以按验证流程分为几个阶段：

```
Design → DRC (nmDRC) → LVS → PERC → LFD → CMP → MP → OPC → SRAF → MDP FRACTURE → Mask
           ↕              ↕        ↕
        RealTime      xACT    Pattern
        Interactive    xRC    Matching
        eqDRC         xCalibrat
        Patterning
```

核心规律：
- **nm** 前缀 = Calibre 的层次化引擎系列（nmDRC、nmOPC、nmSRAF、nmModelflow）
- **x** 前缀 = 提取系列（xACT、xRC、xCalibrat）
- **RealTime** = 在版实时系列
- **MDP** = 掩模数据准备系列
- **RET/OPC** = 分辨率增强系列

---
