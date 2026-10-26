# Python 自建、DeepSeek Harness 与 LangGraph 能力核查

调查日期：2026-10-26（Asia/Shanghai）。范围：公开官方文档与固定提交源码的静态核查。未安装运行候选、未执行故障注入、未验证安全性或性能；本文不作选型决定，也不包含本地业务资料。

## 结论

1. Python 自建是设计路线，不是已有执行产品。标准库提供并发、进程、数据库和日志积木；任务持久化、恢复、审批、身份、隔离与业务审计仍需要设计实现。[Python asyncio](https://docs.python.org/3.14/library/asyncio-task.html)、[multiprocessing](https://docs.python.org/3.14/library/multiprocessing.html)。
2. DeepSeek Harness 官方项目已核实，具备可复用的工具策略、审批、会话日志与恢复接缝。默认 Web 浏览器连接使用 owner 身份；另有非 owner 资源/角色扩展点，不能据此宣称默认 Web 已具备多人分权。沙箱文档也明确文件效果策略不覆盖网络和进程可见性。[连接认证][H-auth]、[Remote 访问控制][H-access]、[沙箱][H-sandbox]。
3. LangGraph 核心库的持久化与人工中断是明确能力，但数据库 checkpointer、恢复的幂等性及外部执行隔离须由应用正确组合。后台运行 API 与自定义资源授权属于 Agent Server 这一部署层，不应算成只安装 MIT 核心库就自动具备。[持久化][L-persist]、[中断源码][L-types]、[后台 API][L-background]、[授权][L-auth]。

## 证据版本

| 候选/证据 | 固定点或日期 | 边界 |
|---|---|---|
| DeepSeek Harness | `22ddee3792402c3e70ec5f86a63c3e3e895a2f60`，提交时间 2026-10-23；根 package.json 为 `0.2.1-rc.3` | [固定源码树][H-root]；以下 H 链接均固定此提交。项目 README 标明 developer preview、可能破坏兼容。 |
| LangGraph 核心 | `a25bfafe4858d2a85bede09ba83f9a591d1f1133`，提交时间 2026-10-25；包版本 `1.2.17` | [包元数据][L-package]、[types.py][L-types]已读取。它们是源码快照，不表示本地装过该版本。 |
| LangChain/LangSmith 文档 | 2026-10-26 访问 | 在线文档会变化；Agent Server、LangChain middleware 的版本未固定、未运行。不得用核心包版本代指整个部署栈版本。 |
| Python | 3.14 文档系列（页面当时显示 3.14.9） | 说明语言积木；未指定或验证实际服务器 Python 版本。 |

## 能力事实与缺口

“已有”仅指文档/源码提供相应机制；“需补建”是基于这些机制边界的工程推断；“未核实”表示本次证据不能支持结论。

| 维度 | Python 自建执行服务 | DeepSeek Harness 固定版本 | LangGraph 核心 / Agent Server |
|---|---|---|---|
| 后台运行与页面断开 | asyncio/进程池能调度任务；HTTP 请求结束与任务生命周期分离、常驻 worker、排队和进程监管均需自建。[P-async][P-process] | Web Host 与 Client 分离；连接层有重连及请求取消语义。仅这些证据不足以保证所有执行入口在关闭页面后继续；必须逐入口实验。[H-auth][H-web] | 核心图执行须由常驻服务承载；Agent Server 文档有 `runs.create` 后查询状态的后台运行 API。[L-background] |
| 服务重启恢复 | 需自行存状态、租约、幂等键及恢复策略；协程和进程池本身不提供业务工作流 checkpoint，此为对标准库职责的推断。[P-async][P-process] | JSONL 事件日志、单写者、flush 屏障与崩溃尾部处理明确存在。resume 会给未闭合回合补 interrupted 结束记录；这不等于自动从外部命令中断的指令处继续，也不承诺外部副作用 exactly-once。[H-persistence] | 有 thread/checkpointer；内存 saver 重启丢失，生产须使用持久存储。固定源码提供 sync/async/exit 三种持久化时机；恢复后的外部副作用仍需幂等设计。[L-persist][L-types] |
| 人工等待与恢复 | 审批状态、等待/超时/取消、恢复入口需自建。 | `ctx.approval`、单次 allowed-once 结果、ask/never 策略以及 asked/decided 日志已存在。未确认一个挂起的交互审批跨 Host 重启后能原样恢复；需单独验证。[H-approval] | `interrupt`+`Command`+checkpointer 明确支持暂停恢复；固定源码明确恢复从节点开头重执行。中断前写操作须考虑重复。[L-types] |
| 身份与权限 | 应用登录、角色、资源所有权、提交与审批身份分离全部需设计。 | 默认 browserPeer 与 `/connect` API Peer 为 owner；Remote access 对非 owner 提供资源角色、策略、过滤与撤权快照扩展，但 owner 绕过其检查。需要独立身份接入、非 owner 映射和业务策略，不能共享默认 owner 登录充当多用户分权。[H-auth][H-access] | 核心 thread_id 是状态键，不是授权凭证。Agent Server 提供 authenticate 和 auth.on 接口，角色与资源规则仍由应用实现；文档指明 self-hosted 无默认认证。[L-persist][L-auth] |
| 工具拦截与强制审批 | 可以封装唯一工具入口；绕过入口的进程能力必须由操作系统控制。Python audit hook 官方明确不是安全沙箱。[P-audit] | tools 有 allow/deny/ask、条件工具集合与执行流水线，approval 消费者默认非 allowed-once 不放行；但任意插件/允许 shell 的能力不能靠提示词限制，仍要外部执行边界。[H-tools][H-approval][H-safety] | 图节点可显式放授权检查；LangChain 另有 wrap_tool_call 可不调用 handler。该 middleware 不是 LangGraph 核心自带的操作系统沙箱，外部 shell/文件/网络权限仍需隔离。[L-middleware] |
| 双任务并发与隔离 | 进程池提供并行机制；两个 worker 不自动隔离同账户可读写的目录、凭据及网络。需资源限额和任务目录策略。[P-process] | session 单写者可防同会话并写，不等于跨任务隔离；沙箱文件策略明确排除网络/进程可见性。双任务能力和资源压力本次未运行验证。[H-persistence][H-sandbox] | 不同 thread 管理不同状态；本次未证明任意并发工具对全局变量、目录或凭据的隔离。核心之外需任务 worker/沙箱、队列容量与取消隔离。[L-persist] |
| 审计 | logging 与 audit 事件可记录过程；业务审批记录及不可篡改留存需设计。[P-audit][P-logging] | 会话 append-only 事件模型及审批 asked/decided 已存在；不是磁盘不可篡改证明，需绑定真正的人类身份、请求内容哈希及独立存储策略。[H-session][H-approval] | checkpoint 可查看历史，模型集成文档介绍 LangSmith tracing；技术 trace 不能自动替代业务审批证据链，需补充审批主体、对象与授权版本。[L-persist][L-models] |
| 模型提供商适配 | 可自行接 provider SDK/HTTP，输入输出、重试与工具协议需统一；标准库没有统一 LLM 协议。 | 官方有 DeepSeek 与 pi-ai 路由；providers 文档包括 API 协议/base URL/模型配置。具体模型 tool calling 与结构化输出兼容性需契约测试。[H-providers] | 核心可调用应用节点；LangChain 提供多个 provider 的统一模型接口，但须装相应集成包，模型能力仍有差异。[L-models] |
| 许可与维护成本 | Python 为 PSF 许可；额外库逐项核查。自建工时无实测，不能把无许可证费当成低维护成本。[P-license] | MIT；第三方依赖另有 notices；developer preview 与升级兼容性是已记录事实。未估算适配工时。[H-license][H-root] | 核心包 MIT。Agent Server/完整 LangSmith 部署不可自动继承核心许可；standalone 文档需 Postgres、Redis 等并注明生产运维责任，完整自托管 LangSmith 是 Enterprise 附加项。具体产品商业条件须选定版本后核查，不据此断言所有 Agent Server 方案均免费或均收费。[L-package][L-standalone][L-selfhost] |

## 三条路线分别剩下什么

- **Python 自建：** 以现成语言积木换取流程自由度，但要明确自建哪些状态机、任务队列、审批和权限部件。这里的工作量是待估项，无可靠依据给出天数。
- **Harness 复用：** 重点验证 owner 默认与非 owner 扩展如何接入真实用户；静态恢复记录是否足够承载业务任务恢复；插件/工具通道是否都受同一外部边界控制。其“已有安全机制”不代表满足强隔离，官方 SAFETY 也明确不能把 Harness 当作不可信任务的唯一安全控制。[H-auth][H-access][H-persistence][H-safety]
- **LangGraph：** 可复用状态图/checkpoint/interrupt，先决定采用“核心库+自建服务”还是“Agent Server 部署”，再估算缺口。两者的后台 API、认证、运维和许可不是同一份能力清单。[L-persist][L-background][L-auth][L-standalone]

## 候选最小验证场景（尚未执行）

以下只是可移交验证票的实验设计，结果必须由运行证据填写，不能因文档支持就标通过。使用合成文件、假工具和测试身份即可，不需要业务数据。

| 场景 | 试验动作 | 可核查结果 |
|---|---|---|
| 页面断开 | 发起一个分阶段长任务；关闭页面/断网；重新打开。 | 服务继续到下一阶段；页面恢复相同 task/run id；没有隐式重复提交。分别覆盖普通执行和等待审批两种状态。 |
| 崩溃恢复与副作用 | 在写入前、写入后但 checkpoint 前、checkpoint 后强杀 worker 并重启。 | 展示明确恢复点；文件/提交等外部效果最多执行一次，或明确标“结果未知需人工核对”；不能静默重做。 |
| 审批持久性 | 暂停审批后重启服务；原提交人尝试自行批准；撤销审批人权限后再回复。 | 原请求与内容版本仍可追踪；未经有效授权不执行；旧许可不用于新改动。 |
| 工具防绕过 | 同一写入分别通过文件工具、shell、Python、子代理、插件入口发起；测试符号链接跨目录。 | 所有路径遵守同一策略；拒绝发生在真实副作用前；每次拒绝能关联任务与主体。 |
| 双任务隔离 | 两任务用同名文件与不同假凭据并行；尝试跨任务写入、审批、恢复或取消，并测试凭据与受保护策略读取。 | 按待确认的操作权限矩阵拒绝越权修改和控制；凭据与受保护策略不可读。材料、结果与日志读取按既定可见范围验证，不预设跨任务读取一律拒绝。记录资源峰值，不把“两条协程并行”当隔离证据。 |
| 身份与恢复 API | 用不同身份查询 task/thread 并尝试写入、恢复、取消和直接调用审批端点。 | 查询按既定可见范围允许或拒绝；操作权限待权限设计明确，服务器端须拒绝越权写入、审批、恢复和取消，不能仅靠隐藏按钮。Harness 需额外证明受限用户请求不会落入 owner。 |
| 模型替换与异常 | 两个候选 provider 跑同一工具调用/结构化输出样本，注入超时与限流。 | 输出契约一致、重试不重复副作用、费用可归属；兼容失败记录具体 provider/model/version。 |
| 审计对账 | 完成一次审批、执行、失败、恢复的闭环，导出事件并对照外部效果。 | actor、task、action、对象内容版本、审批决定与执行结果均可关联；验证修改日志的权限边界。 |

## 来源

所有 GitHub 内容通过 gh 从官方仓库读取；Harness 源码归档只解压阅读，未安装依赖、未执行项目脚本。链接给出固定提交供复核。在线官方文档访问日为 2026-10-26，可能后续更新。

[H-root]: https://github.com/deepseek-ai/deepseek-harness/tree/22ddee3792402c3e70ec5f86a63c3e3e895a2f60
[H-auth]: https://github.com/deepseek-ai/deepseek-harness/blob/22ddee3792402c3e70ec5f86a63c3e3e895a2f60/packages/client/connection/README.md
[H-access]: https://github.com/deepseek-ai/deepseek-harness/blob/22ddee3792402c3e70ec5f86a63c3e3e895a2f60/packages/api/remote-access/README.md
[H-web]: https://github.com/deepseek-ai/deepseek-harness/blob/22ddee3792402c3e70ec5f86a63c3e3e895a2f60/docs/subsystems/web-server.md
[H-persistence]: https://github.com/deepseek-ai/deepseek-harness/blob/22ddee3792402c3e70ec5f86a63c3e3e895a2f60/docs/subsystems/persistence.md
[H-approval]: https://github.com/deepseek-ai/deepseek-harness/blob/22ddee3792402c3e70ec5f86a63c3e3e895a2f60/docs/subsystems/approval.md
[H-tools]: https://github.com/deepseek-ai/deepseek-harness/blob/22ddee3792402c3e70ec5f86a63c3e3e895a2f60/packages/core/tools/README.md
[H-sandbox]: https://github.com/deepseek-ai/deepseek-harness/blob/22ddee3792402c3e70ec5f86a63c3e3e895a2f60/docs/subsystems/sandbox.md
[H-safety]: https://github.com/deepseek-ai/deepseek-harness/blob/22ddee3792402c3e70ec5f86a63c3e3e895a2f60/SAFETY.md
[H-session]: https://github.com/deepseek-ai/deepseek-harness/blob/22ddee3792402c3e70ec5f86a63c3e3e895a2f60/docs/subsystems/session.md
[H-providers]: https://github.com/deepseek-ai/deepseek-harness/blob/22ddee3792402c3e70ec5f86a63c3e3e895a2f60/docs/user/guide/providers.md
[H-license]: https://github.com/deepseek-ai/deepseek-harness/blob/22ddee3792402c3e70ec5f86a63c3e3e895a2f60/LICENSE
[L-package]: https://github.com/langchain-ai/langgraph/blob/a25bfafe4858d2a85bede09ba83f9a591d1f1133/libs/langgraph/pyproject.toml
[L-types]: https://github.com/langchain-ai/langgraph/blob/a25bfafe4858d2a85bede09ba83f9a591d1f1133/libs/langgraph/langgraph/types.py
[L-persist]: https://docs.langchain.com/oss/python/langgraph/persistence
[L-background]: https://docs.langchain.com/langsmith/background-run
[L-auth]: https://docs.langchain.com/langsmith/auth
[L-middleware]: https://docs.langchain.com/oss/python/langchain/middleware/custom
[L-models]: https://docs.langchain.com/oss/python/langchain/models
[L-standalone]: https://docs.langchain.com/langsmith/deploy-standalone-server
[L-selfhost]: https://docs.langchain.com/langsmith/self-hosted
[P-async]: https://docs.python.org/3.14/library/asyncio-task.html
[P-process]: https://docs.python.org/3.14/library/multiprocessing.html
[P-audit]: https://docs.python.org/3.14/library/sys.html#sys.addaudithook
[P-logging]: https://docs.python.org/3.14/library/logging.html
[P-license]: https://docs.python.org/3/license.html
