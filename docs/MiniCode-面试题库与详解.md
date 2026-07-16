# MiniCode 面试题库与详解

> 与 [`MiniCode-项目全景学习手册.md`](MiniCode-项目全景学习手册.md) 配套使用。回答基于根包 `minicode/`；“优化方向”是设计建议，面试时不要冒充为当前功能。

## 使用方法

- 每题先背 **一句话结论**，再练习 1—2 分钟口述。
- 按“动机 → 实现 → 取舍 → 证据 → 优化”展开追问。
- 标有火焰重点标记的共 15 题，是最高优先级。

---

## 一、项目总览与个人贡献

### Q01 🔥 **重点题**：请你介绍一下 MiniCode 项目解决什么问题？和 Claude Code、Cursor 这类工具的区别是什么？

**一句话结论：** MiniCode 是本地终端 Coding Agent；差异不在“能调模型”，而在把恢复、验证、权限和运行时状态放进执行闭环。

**可直接口述：** MiniCode 面向真实本地仓库的持续开发。用户输入任务后，模型决定读什么、改哪里、跑什么命令，工具执行后再把结果反馈模型。项目重点不是单次聊天，而是长任务可靠性：中断能否恢复、错误修改能否回退、模型说完成有没有证据、工具能否越权、上下文过长时会不会遗忘目标。成熟产品的生态和体验更完整；MiniCode 的定位是轻量、本地优先、强调可观察/可恢复/可验证的 Agent runtime，而不是声称替代它们。

**代码锚点：** `agent_loop.py`、`turn_kernel.py`、`session.py`、`permissions.py`、`verification_controller.py`。

**追问与优化：** 这些能力不是 UI 按钮：session 真正落盘，checkpoint/rewind 作用于文件，verification 影响停止决策。下一步需用任务成功率、回退成功率、验证覆盖率证明收益，而不是只列功能。

### Q02：为什么同时有 Python 和 TypeScript 两套实现？当前主实现为什么是 Python？

**回答：** `ts-src/` 是平行 TypeScript 实现；根目录 `pyproject.toml` 的命令入口指向 `minicode/`，因此当前主运行时是 Python。双实现验证相近 Agent 架构在不同生态里的可移植性：TS 更贴近 Node 工具链，Python 更适合快速做 AI runtime、实验与测试。不要说两端功能完全等价；面试讲当前项目时以 `minicode/` 和 `tests/` 为准。

**优化：** 建立能力矩阵和共享协议 fixture，防止双端文档相同、行为不同。

### Q03：你在项目里最核心的负责模块是什么？如何证明能力真正跑在主链路上？

**回答模板：** 用你的真实负责模块替换名称。不要只说“做了 memory”，而要说“把它接入 `run_agent_turn()`，并用集成测试覆盖”。独立单测只能证明函数存在；主链路证据应显示任务开始时检索/注入，任务后回写/反馈，且失败不会破坏 turn。verification 同理：它必须影响 stop reason，而不是返回一个没人消费的 plan。

**优化：** 为每项能力加端到端冒烟测试和 telemetry，测试防回归，埋点证明线上真被使用。

### Q04 🔥 **重点题**：请画出一次用户输入从 TUI 到模型、工具调用、最终回复的完整链路。

```text
用户输入（TUI / headless / gateway）
→ ChatMessage 与运行时配置
→ agent_loop.run_agent_turn()
→ TaskObject + LayeredContext + 按需记忆注入
→ 模型 adapter
→ assistant 文本或结构化 tool call
→ ToolRegistry 查找工具，PermissionGate 检查
→ 执行工具，结果/错误写回消息、session、runtime event
→ turn_kernel 判断阶段、预算、进度与验证证据
→ 必要时 verification、压缩、模型切换、checkpoint
→ 最终答复、stop reason、可回放 session
```

**回答：** 模型通常先探索，再执行，工具结果要重新给模型判断。循环停止结合最大步数、用户等待、阻塞、验证失败、widening 等结构化原因，而不是只看模型说“完成”。编辑任务会被导向验证；session 保存 transcript、运行时摘要与 checkpoint，支持恢复。

**代码锚点：** `tty_app.py`/`headless.py`/`gateway.py` → `agent_loop.py:run_agent_turn` → `turn_kernel.py:decide_assistant_turn`、`decide_tool_turn`。

**优化：** 所有入口消费统一的带 correlation id 的事件流，防止 TUI 和 HTTP 对同一次执行有不同解释。

### Q05：项目依赖很少，为什么能实现模型调用、终端工具、会话？优劣是什么？

**回答：** 标准库已经覆盖 HTTP、子进程、JSON、线程、文件系统；项目通过 adapter、tools、session 自己封装。优点是安装轻、依赖冲突少、机制透明；缺点是要自己处理 HTTP 重试、协议兼容、schema 校验、跨平台终端等大量边界。

**优化：** 不应迷信零依赖。安全和协议复杂区域可引入小而稳定的依赖，并锁版本、做供应链审计。

### Q06：如何定义项目“可用”而不是“功能堆得多”？

**回答：** 至少四层：功能能执行、失败有诊断、风险可控、结果能验证。对 MiniCode，不是能读写文件就算可用，还要能说明 provider 失败原因、编辑前留 checkpoint、完成时给测试证据、可恢复 session。测试数量只是信号。

**优化：** 建立 release readiness：固定任务集、mock 回归、真实 provider smoke、跨平台检查，并用成功率、P95 时延、成本、误编辑率定义门槛。

---

## 二、Agent Loop 与运行时控制

### Q07 🔥 **重点题**：`agent_loop.py` 的核心职责是什么？如何处理模型多轮调用工具的循环？

**一句话结论：** 它是运行时编排层，可靠地组织“模型输出—工具执行—结果回写—再决策”。

**回答：** `run_agent_turn()` 维护消息、调用模型 adapter、解析输出、执行工具、捕获错误、记录 runtime event、更新 session，并把工具结果再次交给模型。停止结合最大步数、await user、blocked、verification failed、widen needed 等原因。代码还区分空响应、可恢复 thinking stop、provider availability 与 fallback，避免把各种异常粗暴 retry。

**追问：** 为什么不全写一个 `while`？因为阶段策略、验证、恢复会与执行耦合，难测且不可解释；所以决策放入 turn kernel，运行时控制放入 orchestrator。

**优化：** 当前文件职责集中。可抽取 `ModelStepExecutor`、`ToolStepExecutor`、`RecoveryPolicy`、`TurnPersistence`，用不可变 step result 传递状态。

### Q08 🔥 **重点题**：为什么单独设计 `turn_kernel.py`？

**回答：** Loop 负责执行，kernel 负责策略。`TurnRecurrentState` 汇集步数、工具成功失败、进度、验证与 widening 状态；`derive_turn_step_policy()` 推导阶段，两个 decision 函数把 assistant/tool 事件变成结构化决策。分离后停止原因可解释，验证不只是 Prompt，策略也可以不请求模型就单测。

**取舍：** 状态对象较多，必须有清晰不变量，否则会过度设计。

**优化：** 增加状态转移表和 property-based test，例如严格验证未有证据时不可直接 `done`；将决策理由写入 runtime event。

### Q09：`explore → execute → verify` 如何切换，各约束什么？

**回答：** explore 消除未知，常用 list/grep/read；execute 在证据充分后最小改动；verify 运行测试、编译或 diff 检查。验证失败可回 execute，理解错误可有限回 explore，但必须记录原因避免无穷搜索。`derive_turn_step_policy()` 综合 recurrent state、预算、进度和验证状态推导策略。

**优化：** 按任务类型设模板：解释任务可跳执行；高风险迁移强制计划、diff review 与全量测试。

### Q10：什么是 widening？为什么不能无限增加最大步数？

**回答：** widening 是已有进展证据但默认预算不足时，受控扩大执行空间。无限增加步数会失控花费、陷入重复并让用户不知道为何迟迟不结束。`activate_widening()` 会记录额外步数、原因和证据。

**优化：** 只在最近 N 步确有信息增益时允许，设硬上限；连续相同 tool signature 时即使预算未耗尽也熔断。

### Q11：空响应、思考中断、工具报错、provider 不可用如何区分恢复？

**回答：** 不能一律 retry。空内容可有限重试；thinking stop 要看 stop reason 与是否已有工具结果；工具错误应以可读结果回传模型换策略；provider 错误先分鉴权、限流、网络、模型下线，必要时有限 fallback。`agent_loop.py` 中存在对应判断函数。

**优化：** 建立错误 taxonomy 与 retry budget：网络超时指数退避，401 不重试，422 检查请求构造，权限拒绝要求用户确认。

### Q12 🔥 **重点题**：`CyberneticOrchestrator` 控制哪些信号和动作？与普通 workflow 的区别？

**回答：** 固定 workflow 是预先写好“检索→生成→保存”；orchestrator 关注运行中的反馈：上下文压力、token/call 成本、工具与模型错误、进度、模型可用性、记忆需求。`step_start()`/`step_end()` 是信号入口；它还能注入/反思记忆、路由模型、错误后切换建议和成本控制。

本质是“观测 → 动作 → 再观测”。例如上下文压力高就压缩，而非超过窗口才失败；工具持续失败就调整恢复，而不是一直重复。**要诚实：** 有控制面不等于已证明自适应最优，仍需开关控制器的消融实验比较成功率、成本、恢复率。

**优化：** 统一 `ControlSignal`/`ControlAction` 事件模型，为每个动作定义目标和效果指标，避免多个 controller 相互冲突。

### Q13：Agent 重复调用同一工具没有进展，如何检测与纠偏？

**回答：** 记录规范化 tool signature 和输出摘要，在滑动窗口检查重复且无新增文件/测试信息；同时看任务图、错误类型、进度是否变化。满足阈值后提示换策略、反思、停止并说明阻塞或请求用户输入。

**优化：** 不要死板地“三次相同即停”，polling 任务可能合理重复；应按任务类型和结果变化判断。

### Q14：如何控制 token、调用次数和成本？只靠 Prompt 吗？

**回答：** Prompt 中说“节省 token”没有硬约束。项目有 `cost_tracker.py`、`cost_control.py`，orchestrator 的 `run_cost_control()` 把 token/call 当信号。可用步骤预算、模型路由、上下文压缩、减少记忆注入、失败熔断和限制 widening 控制。

**优化：** 使用任务级预算；耗尽时返回已完成内容、证据和可继续的下一步，而不是静默失败。

---

## 三、模型接入、路由与可靠性

### Q15：如何抽象不同模型提供商？adapter 共性边界是什么？

**回答：** adapter 把内部统一的消息、系统提示、工具定义、工具结果和配置翻译成 provider 请求，再标准化响应为内部 assistant 文本/tool call。共性边界是“Agent 需要的模型能力”，而不是各家字段。差异仍要承认：工具格式、streaming、停止原因、token 统计和错误语义可能不同。

**优化：** 用契约测试验证同一内部请求在 mock provider 下产生等价语义；显式暴露 capability，不要暗中降级。

### Q16 🔥 **重点题**：`ModelSwitcher` 如何做 fallback？为何不能盲目切所有候选？

**回答：** `ModelSwitcher` 维护当前模型/adapter、切换次数、失败记录与历史。`switch_to_fallback()` 按候选和可尝试性选择，`_can_attempt_model()` 避免反复尝试连续失败模型，`_should_limit_cross_provider_fallbacks()` 限制跨 provider 切换。失败可能是 API Key、请求格式、网络或权限，盲切十个模型只会扩大成本；跨 provider 还可能不兼容工具协议和上下文。

正确顺序是先分类、对暂态错误有限退避、对不可用模型选能力兼容 fallback、记录 switch reason。认证错误直接报告，不要切模型。

**优化：** 结构化声明模型能力（tool calling、JSON mode、窗口、streaming），fallback 按任务需求过滤，不只按名称列表。

### Q17：模型切换如何避免上下文格式与工具协议不一致？

**回答：** 保存 provider 无关的内部消息与工具抽象，切换后由目标 adapter 重建请求；先检查候选能力。目标不支持并行工具调用可降级串行，不支持 JSON schema 则不宜承担高风险编辑任务。

**优化：** 为复杂 schema 做兼容性检查，并将转换失败归类为 capability mismatch，而不是普通模型失败。

### Q18：如何区分本地 Agent 问题和 provider 问题？

**回答：** 看故障层与证据。离开本地前的 schema 校验、权限拒绝是本地；401/403 通常是凭据、429 限流、5xx/连接超时多为 provider/网络、响应格式不符可能是兼容问题。readiness 与 provider diagnostics 应展示分类。

**优化：** 记录 provider、model、request id、状态码、retry count，敏感信息脱敏；用 `/readiness` 预检。

### Q19：模型说“完成”但没改文件或没测，如何拦截？

**回答：** 对修改任务不能相信自然语言。`TurnVerificationState` 可要求证据，`VerificationController` 形成计划，kernel 在 evidence 未就绪时阻止结束或要求补验证；工具事件也能证明是否执行 edit/test。

**优化：** 将完成变成结构化 checklist：目标文件、diff、命令、退出码、结果摘要，并在最终答复中引用。

---

## 四、工具系统、权限与安全

### Q20 🔥 **重点题**：`ToolDefinition` / `ToolRegistry` 如何让模型知道工具和参数？

**回答：** 每项工具统一描述 name、description、input schema、validator、run。注册表汇总后将工具说明放入模型请求；模型据 description 决定何时使用，据 JSON Schema 组织参数。工具调用返回后由注册表定位实现。MCP descriptor 也被转为同一 `ToolDefinition`，所以主循环统一处理内置与外部工具。

模型只提出调用意图；validator 和权限才是真正执行边界。工具 description 是 Agent API 设计，应该写清能力、限制、输入示例和风险。

**优化：** 分析调用日志的混淆率和参数失败率，迭代 description/schema；危险工具增加明确确认字段。

### Q21：参数如何校验？为何 JSON 不够安全？

**回答：** JSON 可解析不代表语义正确。仍要校验字段、类型、范围、路径、命令；副作用前过权限。模型可能幻觉字段、被注入诱导或生成工作区外路径。

**优化：** 高风险工具用严格 JSON Schema 和专门的路径/命令类型，校验失败回传可行动错误，避免透传 validator。

### Q22 🔥 **重点题**：`permissions.py` 如何管理文件、命令和编辑权限？

**回答：** `PermissionManager` 以 workspace root 为基准，路径规范化后判断是否在授权目录内。`ensure_path_access()` 管文件访问，`ensure_command()` 管命令签名和危险分类，`ensure_edit()` 可结合 diff preview。权限既可在当前 turn 临时生效，也可保存；`PermissionGate` 向工具层提供窄接口。

设计关键是分层：读取、写入、编辑差异、运行命令风险不同，不能一个总开关解决。拒绝也要是清晰的权限结果，让模型不会误判成工具故障。

**优化：** 解析真实路径防 symlink 绕过；能力令牌替代长期全局授权；命令进沙箱；网络单列；审计记录关联任务、工具、时间和 diff hash。

### Q23：危险命令如何识别？黑名单如何被绕过？

**回答：** `_classify_dangerous_command()` 识别典型命令，`ensure_command()` 决定确认/拒绝。黑名单能挡常见误操作，却挡不住 shell 拼接、解释器脚本、别名、编码、间接下载和子进程。

**优化：** allowlist + 沙箱为主，黑名单为辅；解析 shell AST，或只暴露受控 command runner。

### Q24：diff preview、checkpoint、rewind 各解决什么？

**回答：** diff preview 是编辑前知情/授权；checkpoint 是留恢复锚点；rewind 是事后真正恢复。分别面向预防、留证、补救。Git 不能替代，因为 Agent 可在未提交且不干净的工作树上连续试探修改。

**优化：** checkpoint 附内容 hash、路径、时间和 turn；rewind 默认 preview，检测用户外部改动防止覆盖。

### Q25：Prompt Injection 诱导读取工作区外敏感文件，如何防御？

**回答：** 文件文本不可信，但工具调用仍过路径权限；工作区外路径应被拒绝/确认，不能直接泄漏。缺口是工作区内 `.env`、symlink、工具间接访问和 MCP server 自身权限。

**优化：** 敏感文件策略、内容脱敏、最小读取范围、MCP 隔离与读取审计；上下文中区分用户指令和文件内容。

---

## 五、Session、Checkpoint 与 Rewind

### Q26 🔥 **重点题**：Session 为什么不只保存聊天记录？`SessionData` 还应保存什么？

**回答：** 聊天记录只说明说过什么，不能说明系统做过什么。开发 Agent 的状态还包括工具调用和结果、workspace、任务摘要、阶段/stop reason、checkpoint、runtime trace、指令层、hook/extension/readiness 摘要等。`SessionData`、`SessionMetadata`、`FileCheckpoint` 将这些落盘，使 inspect、replay、resume 有依据。

例如用户问“为什么上次没完成”，有 transcript、工具错误、provider 诊断和 verification state 才能定位是权限、401 还是测试失败。

**优化：** 版本化 session schema；大工具输出分层存储；敏感数据支持加密/红删；恢复前重新检查 workspace 和 provider 状态。

### Q27：如何实现增量保存与定期合并？性能和恢复如何取舍？

**回答：** `save_session()` 支持 full save 或 delta；`_save_delta()` 写增量，`_consolidate_deltas()` 合并；`AutosaveManager` 依靠 dirty 标记和时间间隔保存。好处是不必每个工具事件重写完整 session。代价是恢复要正确回放基础快照和所有 delta，并处理部分写入、损坏、顺序和版本兼容。

**优化：** 临时文件原子写再 rename；delta 附序号/hash；超过阈值后台合并；恢复时识别损坏尾部。

### Q28：Checkpoint 和 Git 的 commit/checkout 区别？为何仍需要 checkpoint？

**回答：** Git 面向版本控制、协作和长期历史；checkpoint 面向 Agent turn 内短生命周期的自动恢复。用户不希望每次试探修改都 commit，也不能假设工作树干净。checkpoint 精确服务“回退本次 Agent 的改动”，Git 保留项目演进。

**优化：** 记录 checkpoint 与 Git HEAD/index 的关系；冲突时提示选择，不能自动覆盖用户改动。

### Q29：`rewind-preview` 为什么重要？如何防误恢复？

**回答：** rewind 是高副作用操作。preview 先显示 checkpoint、涉及文件和差异，让用户确认范围。执行时应校验路径和 hash；发现 checkpoint 后用户手改必须提示冲突。

**优化：** 支持单文件恢复、生成 patch、审计 rewind，并在恢复前创建一个“恢复前 checkpoint”。

### Q30：进程崩溃或关闭终端后，如何恢复可继续工作状态？

**回答：** session 经 autosave 和 delta/full save 持久化；启动后 list/load latest session 可恢复 transcript、metadata、checkpoint，利用 `/session`、`/session-replay` 查看。恢复不是盲目重放：要重查 provider、工作区变化、子进程状态与命令幂等性。

**优化：** 给工具调用标记幂等性。读取可重试，写文件/发布/删除应保持待确认，不能自动重放。

---

## 六、记忆、上下文与 RAG

### Q31 🔥 **重点题**：working memory、长期 memory、session memory 是什么？为何不能只用向量库？

**回答：** working memory 是当前任务绝不能丢的目标、约束和最近验证证据；session memory 是一段工作过程的可回放历史；长期 memory 是跨 session 的架构约定、历史决策与常见故障。三者生命周期、可信度和访问方式不同。

向量库只擅长相似检索，不天然表达“当前任务硬约束”“已执行证据”“这条知识已过期”。全部混进向量库会丢顺序、召回陈旧信息、增加上下文噪声。

**代码锚点：** `memory.py`、`working_memory.py`、`memory_pipeline.py`、`context_compactor.py`、`layered_context.py`。

**优化：** 统一元数据、不同存储策略：工作记忆固定优先级，长期记忆按相关性+时效，session 用事件序列和摘要恢复。

### Q32 🔥 **重点题**：`memory_pipeline.py` 的 read、inject、write、feedback、maintain 构成什么闭环？

**回答：** `read()` 针对任务检索，必要时 query reformulation；`inject()` 按预算把候选放入分层上下文；完成后 `write()` 写长期价值事实；`feedback()` 根据使用效果更新评价；`maintain()` 做冷却、清理与维护。它不是一次 RAG，而是“使用后改善以后召回”的闭环。

写太勤会固化噪声，读太多会污染上下文，维护太猛会删关键知识。因此代码还包含统计、活跃 domain、自适应 cooldown、spread activation。

**优化：** 写入先是候选，只有多次验证或用户确认后晋升；feedback 结合后续成功和用户纠正，不只依赖模型自评。

### Q33：上下文接近上限，为何不能直接截断最早消息？如何与 memory 配合？

**回答：** 最早消息可能有硬目标、权限约束、已验证结论；直接截断会让模型重复问或产生冲突。应保护稳定任务和关键证据，对冗长历史摘要，持久记忆保存可复用事实；文件内容按需重新读而不是永久塞上下文。

**优化：** 摘要带来源和不确定性标记，压缩前后做一致性检查。

### Q34：如何避免检索到不相关、过期或错误记忆？

**回答：** 不只看 embedding 相似度，还要看任务类型、domain、路径、更新时间、来源可信度、历史使用效果和 token 预算。记忆应标为历史线索，模型采用前应读取当前文件核验。

**优化：** 时间衰减、commit/hash 绑定、负反馈和 source validation；引用文件不存在或 hash 不一致即降权。

### Q35：继续升级记忆，优先混合检索、重排序、过期还是用户确认？

**回答：** 取决于失败数据。没有指标时我优先来源与过期机制，因为陈旧错误记忆比漏召回更危险；再做混合检索/rerank；涉及偏好、敏感知识和长期决策时加用户确认。先可信度，后召回率。

**优化：** 建离线标注集，测 recall@k、nDCG、过期采纳率和记忆导致的返工率。

---

## 七、MCP 与可扩展性

### Q36 🔥 **重点题**：MCP 如何完成工具发现、描述转换和调用？

**回答：** `StdioMcpClient` 使用 JSON-RPC 调 `tools/list`、`resources/list`、`prompts/list`。`create_mcp_backed_tools()` 将 descriptor 的 name、description、`inputSchema` 转为内部 `ToolDefinition`，命名 `mcp__server__tool`。模型在统一工具表中看到它；调用时包装函数通过 `call_tool()` 发 `tools/call`，结果格式化为 `ToolResult`。

资源与 prompts 也建立索引，以内部工具形式暴露 list/read/get。MCP 并非模型直连外部程序，而是 MiniCode 将外部协议映射到自身工具抽象。

**优化：** 缓存 descriptor 与版本，schema 更新做兼容检查；为 MCP 工具声明风险等级并走统一权限门。

### Q37：MCP 客户端如何同时处理 JSON-RPC、Content-Length 与换行 JSON？

**回答：** 请求生成递增 id，并将 response queue 放入 `_pending`；发送侧写 newline JSON 或 `Content-Length` 帧；stdout 线程探测协议并解析响应，按 id 投递 queue。request 超时后清 pending 并附 stderr 报错。

**优化：** 配置优先于自动猜测；增加 framing fuzz test、header 上限及 notification（无 id）处理。

### Q38 🔥 **重点题**：为什么 MCP Server 使用懒启动？改善什么，又有哪些首次调用风险？

**回答：** 若启动 Agent 时拉起所有 MCP server，会拖慢冷启动、浪费资源，某个坏 server 还可能影响整体。懒启动在首次真正需要时 `_ensure_started()`，未使用就不消耗资源，也隔离失败。风险是首次调用延迟高、错误晚暴露，可能打断模型计划。代码用超时、stderr、错误状态、pending 请求唤醒和缓存降低风险。

**优化：** 提供 readiness 检查与可选预热；失败 server 熔断/退避；向工具表暴露 pending/error 状态，避免模型不停尝试。

### Q39：子进程退出、请求超时、超大 payload 如何处理？

**回答：** stdout 线程发现退出时给所有 pending queue 写错误并清空，防止阻塞；request 超时清 pending；代码检查 `MAX_MCP_PAYLOAD_BYTES`，超出即拒绝；`close()` 清 pending、跨平台终止进程并重置缓存。

**优化：** 还需限制并发、输出速率、总内存；若协议支持可取消 server 端任务；重启次数设上限防 crash loop。

### Q40：危险 MCP 工具如何与本地权限协同？

**回答：** MCP 转发不是绕过权限的理由。调用包装工具前要根据 descriptor 和 arguments 过路径/命令检查；server 端也再次校验。最安全是双层最小权限：Agent 获得调用 token，server 在容器/低权限账户运行。

**优化：** descriptor 增加 riskLevel、能力和数据范围；日志关联 Agent session id 与 MCP request id。

---

## 八、质量、测试与工程化

### Q41：如何划分单元、集成、压力与端到端测试？

**回答：** 单元测 permissions、turn policy、session 序列化等纯逻辑；集成测 agent loop 与 mock model/tool/memory 的组合；压力测长上下文、重复工具、并发与大记忆；端到端从入口运行受控/真实 provider，检查 session、输出和副作用。`tests/` 有 agent flow、MCP、memory stress、cybernetics e2e 等分层痕迹。

**优化：** PR 跑快速 deterministic suite；定时跑压力和真实 provider smoke；失败保留 session artifact。

### Q42 🔥 **重点题**：如何为不确定的大模型 Agent 写稳定测试？哪些 mock，哪些真实验证？

**回答：** 控制逻辑主要用 mock：固定模型输出/tool call 序列，断言状态转移、参数、权限、session、fallback、verification。这不受网络和随机性影响。真实 provider 测试只验证 adapter 兼容、凭据、模型可用性、真实工具格式和端到端 smoke，应数量少、隔离密钥、记录版本与 artifact，不能当所有 PR 的唯一门槛。

更高层用任务结果断言，例如文件含目标函数、pytest 通过、没有越界写入，不断言模型逐字回复。

**优化：** provider 支持时固定 temperature/seed；版本化 eval dataset；保存失败轨迹；不同措辞做 metamorphic test。

### Q43：为何要有 `verification_controller.py`？如何推断测试命令？

**回答：** 让模型自由决定验证常会遗漏或跑无关命令。controller 接收 `VerificationSignal` 生成 `VerificationPlan`，通过风险分数得到等级，再按改动文件和模式选命令；Python 文件会按路径推断测试目标，做去重和测试文件识别。这是可见、可测的启发式，不是万能证明。

**优化：** 读取 `pyproject.toml`、package.json、CI 配置和历史测试关联，而不只靠文件名；用实际结果回写风险模型。

### Q44：大量测试通过不能说明什么？还需哪些指标？

**回答：** 不能说明覆盖率、断言质量、真实 provider 成功率、安全、跨平台稳定性和用户任务成功率。重复/脆弱测试可能只覆盖 happy path。

**优化：** 补代码覆盖/mutation score、固定任务集成功率、验证后通过率、成本/延迟、工具错误分布、恢复成功率、权限拒绝处理率、session 损坏恢复率、MCP 健康度。

### Q45：Gateway、Headless、TUI、Cron 如何复用核心能力，避免分叉？

**回答：** 入口只处理协议/交互差异：TUI 渲染，headless 解析参数，gateway 处理 HTTP，cron 负责调度；它们共享 `agent_loop`、config、session、permissions、tool registry、runtime policy。统一事件模型可避免某入口绕过验证或 session。

**优化：** 定义 `AgentRuntime.execute(request)` 返回标准事件流，入口只做 adapter；契约测试保证同一请求在不同入口有等价 stop reason 和副作用。

### Q46 🔥 **重点题**：挑一个当前设计不足，说明问题、影响、重构和迁移风险。

**回答：** 我选 `agent_loop.py` 职责集中。它同时关心模型、工具、恢复、上下文、session、事件、fallback 与产品状态。影响是新增能力会修改许多分支、回归面大，也难定位某 stop reason 来自哪层。

不会一刀切重写。我会先用现有测试和 trace-based 集成测试冻结行为；再抽取纯决策的 `TurnPolicy`、无副作用的 `ModelStepExecutor`、`ToolStepExecutor`、`RecoveryCoordinator`、`SessionEventSink`。主循环只保留依赖装配和顺序编排。每次抽取都用同一 mock transcript 对比重构前后事件序列。

迁移风险是错误顺序变化、session 事件漏记、fallback 时上下文不一致。所以要 feature flag、shadow trace 对比、分阶段发布。大文件也有集中编排、快速迭代的优点；重构目标是降低耦合和提升可测性，不是为了目录整洁。

---

## 重点题速记卡

| 题号 | 10 秒结论 |
| --- | --- |
| Q01 | 本地 Coding Agent，强项是可恢复、可验证、可观察。 |
| Q04 | 输入进 loop，模型-工具闭环，kernel 控阶段/验证，session 可恢复。 |
| Q07 | Loop 编排模型、工具、错误、事件、持久化，不是普通 while。 |
| Q08 | Kernel 分离策略，让阶段、预算、停止和验证可解释可测。 |
| Q12 | 控制论是信号—动作—再观测，效果需指标证明。 |
| Q16 | fallback 先分类失败再有限切换，不能盲换模型。 |
| Q20 | 工具定义是模型的受控 API 文档，schema 不是安全边界。 |
| Q22 | 权限按路径、命令、编辑分层，最小权限与审计更可靠。 |
| Q26 | Session 是工作状态，不是聊天记录，才能 replay/resume。 |
| Q31 | 三种记忆生命周期不同，不能混成一个向量库。 |
| Q32 | 记忆是 read→inject→write→feedback→maintain 的闭环。 |
| Q36 | MCP descriptor 被映射为内部工具，由客户端转发。 |
| Q38 | 懒启动省资源但要治理首次调用时延和熔断。 |
| Q42 | 控制逻辑 mock 测，provider 少量真实 smoke 测。 |
| Q46 | 主循环解耦应测试冻结、逐步抽取、trace 对比。 |

## 最后提醒

把答案讲成自己的项目，关键不是罗列“用了 AI 技术”，而是讲清因果链：**模型和外部工具不确定，所以用状态机控制步骤；文件编辑有副作用，所以有权限、checkpoint 和 rewind；模型会假完成，所以把验证证据纳入停止条件；上下文有限，所以记忆、压缩和任务摘要形成闭环。**

面对未完成的能力，说明当前边界、风险、实现路径和验证指标，比把设想包装为事实更有工程可信度。
