---
title: SecLLM
date: 2024-03-31 11:39:36
categories:
- Theories
- AI
tags:
- NLP
password:
Guoyulab@123
---
# 集团网络安全大模型建设方案

本文主要内容是关于目前网络安全垂直领域大模型情况调研以及集团大模型额度建设方案设想。

<!-- more -->

# 基础情况调研

## 大模型

- 大模型是指具有大规模参数和复杂计算结构的人工智能算法模型。这些模型通常由深度神经网络构建而成，拥有数十亿甚至数千亿个参数。大模型的设计目的是为了提高模型的表达能力和预测性能，能够处理更加复杂的任务和数据
- 大模型在各种领域都有广泛的应用，包括自然语言处理、计算机视觉、语音识别和推荐系统等。大模型通过训练海量数据来学习复杂的模式和特征，具有更强大的泛化能力，可以对未见过的数据做出准确的预测
- 目前所说的大模型主要指的是大语言模型，近一年来又出现了多模态大模型
- 想要训练一个通用大语言模型需要经过下图中所示的步骤：
    
    <center>
	    <img src="/img/SecLLM/SecLLM1.png" width="850">
    </center>
    
    - 第一步是预训练，目前是基于 transformer 架构，如 GPT、BERT、LLaMA 等；
    - 第二步是通过精心构造的问答对数据，对预训练模型进行有监督的微调，培养模型执行具体的下游任务；
    - 最后两步是 HFRL（Human Feedback Reinforcement Learning，人类反馈强化学习）阶段，主要目的是以人类专家角度对预训练大模型的生成内容进行打分，从而给予大模型正/负反馈，协助大模型生成越来越好的内容。

## 应用现状

### 应用模式

- 目前大模型主流的应用模式主要分成两种：
    - 已有系统的 LLM 拼图：将模型集成于已有的产品或服务中，通过提升某一个环节的智能能力，实现整体系统的效率提升，降低成本。例如，原本生产体系中需要人力投入的环节，可由大模型代替或辅助；
    - 围绕 LLM 的全新体系：脱离已有的智能产业独立发展，围绕大模型建立独立的产业体系，形成智能能力（简称“智力”）的生产和消费模式。

### 应用实例

- 腾讯
    - 混元大模型（闭源）
    - 应用模式为将模型集成于已有的产品或服务中，致力于提升已有产品的效率和体验，腾讯云、腾讯广告、腾讯游戏、腾讯金融科技、腾讯会议、腾讯文档、微信搜一搜、QQ 浏览器等 50 余个腾讯业务和产品，已经接入混元大模型测试，并取得初步效果
- 字节跳动
    - 云雀大模型（闭源）
    - 应用模式为围绕大模型建立独立的产业体系，致力于开发 AI 原生应用，成为 AI 时代的应用工程，已上线豆包、扣子等 AI 原生产品外，还将推出 AI 角色互动 APP “话炉”，以及一款或为图片方面的 AI 产品 ”PicPic”
- 华为
    - 盘古大模型（闭源）
    - 应用模式为围绕大模型建立独立的产业体系，盘古大模型3.0是一个完全面向行业提供服务，以行业需求为基础设计的大模型体系，致力于深耕行业，让人工智能服务好千行百业、服务好科研创新
- 阿里
    - 通义千问大模型（开源）
    - 应用模式为围绕大模型建立独立的产业体系，利用开源生态的力量建立完整的大模型生态系统

### 安全应用

- Google Sec-PaLM
    - 利用 Google 威胁情报和 Mandiant 对漏洞、恶意软件、威胁指标和行为威胁行为者特征的前沿情报数据对 Google PaLM 模型进行微调后的大模型
        
    <center>
	    <img src="/img/SecLLM/SecLLM2.gif" width="850">
    </center>
        
- ChatGPT
    - 自从 OpenAI 开放 GPT-4 以及 GPTs 的模式之后，出现了许多基于 ChatGPT 的安全小工具。
    - 这些工具主要聚焦于单点安全问题的解决，比如：
        - ChatGPTScanner - 由 ChatGPT 提供支持的白盒代码扫描
        - chatgpt-code-analyzer - 适用于 Visual Studio Code 的 ChatGPT 代码分析器
        - vulchatgpt - 使用 IDA Pro 反编译器和 ChatGPT 来查找二进制文件中可能的漏洞
- Microsoft security copilot
    - 通过⼈⼯智能驱动的⽣成助⼿，以⼈⼯智能的速度和规模进⾏保护
    - 下图表简化地展示了在 Copilot for Security 背后发⽣的事情。
        
        <center>
	        <img src="/img/SecLLM/SecLLM3.png" width="850">
        </center>
        
        - 当用户提交⼀个提⽰时，⾸先，Copilot 协调器确定上下⽂，并利⽤ Copilot 可⽤的技能构建计划
        - 然后它执⾏计划并收集所有必要的内容和数据
        - 接下来，它结合数据和上下⽂，格式化数据，制定出响应，并交付该响应
- Dropzone AI
    - 为繁忙的 SOC 预训练的智能体（AI Agents）自主执行端到端调查
        - 收集：收到警报后，智能体会连接分散的安全工具和数据堆栈。不知疲倦地查找、获取相关信息，并将其反馈到 LLM 原生系统
        - 理解：Dropzone 的网络安全推理系统，专门建立在先进的 LLM 之上，针对每个警报运行完整的端到端调查。其安全预训练、组织背景理解和防护措施使其具有高度准确性。
        - 总结：然后，Dropzone 会生成一份完整的报告，包括结论、执行摘要和用简单英语写成的全面见解。您还可以与聊天机器人对话，进行临时查询。
- 奇安信/深信服/绿盟
    - 以 SOC 平台为依托，提供智能研判、智能调查、智能问答、智能报告等辅助能力
- 360
    - 提出安全大脑的概念，除了提供智能安全专家角色辅助日常安全运营外，支持在 SOAR 剧本编排中添加智能节点，提升场景执行的智能化程度

### 发展趋势

- 从应用模式看：
    - LLM 作为拼图的应用模式，从替换传统低效、重复性工作，到可简单的识别意图从而调用工具，再到可简单快速的进行动作决策、场景编排
    - LLM 作为独立产业的应用模式，从对话平台，到增加互联网自主搜索能力，再到通过外挂知识库实现的 RAG 方案，以及现在如 GPTs 一样可自行制作专用 ChatGPT 子应用
- 两种应用模式逐渐走向了同一个趋势“智能体”

## 智能体

### 概念解释

- 智能体或者智能代理（Intelligence Agent）是一种抽象的概念和说法，最早其实来源于哲学领域。从一般意义上讲，Agent 是指具有行动能力的实体，Agent 的概念涉及自主性，有着行使意志、做出选择和采取行动的能力，而不是被动地对外部刺激做出反应。
- 在人工智能领域，Agent 是一种计算实体。从本质上讲，AI Agent 并不等同于哲学上的 Agent；相反，它是 Agent 这一哲学概念在人工智能领域的具体化。我们将 AI Agent 视为能够使用传感器感知周围环境，做出决策，然后使用执行器采取行动的人造实体。

### 发展趋势

- AI 智能体的发展趋势大体如下：
    
    <center>
	    <img src="/img/SecLLM/SecLLM4.png" width="850">
    </center>
    
    - 从最开始的符号式 Agents，典型代表为基于知识库的专家系统
    - 接着是反应式 Agents，根据环境的变化作出相应行动
    - 然后是如 AlphaGo 一般基于强化学习技术的 Agents
    - 再下面是结合了迁移学习和元学习技术的 Agents
    - 大语言模型横空出世之后，目前基于大语言模型的 Agents 成为全新的研究方向

### 未来方向

- 根据英伟达公司研究员 Jim Fan 的定义，通用智能体是一个掌握广泛技能，控制许多身体，并能够泛化到多个环境中的单一算法
- 目前的 AI 技术的发展趋势无一例外都遵循着从专家模型到通用模型，再到专业化的通用模型这一发展规律。以网络安全领域为例：
    - AI技术在这一领域最开始的应用是以异常检测算法为主的小模型
    - 到后来通用大模型的出现，开始尝试将通用大模型与行业相结合
    - 但是在结合的过程中发现通用模型的安全领域能力并不出色，于是开始研究网络安全领域的垂直大模型
- 未来通用智能体的发展方向，其实就是以垂直领域大模型为基础，在以下3个方面不断扩展延伸：
    - **技能**：智能体能解决任务的数量
    - **化身**：智能体能够控制的身体形态的多样性，通俗来说就是智能体能够接受多模态的输入
    - **现实**：智能体能掌握的虚拟或物理空间的数量，换句话说其实就是智能体能够理解和应对的场景数量
- 如下图所示：
    
    <center>
	    <img src="/img/SecLLM/SecLLM5.png" width="850">
    </center>
    
    - 从 AlphaGo 开始，它在三个维度上的能力及其有限，只是一个面对单一场景，解决专项问题的专项算法
    - Voyager 来自于我的世界这款游戏，它被定义为游戏中一个可以自主学习各项技能的智能体，能够根据环境的反馈不断学习新的技能
    - MetaMorph 是英伟达推出的一种控制多个不同形态机器人的算法，它实现了化身的多样性
    - IsaacSim 作为英伟达的模拟平台，能够不断的模拟现实世界的各种场景，那么它很有可能泛化到真实的物理世界
    - 最终的发展趋势，也就是基础智能体，在3个维度同时都具有很强的能力

# 安全大模型设想

## 整体架构

- 安全离不开业务，所以安全大模型也不能脱离集团大模型。
- 因此我们设想的集团大模型整体框架如下所示：
    
    <center>
	    <img src="/img/SecLLM/SecLLM6.png" width="850">
    </center>
    

## 基础设施层

- 基础设施层为大模型的训练和推理提供必要的基础设施和高效的管理，涵盖数据和算力两部分。

### 大数据仓库

- 大数据仓库主要用于存储大模型训练需要的语料，数据类型涵盖通用数据、行业数据和领域数据等。
- 对于安全大模型来说，大数据仓库需要具备多源异构数据的接入能力，即允许从不同类型额数据源中获取信息,如网络流量、系统日志、恶意软件样本等。这有助于构建更全面、综合的数据集，为模型提供更深入的理解和分析能力，从而更准确的检测潜在威胁。

### 算力集群

- 算力集群作为整体框架的硬件底座，为整体框架提供算力支持和调度，分为训练集群和推理集群。
- 算力集群需要有高效的 CPU / GPU 资源调度技术，优化计算资源的使用，确保模型训练和推理过程中的高性能。通过合理的资源分配，使得模型能够在相同时间内处理更多的数据，提高训练速度、检测速度和响应能力。
- 训练集群需要有高速的分布式训练技术，支持并行训练，加快模型的训练速度。这有助于提高模型的适应性，更快的适应新的威胁和变化。
- 推理集群需要支持模型推理加速和效率优化技术，降低模型的计算复杂度，减少计算资源的需求。这有助于提高模型效率，使其能够在较小的资源开销下进行更快速的分析。
- 目前对于算力集群的建设，主流方案主要有：
    - **英伟达 Nvidia**
        - 业界标杆，生态完备
    - **华为昇腾 Ascend**
        - 大模型生态
            
            <center>
	            <img src="/img/SecLLM/SecLLM7.png" width="850">
            </center>
            
        - 开发框架
            - 昇思MindSpore
                
                [MindSpore](https://www.mindspore.cn)
                
            - 第三方框架
                
                [昇腾社区-官网丨昇腾万里 让智能无所不及](https://www.hiascend.com/zh/software/ai-frameworks)
                
                - 昇腾 PyTorch
                - 昇腾 TensorFlow
        - 训练、部署组件
            - MindX DL
                
                [昇腾社区-官网丨昇腾万里 让智能无所不及](https://www.hiascend.com/zh/software/mindx-dl)
                
            - ModelZoo
                
                [开发者主页-昇腾社区](https://www.hiascend.com/software/modelzoo)
                
    - **昆仑芯**
        - 整体生态
            
            <center>
	            <img src="/img/SecLLM/SecLLM8.png" width="850">
            </center>
            
            - 软硬件工具栈：
                - 昆仑芯 SDK / 昆仑芯 Anyinfer
                - 飞桨（PaddlePaddle）对其支持较成熟
                - PyTorch / DeepSpeed
    - **摩尔线程**
        - 整体架构
            
            <center>
                <img src="/img/SecLLM/SecLLM9.png" width="850">
            </center>
            
        - MUSA 软件栈
            
            <center>
                <img src="/img/SecLLM/SecLLM10.png" width="850">
            </center>
            
            - 兼容 CUDA
        - 摩尔线程智算中心解决方案 - MTT KUAE 集群
            
            <center>
                <img src="/img/SecLLM/SecLLM11.png" width="850">
            </center>
            
            - MTT KUAE Platform
                - 用于 Al 大模型训练、分布式图形渲染、流媒体处理和科学计算的软硬件一体化平台，深度集成全功能 GPU 计算、网络和存储，提供高可靠、高算力服务。
                - 通过该平台，用户可灵活管理多数据中心、多集群算力资源，集成多维度运维监控、告警和日志系统，帮助智算中心实现运维自动化。
            - MTT KUAE ModelStudio
                - 覆盖大模型预训练、微调和推理全流程，支持所有主流开源大模型。
                - 通过摩尔线程 MUSIFY 开发工具，可以轻松复用 CUDA 应用生态，内置的容器化解决方案，则可实现 API 一键部署。
                - 该平台意在提供大模型生命周期管理，通过简洁、易操作的交互界面，用户可按需组织工作流，大幅降低大模型的使用门槛。
    - **天数智芯**
        - 整体架构
            
            <center>
                <img src="/img/SecLLM/SecLLM12.png" width="850">
            </center>
            
        - 天数智芯软件栈
            - FlagPerf
                
                [GitHub - FlagOpen/FlagPerf: FlagPerf is an open-source software platform for  benchmarking AI chips.](https://github.com/FlagOpen/FlagPerf)
                
            - PaddlePaddle / PyTorch / DeepSpeed / Nvidia Megatron
    - **沐曦**
        - MXMACA 运算平台
            - 该运算平台提供了一种简单易用的类C编程语言，供用户为MXMACA架构编写程序，使其在METAX GPU处理器上以超高效率运行。该编程语言能够兼容主流的C/C++异构计算语言。
            - MXMACA异构计算平台支持多种开源技术，包括AI神经网络框架（TensorFlow/PyTorch等）、库（Blas/DNN等）和Linux Kernel支持等。

## 模型孵化层

- 网络安全的实质是业务安全，网络安全大模型不能够脱离对集团业务相关知识数据的理解，因此我们认为建设网络安全大模型的前置条件是建设涵盖集团整体知识数据的基座大模型。
- 在集团基座大模型的基础上，通过调整安全、业务和通用数据的配比建设网络安全大模型。

### 集团模型

- 集团模型是指以通用数据、领域数据和集团数据等为训练语料建设而成的基座大模型。形态应有自然语言基座大模型和多模态基座大模型，用于不同的应用场景。
- 目前垂直行业大模型的几种训练策略有：
    - **从零训练**：使用通用数据和领域数据混合，从头开始（from scratch）训练一个大模型，最典型的代表是 BloombergGPT
    - **二次预训练**：在一个通用模型的基础上做二次/继续预训练（continue pretrianing），最典型的代表是 LawGPT
    - **微调**：在一个通用模型的基础上进行有监督微调（Supervised Fine-Tuning），这也是现在开源社区最普遍的做法
    - **通用大模型 + 向量知识库**：通用大模型加上领域知识库，针对通用大模型学习过的领域知识比较少的问题，利用向量数据库等方式根据问题在领域知识库中找到相关内容，再利用通用大模型强大的总结和问答能力生成比较流畅的回复
    - **上下文学习（In context learning）**：随着通用大模型能够接受的上下文窗口越来越大，将领域知识直接放在提示词（prompt）中，同样利用通用大模型强大的总结和问答能力生成比较流畅的回复
- 最后两种策略不涉及到对通用大模型的调教，自然也无从谈起自有的集团基座大模型
- 通用大模型微调的方案虽然见效比较快，且对于数据量大小的需求也不是很严苛，但是能够达到的上限不高
- 从零训练和二次预训练的方案比较适合建设行业基座大模型，两种方案的数据量和算力资源需求也不相同
    - 根据模型参数量大小，两种方案的算力需求最低推荐为百卡起步
    - 数据方面除了要考虑数据量大小外，不同类型数据的配比以及数据清洗处理后的质量对于最终模型的效果至关重要，一般认为二次预训练中领域数据的比例在 10% - 15% 比较合适

### 孵化流程

- 包括集团模型在内，以及以集团模型为基础衍生而来的领域专用模型，都遵守统一的孵化流程，以安全大模型为例：
    - **需求**：需求阶段要明确需求，确定安全大模型本轮的训练目标，如
        - 通过二次预训练补充特定方向的安全知识
        - 通过微调训练出更适合下游任务的专业安全大模型
    - **数据**：明确需求后，相应的数据类型和配比也能够明确
        - 二次预训练任务的数据类型需要和基座大模型预训练时保持一致，特定方向的安全领域数据占比控制在 10% - 15%，最多不超过20%
        - 微调任务中特定方向的安全领域数据量不大的情况下，和通用数据的比重可为 1:1
    - **算法**：任务类型明确之后，算法阶段就是选择合适的算法
        - 二次预训练主要借助于诸如 DeepSpeed / PaddlePaddle 等框架实现
        - 目前主流的微调算法有 LoRA / QLoRA / P-Turning / Freeze 等
    - **训练**：训练阶段即是通过分布式技术在训练集群中并行训练模型
    - **评估**：模型训练完成之后，需要对模型进进行能力评估，主要涉及的方面有
        - 基座大模型能力的保持程度，即训练后的模型是否依然保持很强的总结、问答、命令执行等能力
        - 安全领域知识的学习程度，即对于准备的领域知识，大模型的理解吸收能力
    - **测试**：测试阶段主要是测试训练后的大模型
        - 是否存在严重的记忆遗忘、“幻觉”等问题
        - 是否存在泄露隐私数据等数据安全问题
    - **打包**：符合部署标准的训练后大模型按照应用部署环境进行打包
    - **部署**：根据具体环境进行部署，如提供 API 接口、嵌入式部署等
    - **发布**：确定数据，模型，应用之间的交互流程之后即可发布
    - **监控**：监控部署后模型整个生命周期中的信息，如推理算力使用情况、数据流转情况等
    - **反馈**：建立大模型运行过程中的用户反馈机制，收集运行中遇到的实际问题及反馈，反馈数据可用于下一步大模型的优化阶段
    - **需求**：根据实际运营中的反馈，针对收集总结的问题，转换成具体的需求，开始新一轮的孵化流程

### 领域模型

- 在集团大模型的基座上，通过调整数据类型配比，衍生出各个子领域专用大模型，比如业务大模型、安全大模型等。
- 在这些子领域专用大模型的基础上，通过针对性的微调，使得这些专用大模型转变成为专家大模型。
    - 比如业务大模型可以转变成精通电力知识的专家大模型，或者熟悉电力调度的专家大模型等
- 其中从安全大模型衍生而来的专家大模型，主要可分为以下五个方向：
    - **安全运营**：安全运营数据清洗，多源异构数据提炼，告警分析智能研判，威胁事件智能解读等
    - **威胁检测**：网络攻击报文分析，终端异常行为分析，恶意代码检测，社工钓鱼检测等
    - **安全防护**：情报联防联控，安全事件溯源等
    - **攻防实战**：资产暴露面收集，漏洞利用评估，漏洞检测工具生成，响应处置咨询等
    - **风险评估**：风险模式识别，风险关联分析，资产风险识别，拓扑关系分析等

## 模型能力层

- 完成构建集团安全领域大模型之后，模型能力层主要负责的是将安全大模型的能力通过简单易配置的框架，以接口或服务等方式为各个应用场景赋能。

### 模型部署框架

- 模型部署框架主要负责安全大模型本身的推理部署。
- 为了应对多种多样的部署环境，除了支持模型高精度推理外，还需要支持通过模型量化，蒸馏和剪枝等技术，在尽可能保持模型原本能力的基础上，减少模型的资源占用，提升推理速度。
- 部署模式除了支持常见的 API 接口的方式，同时需要支持嵌入式部署优化模型的方式，满足某些特殊场景中将大模型技术与端点设备集成的需求。

### Agent 配置框架

- 模型能力最关键的模块是 Agent 配置框架，该框架需要支持智能体的快速定义和生成，提供即插即用的应用接口，场景模式的可视化编排，以及智能体全生命周期的监控，并能根据监控数据和反馈信息，及时由安全专家介入干预和修正。
- 智能体的架构主要可以分为 3 个模块，脑模块，感知模块和行动模块。
    - 脑模块作为智能体的中央处理核心
        
        <center>
            <img src="/img/SecLLM/SecLLM13.png" width="850">
        </center>
        
        - 它的首要功能是自然语言交互功能，其中涉及到核心问题有：
            - 多轮交互下的信息有效性和统一性问题
            - 高质量自然语言生成问题
            - 语言中隐藏含义的理解问题等
        - 第二个功能涉及到知识领域，每个智能体需要了解对应的知识，大体上可分为 3 种：
            - 语言知识，意味着智能体需要了解输入的统一数据表示的意义
            - 常识知识
            - 专业领域知识
        - 第三个是记忆功能，如何保证多轮交互以及多 Agent 交互之后的信息同步是很重要的课题。为此可能的解决办法有：
            - 提升 transformer 模型的上下文长度限制
            - 将信息抽象提炼，再存入记忆模块
            - 压缩信息，寻找更为高效的信息表示方法
            - 共享记忆，类似外挂知识库的方案
        - 第四个是推理和规划功能，旨在培养智能体形成人类一样的链式思维，一步一步的进行推导和规划
        - 最后是学习功能，为了形成自洽的系统，智能体必须要自主学习，从而摆脱必须依赖人类指令运行的情况
    - 感知模块，这个模块相对而言比较简单明了，就是将多源异构的数据转换成统一的数据表示
        
        <center>
            <img src="/img/SecLLM/SecLLM14.png" width="850">
        </center>
        
    - 行动模块，主要分为 2 个子功能：
        
        <center>
	        <img src="/img/SecLLM/SecLLM15.png" width="850">
        </center>
        
        - 第一个是文字或图像输出功能，即将抽象的统一数据转换成人类理解的自然语言或图像
        - 第二个是工具模块。这里主要考虑的是在响应和恢复阶段，智能体需要能够根据分析研判结果，调用甚至制造对应的工具实现攻击阻断和风险修复等工作任务
- 此外，需要考虑整个智能体系统内的信息交互模式
    
    <center>
	    <img src="/img/SecLLM/SecLLM16.png" width="850">
    </center>
    
    - 首先对于单个 Agent 来说，主要分为 3 种情况：
        - 第一种是任务导向，那么这个智能体只需要接受输入，根据人类指定的行为范式给出输出
        - 第二种是灵感导向，意思是人类给出一个目标，智能体从结果逆推，自行寻找需要的输入并完成目标
        - 第三种是生命周期导向，是指类似 AutoGPT 那样的实体，自身实现一个任务的闭环运行无需人类的指令
    - 其次，对于 Agents 之间的交互场景概括起来其实很简单，就分为两种情况：
        - 一是多个 Agents 合作完成任务的模式
        - 二是 Agents 之间通过对抗的模式相互促进
    - 最后是人机交互，也分为两种情况：
        - 一是指导-执行范式。指人类给出指令，指导智能体执行对应动作、
        - 二是合作范式，指人类和机器合作完成相应任务目标

## 场景应用层

- 场景应用曾主要是具体的应用场景，即以模型本身或者智能体的形态为具体的应用或落地场景进行赋能。

### 安全运营

- 安全智能问答
    - 理解用户的提问，在答案中进行检索或者生成答案以满足用户的需求。在安全领域，智能问答系统的价值体现在其可以针对用户安全场景对安全专业知识、安全产品与解决方案等方面问题进行回答。
    - 为安全专业人员和普通用户提供高质量、高效率、智能化的安全问题解答和支持，帮助他们更好地理解安全问题、协助做出安全决策。同时结合知识图谱辅助问句理解，借助只是图谱中节点的属性及关系，通过命名实体识别等技术发现问句中的实体，进而更好地理解问题。完成意图理解后，通过图匹配从安全知识图谱检索相关的实体作为应答，同时通过信息检索获得文本应答，最后将实体应答与文本应答拼接形成回复答案。
- 多源情报整合
    - 整合多个不同来源的安全情报信息，提供更准确、全面的威胁情报，帮助组织更好地应对安全威胁。
    - 借助安全大模型地语义理解能力实现信息总结、情报归类、实体标定等。信息总结功能主要涉及到阔平台信息整合、垃圾信息过滤、重点提炼等。通过安全大模型对信息进行语义分析和关联性识别，将来自不同渠道的多种形式的信息汇聚为有机的整体，消除信息碎片化问题，从而提高信息整理的效率和全面性。接下来对于情报之间的关联点、相似性，将其有序归类，实现情报梳理和整合的自动化。最后对信息中的现实实体（包括但不限于设备、人员、组织、漏洞、事件等），分析其性质、热度、上下文，以及实体之间的关联性，综合考量后给出初步判断（包括其风险度、紧急度等内容），为情报分析提供更深入的视角。
    - 最终目标是在多来源、多模态信息上提供出色的智能归纳和综合分析能力，有效提高情报信息额利用效率和价值，为决策者提供更全面、准确的情报信息支持。
- 安全日志智能解析
    - 传统的安全信息和事件管理（SIEM）系统，包括安管平台、SOC平台等，日志解析是一个关键卡点，目标整合来自多个安全设备和应用程序的日志数据，并提供集中化的监控、分析和报告功能。
    - 基于规则将各种日志的解析内置 SIEM 系统，遇到新的日志时就需要补充新的解析规则。为了减少这部分工作量，一种方案是将目前 SIEM 系统的解析日志和规则的收集起来，用于训练安全大模型或智能体自动化生成解析规则用于新的日志。另一种方案是通过综合分析新型日志的内容，如果 SIEM 可以正确解析不同的日志，则跳过现有解析规则，直接将解析完毕的日志结果转换成为 JSON 格式对接 SIEM 系统，使其能够准确地提取有用地信息，并将其转化为可供安全分析和威胁检测的形式，也即将安全大模型或智能体加强日志解析的能力直接输入到 SIEM 系统，由 SIEM 系统提供日志检索和进一步的威胁分析。
- 安全事件分析研判
    - 将安全大模型机器衍生的智能体应用于安全事件分析研判，协助安全运营人员快速、准确、全面地识别和评估威胁攻击事件，解决安全领域中的安全告警和日志数据处理难题、多维度数据关联复杂和复杂威胁识别难题。
        - 在准备训练样本时，应当尽可能覆盖各种类型的安全事件。在仿真环境中采集黑样本是必要但远远不够的。而从真实环境中采集数据时应注意，安全事件数据分布会随网络环境、业务种类、防护策略等因素差异很大，应避免从单一环境中进行采集。
        - 数据标注方面，需要由人类专家对安全事件进行标注，虽然成本很高，却是唯一实际有效的方法。人工标注中一般需要进行抽样，需要首先设计并验证抽样方法的有效性。需要特别注意的是，标注数据集应当平衡正负样本数量，尤其需要尽可能完善各类正常业务活动产生的日志及其标注，而不能只关注攻击事件的日志。
        - 由于安全事件数据本身并非自然语言，如果直接输入以自然语言语料训练的基础模型，将无法有效利用预训练模型中的知识。需要设计并持续优化一套特征方法，用以将安全事件数据转换为模型易于处理的形式。
    - 安全大模型或智能体预期能够实现的部分能力有：
        - 通过扩展事件的多样化，可以对不同编码和混淆的告警信息进行事件的准确研判
        - 通过 RLHF，从告警信息中提取较为深层次的抽象信息，如攻击者所使用的攻击手法、期望达到的目的、可能造成的影响等，然后将其转化为通俗易懂的买哦书形式。

### 威胁检测

- 深度威胁检测
    - 基于安全大模型，对于网络侧的流量，主机侧的文件、日志、进程行为等维度进行威胁检测：
        - **异常检测**：大模型可以通过学习正常的网络流量模式和行为特征来检测异常。当网络活动与正常模式不符时，发出警报并进行进一步的调查。
        - **恶意软件检测**：大模型可以分析文件、应用程序和代码，以检测潜在的恶意软件。它们可以识别恶意软件的特征、行为模式和代码签名，并通过比对已知的恶意软件数据库来进行分类。
        - **行为分析**：大模型可以分析用户和设备的行为模式，以检测潜在的异常活动。识别恶意操作、数据泄露和内部威胁，并采取相应的响应措施。
        - **社工攻击检测**：大模型可以分析电子邮件、消息和社交媒体活动，帮助检测可能的社交工程攻击，提高检测效率和效果。

### 安全防护

- 勒索情报挖掘
    - 借助大模型的相关能力在分析勒索威胁情报过程中解决勒索情报来源的多样性、多维度勒索情报信息整合和复杂威胁特征分析及洞察等难题，挖掘、揭示攻击者的策略和行为模式，结合资产及相关风险信息，预测可能的下一步攻击目标，协助安全团队更好地理解和应对勒索软件地威胁，提前制定防御策略。
    - 整体流程可以拆分为 3 个智能体相互协助完成：
        - **威胁情报文本主题分类智能体**：该智能体通过勒索情报主体分类的方法，快速定位和识别与勒索软件相关的情报，了解勒索软件的种类和演变趋势，为安全策略的制定和威胁预警提供更全面的情报洞察。
        - **勒索相关情报的关键信息提取智能体**：该智能体主要负责提取勒索相关的威胁情报中包括勒索软件的加密方式、攻击手段、利用漏洞、勒索情报地址和 IOCs（Indicators of Compromise）等数据。这些信息可以用于协助其他智能体快速了解勒索事件的特征和威胁程度，以便及时采取相应的安全措施。
        - **勒索智能分析处置智能体**：该智能体负责根据上一步提取的相关数据，分析事件特征和威胁程度，自动化生成防御策略，同时结合资产数据及时通报存在风险的资产。
- 智能防护助手
    - 面对安全防护工作压力大、人员不足的困境，亟需智能化安全运营工具提升防护效率。智能防护助手可以实现人机协同，进一步增强安全防御的智能化和及时性，提升威胁检测、响应和决策的能力，主力安全团队更有效地维护网络安全。
    - 衍生于安全大模型的智能防护助手能够调用相关工具执行防护人员的决策和规划，通过将决策转化为实际行动，更好地应对不同情况下的挑战。同时智能防护助手还需要及时获取工具执行后的反馈信息，从而对决策和行动进行实时的调整和优化。
    - 智能防护助手作为决策中枢，接收交互界面采集的用户多模态输入，进而完成用户意图理解以及任务管理调度的核心决策逻辑，进而调用告警数据查询、分析统计报告、告警智能分诊、告警上下文溯源等分诊加速工具，形成对专家研判过程的辅助能力，或者针对高置信度告警事件自动化处置。

### 攻防实战

- 智能攻防对抗演练
    - 攻防演练是模拟真实攻击和防御情景的训练活动。场景涵盖网络入侵、恶意代码、数据泄露等多种攻击，并验证团队的应急反应，以使组织能够迅速、高效地应对真实威胁。通过训练活动帮助组织有效地应对现实中地安全威胁和攻击，提升整体的安全性和应急响应能力。
    - 基于大模型的智能体通过自动对抗，形成长期自主进化的攻防对抗能力。这一能力源于模型在复杂攻防环境中的训练和学习，它能够识别并适应新的威胁，不断调整策略以有效应对不断变化的攻击技术。输入攻防演练需求，通过模拟各类攻击和防御场景，基于安全大模型的智能体不仅学习攻击方法，还能掌握多种防御策略，从而逐步形成多样化的应对能力。智能体可以快速感知威胁，做出智能决策。并将决策转化为实际行动，从而提高安全防御、响应的效率和准确性，最后输出演练报告进行总结复盘。

### 风险评估

- 代码漏洞挖掘
    - 漏洞挖掘和代码审计是两种密切相关的安全实践，在软件开发和安全领域中都有重要作用。漏洞挖掘分析是代码进行静态或动态的安全分析，以发现代码中存在的潜在漏洞或风险。代码审计重在对代码的静态分析，以查找可能存在的漏洞、弱点和安全风险，从而提高代码质量，降低开发成本，提升安全防护能力。
    - 借助大模型的能力可以高效的处理大量的代码数据，利用其自身强大的学习能力和推理能力，快速发现代码中的潜在漏洞，根据上下文和目标，自动地提供合适的代码建议和修改方案，提高代码的质量和安全性。
- 软件供应链安全
    - 软件供应链安全的需求涵盖了软件从源代码审查到交付和部署的整个过程，以确保软件在每个环节都是安全和可信的。这有助于减少潜在的威胁和漏洞，保护集团的数据和声誉。
    - 借助安全大模型丰富的先验知识和强大的语义理解能力，结合知识图谱，不仅能有效融合软件供应链领域中的多源信息，还能深入分析实体的上下文信息，进而提升模型对于软件供应链中实体与关系的语义理解水平。最终实现通过智能体之间的配合完成信息抽取、供应链健康评分、供应链敏感发现、智能分析预警等任务。
        - 软件供应链领域涉及众多的非结构化数据源，如 CVE、NVD、CNVD 等开源漏洞库、漏洞 Twitter、安全通告等数据。通过智能体自动化将多源数据处理成结构化数据存入知识图谱，从而对知识图谱中额缺失节点、关系进行信息补全。
        - 供应链健康评分智能体主要根据两方面的数据进行建模，其一为开源软件的社区热度、影响范围、版本维度频率、安全风险等结构化数据，其二为开源软件的 issue 文本、版本描述、更新信息等非结构化文本。智能体需要先对非结构化文本进行向量化操作，获取非结构化文本中的语义特征，然后融合语义特征和结构化字段构建健康评分模型，最后分别考虑社区热度、安全风险、影响范围、版本维护等多为信息，为供应链软件的健康程度提供综合评价。
        - 敏感信息是资产脆弱性分析的重要维度。通过对开源供应链的 git 仓库、组件官方公告、相关供应链公众号文章进行敏感信息抽取，例如电话、邮箱、身份证、IP、银行卡、URL、域名、MAC、IPv6、JDBC 等敏感实体类型，以扩充供应链图谱的敏感信息，进而有利于资产风险发现。
        - 最终结合上述流程中获取的数据，结合实际资产信息，智能分析资产风险及脆弱性，及时预警。
- 集团 EASM 评估
    - 外部攻击面管理（External Attack Surface Management，EASM）是企业安全评估中不可忽视的能力。企业安全 EASM 评估对企业的外部攻击面进行全面的管理和评估，包括发现、监测和缓解外部攻击可能的入口。EASM 确保企业的应用程序在安全性、合规性、风险管理和应急响应等方面得到有效的管理和保护。
    - 安全大模型或者衍生而来的智能体站在潜在攻击者的角度来持续不断地审视与管理组织资产和薄弱环节，包含以下三部分：
        - **持续监控和全面收集数据**：协助组织监控和处理多种数据，包括社交媒体、SSL 证书、域名信息、漏洞数据库、违规数据集、深网/暗网资源、代码存储库等。不同数据整合到安全大模型或智能体，快速识别和清理无效信息，收集和整理与组织相关地公开可见信息。
        - **智能体 + 威胁情报分析**：将大模型或智能体与威胁情报数据相结合，对外部攻击面进行威胁分析。通过与威胁情报数据地对比，及时发现可能受到攻击的系统和服务，有效协助组织及时进行攻击面收敛，将配置不当、开放端口、未修复漏洞等根据紧急程度、严重性、风险等级确定修复优先次序。除此之外，结合已有资产信息以及调用漏洞扫描等各类工具，对资产列表进行实时维护。
        - **外部攻击面评估**：利用大模型或智能体进行数据分析和挖掘，对组织的外部攻击面进行评估。将合适的数据和技术工具化，对各种风险进行排序，发现潜在的漏洞、安全风险和脆弱性，帮助安全团队识别潜在的攻击入口、攻击路径等。
- 数据安全
    - 安全大模型大模型在数据安全方面的应用可以帮助组织保护其敏感数据，并识别潜在的数据安全风险。可能的应用有：
        - **数据分类与标记**：大模型可以帮助组织自动识别和分类其数据，包括个人身份信息（PII）、医疗记录、财务数据等敏感信息。通过对数据进行标记和分类，组织可以更好地了解其数据资产，并制定相应的保护策略。
        - **数据泄露检测**：大模型可以监控数据传输和存储过程，以检测可能的数据泄露事件。它们可以识别异常的数据访问模式、异常的数据传输行为，并发出警报以及采取相应的应对措施。
        - **身份验证与访问控制**：大模型可以用于用户身份验证和访问控制，以确保只有授权用户能够访问敏感数据。它们可以分析用户的行为模式、设备信息和其他身份验证因素，并根据风险评估来动态调整访问权限。
        - **数据加密与脱敏**：大模型可以帮助组织选择适当的数据加密和脱敏技术，以保护敏感数据免受未经授权的访问。它们可以分析数据的敏感程度和业务需求，制定相应的加密和脱敏策略。
        - **数据审计与合规**：大模型可以帮助组织进行数据审计和合规性监管，以确保其数据安全措施符合相关法规和标准。它们可以分析数据访问日志、操作记录，并生成合规性报告和审计跟踪。