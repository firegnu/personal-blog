---
title: "多 Agent 编排与协作技术地图，兼 Herdr 解耦架构建议"
date: 2026-08-08
draft: false
tags: ["AI", "多 agent", "Herdr", "协议", "ACP", "A2A", "AHP"]
summary: "围绕 AHP / ACP / A2A 等协议的编排技术地图，以及对 Herdr 项目解耦为 ACP 通信层、A2A 概念层、Gate 审计自研核心三层的架构建议。"
---

# 多 Agent 编排与协作技术地图，兼 Herdr 解耦架构建议

> 整理时间：2026-08-08
> 背景：围绕 AHP（Agent Host Protocol）展开的一系列探索，延伸到 ACP、A2A、raft.build、buzz、Claude Code 原生多 agent 特性，以及一份完整的开源编排工具地图，最终落到 Herdr 项目的解耦架构建议上。

## 1. 起点：Agent Host Protocol（AHP）是什么

AHP 是微软发布的一个仍处于草案阶段的开放协议，官方定位是"a state synchronization protocol for agent experiences"。它要解决的问题是：当一个 AI agent 会话（比如一次 Claude Code 的编码会话）需要被多个客户端同时观察和交互时——例如 IDE 插件、网页控制台、手机端——这些客户端如何保持完全同步的状态视图。AHP 的核心抽象是"channel"，每个会话、每个聊天、每个终端、每个改动集都是一个 URI 标识的可订阅频道；服务器持有每个频道的权威状态树，所有变更以有序的 action 广播给所有订阅者，客户端可以乐观地本地应用自己的操作再与服务器流做一致性回收（reconciliation）。协议底层是 JSON-RPC，官方提供 Rust、TypeScript、Kotlin、Swift、Go 五种语言的客户端库。

AHP 的 doctrine 文档明确列出了一份"anti-goals"清单，其中最关键的一条是：AHP 不定义"agent-to-agent coordination semantics"。也就是说，AHP 从设计上就只管"多个人类客户端怎么看同一个 agent"，完全不管"多个 agent 之间怎么互相协作"。这个边界在后续所有讨论中都是关键判断依据。

AHP 还专门写了一篇文档对比自己和 ACP 的关系，给出了一个很清晰的类比："AHP is a mutex over ACP"——AHP 是协调层，ACP 是通信层，两者天然可以组合：一个 host 对客户端说 AHP，对 agent 后端说 ACP，中间做翻译桥接。

## 2. ACP（Agent Client Protocol）：单个 agent 与其使用者之间的标准通信协议

ACP 由 Zed 主导（文档站是 agentclientprotocol.com），定位类似"给 coding agent 领域做的 LSP"：标准化编辑器/宿主与单个 AI coding agent 之间的通信，包括初始化、鉴权、prompt、流式更新、工具调用、权限确认（`session/request_permission`）等。协议同样基于 JSON-RPC，区分本地 agent（作为宿主子进程、通过 stdio 通信）和远程 agent（走 HTTP/WebSocket，官方说这部分仍在完善中）。

ACP 官方维护了一个 Registry，列出所有支持鉴权的 ACP 兼容 agent。查证结果显示，Claude（"Claude Agent"）、OpenAI 的 Codex（"Codex"）、以及 pi（"pi ACP"）都已经有官方或社区维护的 ACP 适配器——这三个正好是 Herdr 项目里点名要管理的三个 Runtime。这是后面给 Herdr 解耦建议时最重要的一条事实依据。

需要特别提醒一个命名陷阱：IBM/BeeAI 也发布过一个同样缩写为 **ACP** 的协议（Agent Communication Protocol），但它解决的其实是"agent 与 agent 之间跨系统通信"的问题，定位更接近 A2A。查证结果显示，IBM 的这个 ACP 已经在 2025 年正式并入了 Linux Foundation 的 A2A 项目，技术并轨，不再独立发展。所以今天再提"ACP"，基本默认指 Zed 的 Agent Client Protocol；IBM 那个名字虽然还挂在网上（agentcommunicationprotocol.dev、GitHub 的 i-am-bee/acp），但已经不是一个独立演进的标准了。

## 3. A2A（Agent2Agent Protocol）：agent 与 agent 之间的任务委派协议

A2A 最初由 Google 发起，现已捐给 Linux Foundation，由 AWS、Cisco、Google、IBM Research、Microsoft、Salesforce、SAP、ServiceNow 等共同组成的技术指导委员会维护。它的定位非常明确：agent 之间如何发现彼此、委派子任务、交换结果，而且是面向"不透明黑盒 agent"设计的——双方不需要共享内部记忆、工具或专有逻辑。

A2A 的核心概念包括：Agent Card（描述一个 agent 的身份、能力、skill、鉴权方式的 JSON 元数据，用于发现）；Task（有唯一 ID、有生命周期状态机——submitted/working/input-required/completed/failed 等——的有状态工作单元）；Message 和 Part（对话轮次及其内容容器，支持文本、文件、结构化数据）；Artifact（agent 产出的具体交付物）；以及三种交互机制——请求/响应轮询、Server-Sent Events 流式、以及面向长任务和断线场景的 Push Notification（服务器主动回调客户端提供的 webhook）。A2A 官方也明确划了边界："不是 agent 开发框架，不是 sub-agent/工具调用协议，不是 MCP 的替代品"——MCP 管 agent 到工具，A2A 管 agent 到 agent，两者互补。

## 4. 命名或角色容易混淆的地方小结

这几轮讨论下来，有必要把几个容易混的概念并排放一下。MCP（Anthropic）解决"模型怎么接工具/数据"，是最底层、几乎所有其他协议都会提及并与之互补的一层。ACP（Zed）解决"宿主怎么和单个 agent runtime 对话"，是 1:1 协议。IBM 的旧 ACP 已并入 A2A，不必再单独考虑。A2A（Google → Linux Foundation）解决"agent 与 agent 之间怎么委派任务、拿结果"，是多 agent 协调协议。AHP（微软）解决"多个客户端怎么同步观察同一个 agent 会话"，是展示层协议。此外还有 AGNTCY（Cisco 主导、同样捐给 Linux Foundation 的"Internet of Agents"项目），更偏向 agent 的发现、目录、身份基础设施，与 A2A 是同盟而非竞争关系，本次没有深入查证，只作为地图上的一个坐标记录。

## 5. 产品/平台层：raft.build、buzz，以及 Claude Code 原生化的多 agent 能力

在协议之上，是把这些能力包装成给人用的产品。raft.build 官方定位是"multi-agent collaboration platform"：多个 agent（可以跑在 Claude、Codex、Hermes 等不同 runtime 上）拥有持久身份和记忆，可以认领任务、并行工作、互相交接、在共享 thread 里互相 review，同时人类和 agent 混在同一个工作区（channel/thread/task/@mention）里协作。它想解决的问题正好落在 AHP 明确声明"不管"的那条 anti-goal 上（agent-to-agent coordination），也更接近 A2A 想标准化的那一层。

buzz 是 Block（Jack Dorsey）2026 年 7 月 21 日发布的开源项目，理念与 raft.build 高度相似（人和 agent 共享同一个频道/线程，agent 是工作区里的一等公民），但技术选型不同：它跑在用户自己搭建的 Nostr 去中心化中继上，每条消息、每次 git 事件、每次 code review 都是一条带签名的事件，天然形成一份人和 agent 共用的身份体系和审计轨迹；agent 接入方式是通过 ACP（再桥接 MCP），已知支持 Claude Code、Codex、goose 等。

值得注意的是，Anthropic 自己也在把类似能力做进 Claude Code 原生功能里，只是分散成几个独立模块。**Agent Teams**（目前仍是需要手动开启的实验特性，`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`）让一个 session 作为"team lead"拉起若干独立的"teammate"（同样是 Claude Code 实例），共享一份有依赖关系的任务列表（pending/in progress/completed），teammate 之间通过基于 JSON 文件的 mailbox 互相发消息，认领任务用文件锁防止竞态，还提供 TeammateIdle/TaskCreated/TaskCompleted 三个 hook，可以在 teammate 想收工或任务想被标完成时用退出码拦截打回——这基本是一个轻量版的 Gate 机制。**cross-session messaging**（v2.1.224 起）让不同的 Claude Code session 互相发消息、用 `/list-agents`（底层是 ListAgents/SendMessage 工具）发现彼此，但这里有一个重要的不对称性：同一台机器上的 session 之间通过本地 Unix domain socket 直接双向通信，完全不经过 Anthropic 服务器；而跨机器（或跟 Claude Code on the web 的云端 session）通信则必须经过 Anthropic 服务器中转、并且**只能"回复"、不能主动发起**——也就是说本地这一端不能凭空联系一个从未联系过自己的远程 session。**Remote Control** 则对应 AHP 想标准化的那个场景：网页端、手机端、桌面端可以同时观察同一个正在跑的 session。这三者加上原有的 subagent（同一 session 内部的临时委派）、background agent（独立 worktree 隔离的持久后台任务），基本就是 Claude Code 版的"raft.build/buzz"，只是完全锁定在 Claude 自家生态里，teammate 只能是 Claude Code 实例，不能是 Codex 或 Pi。

## 6. 开源编排工具生态地图

在协议和大厂原生功能之外，还有一个庞大得多的开源工具生态，专门有一份持续更新的策划清单 `andyrewlee/awesome-agent-orchestrators` 在做普查（Herdr 本身也被收录在其中）。这份清单把工具分成几类。第一类是纯终端里并行跑多个 agent 会话的工具，靠 tmux 面板或 git worktree 隔离，代表有 claude-squad、amux、dmux，以及 Herdr 自己。第二类是带图形界面的桌面/网页版本，同样是并行会话加 diff 审阅合并，代表包括 Traycer、OpenHands、jean、Fletch 等数十个项目。第三类是"Multi-Agent Swarm"，也就是多个 agent 主动协调、通信、共同完成目标的系统，这一类和 Herdr 的定位最接近，里面几个值得重点关注的项目包括：Fusion（plan-review-execute gates 加 per-task worktree 加分层任务）、NXTG-Forge Orchestrator（file locking 加 knowledge capture 加 drift detection，Rust 实现单二进制）、gastown（可扩展到 20-30 个 agent，git-backed issue tracking，Bors 式合并队列）、agent-kanban（leader-worker 任务板加密码学身份绑定）、以及明确写着支持 "Claude Code、Codex、Pi、OpenCode" 的 Ouijit（基于 git worktree 的任务管理器）。第四类是"Autonomous Loop Runner"，也就是单一目标驱动、不断重试直到验证通过的循环模式（Ralph Wiggum 流派一大堆同类实现）。第五类是"Autonomous Task Runner"，由外部工单系统（Linear、GitHub Issue、定时任务）驱动、无人值守运行并把状态同步回去，代表有 OpenHands、cyrus、sortie 等。第六类是"Agent Infrastructure & Primitives"，也就是控制面、协调协议、harness 适配器这类更底层的基础设施，其中 Archon（"首个开源 harness builder"，主打让 AI coding 确定性、可重复，带 approval gates 和隔离 git worktree）和 LionClaw（"durable, auditable workers"）的措辞几乎就是在描述 Herdr 想做的事。第七类是常驻的"Personal Assistant"，跨 session 记忆、按自己的日程运行，工作范围不限于写代码。

调研过程中还发现了另一份邻近的清单 `ai-boost/awesome-harness-engineering`，专门聚焦"harness engineering"这个新分类，覆盖工具、模式、评测、记忆、MCP、权限、可观测性和编排，标题里就写着"documents the shift from single-agent to orchestrated multi-agent teams"，可以作为 awesome-agent-orchestrators 的姊妹清单继续深挖。

此外还有一条完全不同的轴：从零构造单个 agent 内部推理/工具循环的框架，比如 LangGraph、CrewAI、AutoGen/AG2、OpenAI Agents SDK、Google ADK。这些跟 Herdr 关系不大——Herdr 编排的是已经成型的 CLI coding agent（Claude Code、Codex、Pi），而不是从零搭一个新 agent 的推理循环——但了解这条轴的存在有助于避免把这些框架和上面的编排工具做不恰当的类比。

## 7. 行业趋势判断：从单 agent 长跑推理到多 agent 协作，且关键在"跨 harness"

查证结果支持"行业正从单 agent 长时间自主推理转向多 agent 协作"这个判断：Anthropic 自己的《2026 Agentic Coding Trends Report》把"multi-agent coordination"列为八大趋势之一；有分析文章指出"2025 年关于要不要做多 agent 的争论已经结束，Anthropic、Cognition、OpenAI 都收敛到了 orchestrator + 隔离 subagent 这个模式"。

但这里有一个重要但容易被忽略的区分：大厂讲的"多 agent"，绝大多数是**同一个 harness 内部**的编排——Claude Code 的 subagent 和 Agent Teams 里的 teammate 清一色都是 Claude 实例，这是因为每家厂商都有结构性动机把用户留在自己的模型生态里，"多 agent"对它们而言更像是"怎么把一个大任务拆给多份自家模型实例去跑"。真正**跨 harness、跨厂商异构**的多 agent 协作——也就是 Herdr、hcom、Ouijit、buzz、NXTG-Forge 这些明确支持"Claude Code + Codex + Gemini/Pi/OpenCode"混跑的项目——几乎全部来自开源社区和独立开发者,而不是大厂官方产品，原因很直接：没有一家模型厂商有动机去做好"让自己的 agent 乖乖被别家 agent 调度、甚至被第三方审计"这件事，这一层的标准化工作天然需要厂商无关的第三方来补，而这正是 A2A、ACP 这些开放协议存在的意义，也正是 Herdr 所在的赛道。

## 8. Herdr 项目现状回顾

根据你给出的项目描述，Herdr 是"Herdr Interactive Worker 的本地控制层和权威实现"，目标是把"将任务交给另一个 AI Agent"变成一套可审计、可恢复、模型无关的工程流程。用户只批准任务范围、Runtime、权限、验收标准以及集成/清理等语义边界，系统负责把这些决定编译成确定性的执行步骤，并通过独立 worktree、单写者约束、身份绑定、哈希记录和显式 Gate，保证 Claude、Codex、Pi 等持久 Worker 不会越权修改、静默合并、推送或清理目标项目。当前架构以目标项目中的主 Agent Session 作为 Coordinator，无守护进程/数据库/后台队列的 Python 工具 `herdr-bridge` 是确定性控制平面，负责能力探测、Runtime 锁定、私有 Dispatch 创建、Worker 启动、生命周期控制及恢复；每个 Worker 在独立 worktree 中运行，任务、身份、结果保存在 Git 外的私有、追加式、哈希绑定记录里；结果先写入持久 callback receipt，再由 callback-inbox 交给 Coordinator 串行审查；验收通过后第一道 Gate 负责受限提交、候选验证和 fast-forward 集成，第二道 Gate 才允许安全清理；多 Dispatch 与 Batch 是建立在独立 Child 生命周期之上的编排层；旁路的 Delegation Recommendation Rule 和 Capability Profile Reader 只做建议和参考，不控制生命周期或自动派遣。

## 9. Herdr 解耦的具体建议

你的目标是把 Herdr 的核心控制逻辑和具体平台/App（tmux、Herdr 这个产品名本身）解耦，不想锁死。结合前面调研到的整个协议和产品地图，建议把 herdr-bridge 现在耦合在一起的东西拆成三层，每一层单独决定"用现成标准"还是"自己定义"。

第一层是 Coordinator 与单个 Worker 之间"怎么对话、怎么拿流式输出、怎么申请工具权限"，建议直接采用 **ACP**。查证确认 ACP Registry 里已经有 Claude、Codex、pi 三个官方或社区维护的适配器，正好覆盖 Herdr 点名的三个 Runtime。把 herdr-bridge 里现在"怎么拉起 tmux、怎么解析终端输出、怎么给不同 Runtime 写不同探测脚本"的部分，替换成统一的 ACP 客户端实现，就能彻底摆脱对 tmux 这种终端复用工具的依赖：会话创建、prompt、流式更新、工具调用、权限请求（`session/request_permission`）都是结构化 JSON-RPC 消息而不是抓屏幕文本，未来新增 Runtime 只要对方有 ACP 适配器就能直接接入，不需要重新写一套集成代码。这一层几乎不需要自研，是投入产出比最高的一步。

第二层是"Coordinator 要同时调度多个持久 Worker、跟踪任务状态、拿到结果回调"，也就是现在的 Dispatch/Batch/callback receipt。这里建议借鉴（而不是直接接入）**A2A** 的概念模型：Task 的有限状态机（可以对应 Herdr 里 Dispatch 从创建、Worker 运行、结果写入 receipt、Coordinator 审查、到返修/暂停/换任/丢弃的完整生命周期）、Artifact（对应 Worker 产出的具体文件/diff/结果）、Push Notification（对应现在 callback-inbox 的回调机制，即便本地实现不用走 HTTP webhook，也可以在架构文档里用同一套语义描述）、以及 Context/contextId（对应把同一批相关 Dispatch 分组到一个 Batch 下）。这样做的好处是：即使现在只是单机多进程的本地实现，只要状态机和词汇跟 A2A 的公开语义兼容，以后如果要支持跨机器分发 Worker、或者对接别人的 A2A 生态，迁移成本会低很多，不需要推倒重来。不建议现在就直接引入完整的 A2A 协议栈（它设计成走 HTTP + 面向跨厂商远程场景，对单机场景偏重），概念先行、实现从简。

第三层，也是 Herdr 真正的核心价值所在，是独立 worktree 隔离、单写者约束、身份绑定、Git 外的哈希追加式审计记录，以及两道 Gate（受限提交/候选验证/fast-forward 集成，再到安全清理）。这一层没有任何现成协议覆盖——ACP 不管，A2A 不管，AHP 明确把它列为 anti-goal，Claude Code 原生的 Agent Teams 也只有一个简化的 hook 拦截机制，远没有达到哈希绑定审计和两道正式 Gate 的强度。这部分建议完全自研，但要用"写协议规范"而不是"写产品文档"的方式来写，参考 AHP 的写法——doctrine（原则）、anti-goals（明确不管什么）、分层交互图、以及可选的 JSON Schema——把它整理成一份独立于"Herdr"这个产品名的规范文档，可以取名类似"Dispatch Gate Protocol"或"Worker Delegation Gate Spec"。文档里应当独立定义身份绑定方案（Worker 的身份如何生成、如何与 Dispatch 绑定、如何防止伪造）、哈希追加日志的具体格式（每条记录哈希谁、和上一条怎么链接、如何做完整性校验）、单写者 worktree 锁的协议（如何声明、如何检测冲突、锁的生命周期）、以及两道 Gate 各自的输入输出契约（Gate 1 接受什么状态的 Dispatch、做哪些校验、Gate 2 的触发条件和清理范围）。这份规范应当完全不出现 tmux、Herdr 品牌名这类具体实现细节，只描述抽象的状态机和数据格式，这样无论底层用 tmux 还是别的终端方案、上层叫 Herdr 还是别的名字，这份 Gate/审计规范都可以被复用。

在这三层之上，如果未来有需求让多个 UI（比如你自己的终端界面加一个网页监控面板）同时观察同一个 Coordinator session 的状态，可以再加一个可选的第四层，借鉴 AHP 的 channel/action/reducer 模型来做状态同步，但这不是当前解耦的重点，可以往后放。

具体到代码结构上的落地路径，建议把 herdr-bridge 拆成三个独立可测试的模块：一个 `WorkerRuntimeAdapter` 接口，内部用 ACP 客户端实现，对上层暴露"创建会话/发送 prompt/接收流式更新/响应权限请求"这几个与 Runtime 无关的方法；一个 `DispatchEngine`，管理 Task 状态机、Artifact 收集、Batch 分组，接口设计上参考 A2A 的词汇但实现保持本地化；一个 `GateAuditCore`，完全独立于前两者，只依赖抽象的"Dispatch 对象"和"文件系统/git 操作"，负责身份绑定、哈希链、worktree 锁、两道 Gate 的校验逻辑，这个模块应该是唯一一个理论上可以被抽出去单独开源、单独发布规范文档的部分。

## 10. 值得继续深挖的对标项目

已经在浏览器里打开了五个和 Herdr 定位最接近的开源项目页面，建议逐个看它们的 README 和源码，重点关注每个项目的 Gate/审计/身份实现方式：Fusion（github.com/Runfusion/Fusion，plan-review-execute gates 加 per-task worktree）、NXTG-Forge Orchestrator（github.com/nxtg-ai/forge-orchestrator，file locking 加 drift detection，单 Rust 二进制）、Ouijit（github.com/ouijit/ouijit，明确支持 Claude Code/Codex/Pi/OpenCode 四种 Runtime，是目前调研范围内 Runtime 覆盖面和 Herdr 最接近的项目）、gastown（github.com/gastownhall/gastown，Bors 式合并队列，规模做到 20-30 个 agent）、以及 Archon（github.com/coleam00/Archon，标榜"首个开源 harness builder"，approval gates 加隔离 worktree）。另外两份策划清单 `andyrewlee/awesome-agent-orchestrators` 和 `ai-boost/awesome-harness-engineering` 值得定期回顾，因为这个生态目前更新速度很快，新项目和新模式还在持续出现。

## 11. 需要留意的风险和不确性

这里列出的协议里，ACP、A2A、AHP 都明确标注自己还在"active development"或"draft"阶段，wire 格式和 API 都可能发生破坏性变更，直接生产依赖需要做好版本锁定和适配层隔离。ACP 对远程 agent 场景的支持官方原话是"work in progress"，如果 Herdr 未来要支持远程/云端 Worker，这块需要持续跟进。A2A 设计上偏重跨网络、跨厂商的远程调用场景，直接套用可能带来不必要的 HTTP/鉴权复杂度，建议只借用概念模型、不引入完整协议栈。Claude Code 的 Agent Teams、cross-session messaging 都还是实验特性，行为随版本频繁变化（changelog 显示几乎每个版本都有相关修复），如果 Herdr 未来考虑与 Claude Code 原生多 agent 能力做集成或对比，需要持续关注其变更日志。
