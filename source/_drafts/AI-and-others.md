AI 行业是未来整体的方向

但是 AI  行业会包括方方面面，本文希望对这个行业有一些稍微的了解

AI 不仅仅包括 LLM（也就是大模型），但是最吸引眼球的是 LLM

本文希望能探索 AI 整个行业，从上游到下游可能的产业，公司等，由于能力有限，难免会有错漏，算是一个快照吧。

> 部分由 LLM 产生的结构，后面需要确认或者调整

1. 上游：硬件与基础设施（支持 LLM 训练与推理）
上游聚焦计算资源和制造，支持 LLM 的海量参数处理（当前 LLM 参数量已超 5000 亿）。 关键链路包括半导体制造 → 芯片设计 → GPU/加速器 → 数据中心基础设施。供应链瓶颈（如 TSMC 产能）是主要挑战。

链路,标杆公司,标杆产品,产品特色,竞争力
"半导体制造（晶圆代工）
（基础硅片加工，先进节点如 3nm/2nm）",TSMC（台积电）,N3E/N2 工艺节点,支持高密度集成，EUV 光刻技术降低功耗；产量高，适用于多芯片模块 (MCM) 封装。,全球 90% 先进节点份额；客户多样化（NVIDIA、AMD、Apple），抗风险强；但地缘政治依赖台湾。
"芯片设计工具 & IP 核心
（EDA 软件与神经网络 IP）",Synopsys / Cadence,Fusion Compiler / Cerebrus AI 工具,AI 优化设计流程，自动化布局布线；集成 IP 如 Arm Neoverse 核心，支持并行计算。,高切换成本（知识产权壁垒）；加速设计周期 20-30%；与 TSMC 深度合作，提升供应链效率。
"GPU/加速器设计
（AI 专用芯片，处理 LLM 并行计算）",NVIDIA,H100 / B200 GPU（Blackwell 架构）,每瓦性能领先，支持 NVLink 互联；CUDA 生态优化 LLM 训练，处理万亿参数模型。,90% AI GPU 市场份额；软件锁定（CUDA 如 iOS）；年迭代快，2025 年出货超 500 万片。
,AMD,MI300X / MI325X,高带宽内存 (HBM3e) 集成，性价比高；支持 ROCm 开源软件栈。,作为 NVIDIA 备选，价格低 20-30%；LLM 开发者青睐多样化供应；CEO Lisa Su 领导下快速追赶。
,Intel,Gaudi 3 / Xeon 6,FPGA 基加速器，集成 AI 引擎；支持混合 CPU-GPU 工作负载。,Foundry 服务重建中，与 NVIDIA 合作嵌入 GPU；边缘 AI 强，成本低但性能落后 10-20%。
"数据中心基础设施
（服务器、冷却、网络）",Dell / HPE,PowerEdge XE9680 / ProLiant DL380,液冷支持 8x GPU 集群；模块化设计，易扩展到 PB 级存储。,端到端集成，OEM 定制强；液冷降低能耗 40%，适应 AI 高密度负载。
,Broadcom,Tomahawk 5 交换芯片,InfiniBand 支持非阻塞互联；带宽达 51.2 Tbps。,网络占数据中心支出 15-20%；NVIDIA 备选，降低锁定风险。
"云基础设施
（ hyperscaler，提供 GPU 云服务）",AWS / Azure / Google Cloud,EC2 P5 / Azure NDv5 / Vertex AI,按需 GPU 集群；集成安全合规工具。,规模经济，AWS 占 31% 云市场；Azure 与 OpenAI 深度绑定。


下游强调 LLM 在企业场景的落地，如自动化和个性化。2025 年，企业 LLM 采用率达 78%，市场规模超 1300 亿美元      https://aloa.co/ai/comparisons/llm-comparison/best-enterprise-llm-solutions



总结：建设一个 AI 数据中心需要的“购物清单”
核心系统	具体需求	关键上游组件	全球/行业标杆供应商
电力	稳定、高密度供电	变压器、中高压开关	Schneider, Eaton, ABB
		UPS、备用电池	Vertiv, Tesla (Megapack)
制冷	液冷 (AI 核心趋势)	冷板、CDU (分配单元)	Vertiv, CoolIT, nVent, 英维克
		快接头、冷却液	Parker, 3M, Shell
网络 (内)	GPU 间高速互联	800G/1.6T 光模块	Innolight (中际旭创), Coherent, Finisar
		光芯片 (激光器/DSP)	Lumentum, Broadcom, Marvell
		光纤光缆	Corning, CommScope
集成	服务器组装	AI 服务器/整机柜	Foxconn (工业富联), Quanta, Supermicro

一句话投资/关注逻辑：
目前最紧俏、技术迭代最快、与 AI 算力增长呈强正相关的环节是 液冷温控（Vertiv） 和 高速光互联（光模块/Innolight）。因为 GPU 越多，卡越热，通信需求越大。


中际旭创（InnoLight）和新易盛这样的光模块厂商，本质上是高端精密组装与集成商。

光模块的成本结构中，上游芯片（光芯片 + 电芯片）占据了 60%-70% 的成本。这些上游供应商掌握着极强的话语权。

我们可以把光模块拆开，看看里面最值钱的零件是谁造的：
- 第一层：电芯片（The Brains）—— 利润的大头
      竞争态势： 中际旭创和新易盛主要是在给 Broadcom 和 Marvell 打工。谁能更早拿到 Broadcom 的最新 5nm/3nm DSP 芯片，谁就能更早出货光模块。
- 第二层：光芯片（The Heart）—— 物理的极限
  发射端芯片      EML：800G 的主流，技术难。     Lumentum (全球龙头，苹果也是客户)、Coherent (自产自销)、Mitsubishi (三菱电机)、Sumitomo (住友电工)。<br>国产突破： 源杰科技 (Yuanjie)（正在攻克高端 EML）。
                  VCSEL（短距离/多模）：用于机柜内部连接   Broadcom/Lumentun 长关华芯
- 第三层：无源器件与精密件（The Skeleton）—— 隐形冠军的聚集地
- 第四层：PCB 与 外壳（The Body）


如果把 AI 产业链 比作一家餐厅：

    NVIDIA (食客)： 点菜的人，而且只吃最贵的（H100/GB200）。

    中际旭创 (主厨)： 负责把菜做出来（组装光模块），通过 NVIDIA 的口味测试。

    Broadcom/Marvell (顶级食材商)： 卖顶级牛肉（DSP 芯片）的，垄断货源，价格死贵，主厨没它不行。

    Lumentum/Coherent (生鲜供应商)： 卖海鲜（光芯片）的，技术含量高。

    天孚通信 (刀具与摆盘供应商)： 提供顶级厨具（精密光器件）的，虽然不卖肉，但主厨离不开它的精密工具，而且它利润很高。
  接收端芯片       PD                        Hamamatsu    Kyosemi
