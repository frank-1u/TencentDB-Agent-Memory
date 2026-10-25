# Orca worktree 与 TencentDB Agent Memory：源码事实

日期：2026-10-25。本节只读核对当前仓库 `feat/server_team`，HEAD `6a4bb55ea54834d8265ce52024d3e3c75db30d38` 的内置 CodeGraph 实现；没有启动服务、操作 Orca、迁移索引或修改配置。索引情况与工作流结论是源码证据，不是部署验收。

## CodeGraph 事实

### 1. 内置 CodeGraph 资产与 GitNexus 索引是两套机制

MemoryKnowledge 使用 `@colbymchenry/codegraph` 的运行时与 ToolHandler，见 [bridge.ts:17](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/engines/code/bridge.ts:17)。它把一个仓库分支索引包装为有 `code_graph_id`、团队归属和工具入口的知识资产；这不是根目录 `AGENTS.md` 所描述的 GitNexus `.gitnexus` 索引。

CodeGraph 的服务目录及实例池由 Knowledge 服务管理，见 [module.ts:443](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/module.ts:443)。因此“新建一个 Orca worktree”与“迁移该 CodeGraph 资产的索引”并没有天然对应关系。GitNexus 是否需要在某个 worktree 中索引，应另按 GitNexus 的仓库绑定和新鲜度处理，不能用这里的服务端资产规则代替。

### 2. 普通来源是 Git URL 和指定分支，不是客户端工作目录

CodeGraph 创建请求要求 `repo_url`，未填写 `branch` 时使用字面值 `main`，见 [code-graph.ts:184](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/routes/code-graph.ts:184)。这不表示自动读取远端默认分支，更不表示自动读取当前 Orca worktree 的分支。

SourceFetcherRegistry 当前只注册 Git fetcher，LocalSourceFetcher 仍是未来注释，见 [registry.ts:14](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/source-fetcher/registry.ts:14)。虽然类型探测能把 `file://`、`/absolute/path`、`./relative/path` 识别成 local，但没有对应 fetcher，见 [registry.ts:25](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/source-fetcher/registry.ts:25)。Git fetcher 的协议白名单为 HTTPS 和 SSH，拒绝 `file://` 等，见 [git-fetcher.ts:169](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/source-fetcher/git-fetcher.ts:169)。因此不能把 `/home/.../orca/.../worktree` 填进普通 `repo_url` 就认为已支持本地目录索引。

实际构建 worker 在 Knowledge 服务端执行，克隆目标为该资产自己的 `dir`，见 [module.ts:269](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/module.ts:269)。目录由 `dataRoot / service_id / team_id / code_graph_id` 生成，见 [code-graph-service.ts:189](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/store/code-graph-service.ts:189)。`dataRoot` 来自 Knowledge 服务配置，默认 `KNOWLEDGE_DATA_DIR=./data`，见 [config.ts:168](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/config.ts:168)。若服务运行在容器中，这个目录属于服务端/容器的数据空间，不是发起 LLM 请求时客户端的 cwd。

### 3. 它索引所登记远端分支的快照，不自动跟随本地未提交代码

首次抓取使用 `git clone --depth 1 --branch <登记的branch>`，并记录 checkout 的 HEAD，见 [git-fetcher.ts:187](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/source-fetcher/git-fetcher.ts:187)。分支不存在时返回可操作错误，不自动改用 `master` 或另一个分支，见 [git-fetcher.ts:214](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/source-fetcher/git-fetcher.ts:214)。创建 API 没有单独的指定 commit 参数；返回的 `commit_hash` 记录本次建图版本，不能把它理解为客户端要求固定 checkout 任意 SHA 的配置。

显式 `/sync` 按资产 ID 取出已登记的 repo/branch；它不会接受一个当前 worktree 分支来自动替换资产分支，见 [code-graph.ts:280](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/routes/code-graph.ts:280)、[code-graph-service.ts:404](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/store/code-graph-service.ts:404)。可增量时抓取远端该分支并重置服务端 checkout 到 `origin/<branch>`，保留 `.codegraph` 索引目录，见 [git-fetcher.ts:244](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/source-fetcher/git-fetcher.ts:244)；其余分支按 worker 条件重新克隆，见 [module.ts:288](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/module.ts:288)。

因此，某个本地 worktree 的未 push commit、未提交修改、未跟踪文件，不会仅因该 worktree 发起了会话而进入此远端资产索引。资产更新完成后记录 `commit_hash` 与 `last_sync_at`，见 [code-graph-service.ts:458](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/store/code-graph-service.ts:458)；界面/API 明细也返回 branch、commit_hash、同步状态与时间，见 [api-helpers.ts:146](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/api-helpers.ts:146)。这些字段可以用于判断快照新鲜度。

### 4. 有内部“本地构建”，但路径仍固定在服务端资产目录

现有 `/code-graph/build` 跳过 clone/fetch，用**该资产已经存在的服务端目录**重建索引，主要供内部/调试场景，见 [code-graph.ts:300](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/routes/code-graph.ts:300)。`BuildOptions` 明确不接受外部传入目录，见 [code-graph-service.ts:74](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/store/code-graph-service.ts:74)；worker 仍对自身 `dir` 调 `indexProject`，见 [module.ts:376](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/module.ts:376)。

如果管理员事先把内容放进那个服务端目录，内部构建可索引目录中实际存在的代码；但它不是“自动挂接客户端 cwd”，也没有自动收集 Orca worktree 文件的传输过程。这个能力不构成让每个新 worktree 手动复制/迁移索引的必要理由。上述事实也不证明这种目录灌入流程已经部署或受普通客户端支持。

### 5. 资产归属与 Agent 绑定，与 worktree 的磁盘路径分开

普通 CodeGraph 的创建幂等键为 `(service_id, team_id, repo_url, branch)`，同一组合返回已有资产，见 [sqlite-store.ts:65](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/store/sqlite-store.ts:65)。它没有把 worktree 路径纳入资产标识。换成另一条远端分支时可以登记另一资产；这属于需要分支快照时的显式选择，并非迁移现有 worktree 索引。

Panel 创建接口要求 Team 成员身份，见 [code-graph-routes.ts:47](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryPanel/src/panel/http/routes/knowledge/code-graph-routes.ts:47)。将 CodeGraph 分配给业务 Agent 时，`/knowledge/allocate` 校验资产与 Agent 同属该 Team，再写 `agent-fixed-asset` 绑定，见 [allocate-routes.ts:53](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryPanel/src/panel/http/routes/knowledge/allocate-routes.ts:53)、[allocate-routes.ts:108](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryPanel/src/panel/http/routes/knowledge/allocate-routes.ts:108)。

Proxy 注入知识时优先读取当前业务 Agent 的绑定列表，按 asset ID 联查资源，而不是按客户端目录找资产，见 [knowledge-tools-injector.ts:337](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/injection/injectors/knowledge-tools-injector.ts:337)。该 injector 注释也说明缺 user-key/Agent 时有 team-list 兼容路径，见 [knowledge-tools-injector.ts:12](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/injection/injectors/knowledge-tools-injector.ts:12)；不应据此承诺每个正常业务 Agent 都自动获得整个 Team 的所有资源。

同一主仓库的多个 worktree **可以复用同一个已授权、已绑定的 CodeGraph 资产**，没有必要仅因新目录而复制资产。能否在新会话得到该资源，取决于它最终选中的 Team/业务 Agent 及权限；新建 Orca worktree 本身不会替用户建立新的业务 Agent 绑定。

### 6. 已有仓库锚点提示；没有在本次追踪路径中发现自动 worktree 分支同步

Proxy 的资源提示并不是完全不认识仓库：

- [knowledge-tools-injector.ts:120](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/injection/injectors/knowledge-tools-injector.ts:120) 从 Git URL 派生 repo slug。
- [knowledge-tools-injector.ts:162](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/injection/injectors/knowledge-tools-injector.ts:162) 给 CodeGraph 资源渲染 `match` 和 `branch` 属性。
- [knowledge-tools-injector.ts:180](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/injection/injectors/knowledge-tools-injector.ts:180) 提示模型比对当前工作区 `git remote / 仓库名`，匹配后才使用该索引；并明确索引是分支快照，刚修改或未提交代码的精确内容以当前工作区源码为准。

这是**模型收到提示后的仓库锚点判断**，不是服务端从 cwd header 自动识别 Orca worktree、复制绑定或切换分支的程序接口。工具调用按 `knowledge_id` / `code_graph_id` 选择资源；Knowledge 查询入口用服务 ID 和资产 ID 定位服务端目录，见 [code-graph.ts:373](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/routes/code-graph.ts:373)。当前 create/sync/build/query 合同中没有接受客户端 cwd 作为索引源的字段；`update-meta` 也只更新名称/说明，见 [code-graph.ts:265](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/routes/code-graph.ts:265)。

Bridge 内有一处 worktree 注释，是把服务端资产 `projectPath` 设置为 ToolHandler 默认 hint，防止它错用 **Knowledge 进程 cwd** 检测 worktree，见 [bridge.ts:89](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/engines/code/bridge.ts:89)。它不能证明已有 Orca worktree 自动发现功能。

| 能力 | 当前结论 |
|---|---|
| 绑定同一 Team/Agent 后复用远端分支 CodeGraph 资产 | 已有资产与绑定机制支持 |
| 提示模型按本地 git remote 识别匹配的已绑定仓库 | 已有注入提示支持 |
| 明确区分远端分支快照与本地未提交代码 | 已有注入提示说明；精确代码仍需本地读取 |
| 对已登记资产手动 sync，更新登记分支的快照 | 已有管理接口 |
| 创建 Orca worktree 就自动选择 Team/Agent/Task 或复制 CodeGraph 绑定 | 本次追踪合同没有此机制；若要免手选需另有启动参数/集成，不能声称已具备 |
| 自动把 worktree 的新分支或未提交修改推给服务端索引 | 本次追踪合同没有此机制；不应为本轮简单使用流程新增同步系统 |
| 把当前客户端目录直接作为普通 CodeGraph repo_url | 当前来源 registry/fetcher 不支持 |

### 最小 CodeGraph 使用方式

1. 对需要共享的仓库，在 Team 下登记一个明确远端分支的 CodeGraph，建图完成后绑定给常用业务 Agent。
2. 在不同 Orca worktree 中工作时，继续使用有权访问该资产的业务 Agent；不因目录变化迁移/复制资产索引。会话身份如何共享见后续记忆 scope 部分。
3. 使用 CodeGraph 理解仓库结构、符号和调用关系；要修改当前文件或检查未提交改动时，回到当前 worktree 的真实源码。需要判断差异时核对资产 branch/commit_hash/last_sync_at。
4. 远端分支已更新且确实需要新快照时，显式 sync 该资产。只有长期需要另一个远端分支的独立图谱时，再考虑另建该分支资产；这不是每个 worktree 的默认动作。

本节创建前执行了 GitNexus `query` 追踪相关流程，再用源码确认；对新文档执行 `impact` 返回 UNKNOWN，随后确认目标尚不存在。本轮只创建此研究文件，不把 UNKNOWN 作为修改既有业务符号的安全证明。

## 已确认的方案取舍

用户明确认为聚合记忆方案过度设计，恢复上一版：**编码请求由 MemoryProxy 注入/归档后交给 cc-Switch；记忆加工继续使用独立的 MEMORY_LLM API 配置**。不继续推进 cc-Switch 聚合记忆，也不为本次 worktree 使用流程新增索引迁移、分支上传或后台同步系统。

## Proxy 模式下，记忆究竟按什么范围划分

这不是“自动按项目目录划分”或“一个用户无条件统一一份”的二选一。Proxy 根据登录身份和会话选中的业务 Team / Agent / Task 组装数据身份，记忆层按各自范围读写。

| 层次 | 当前源码范围 | 对多个 worktree 的影响 |
|---|---|---|
| 服务实例 | URL 的 space/service ID | 通常复用同一实例；不是本地工作目录 |
| 用户 | user_key 校验得到 user_id，配合资产 ACL | 同一 Key 不等于自动选中相同 Team / Agent |
| L0 原始对话 | Team / User / Agent / Session，以及可选 Task | 不同会话保留自己的记录，不要求合并 Session |
| L1 原子记忆召回 | Team / User / Agent，可选 Task；明确跨 Session | 同 Agent 可跨会话召回，但当前 Task 不同会限制结果 |
| L2/L3 长期记忆 | Team + 业务 Agent；忽略 User / Session / Task | 同组合可跨目录、会话、任务，甚至授权用户积累共享 |
| CodeGraph / Wiki 资源 | Team 归属、业务 Agent 的资产绑定与权限 | 不按客户端 cwd 复制；是否注入取决于最终身份与绑定 |

关键证据：

- [deriveTdaiIdentity:26](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/tdai/identity.ts:26) 从 session/auth 读取 team_id、user_id、agent_id、session_id、task_id。没有用 cwd / Git 目录生成这个身份；缺必要身份返回 null，而不是自动建立项目记忆空间。
- [TdaiClient.addConversation:142](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/tdai/client.ts:142) 将这些维度写入 L0；[searchL1ForCtx:191](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/tdai/client.ts:191) 会向原子召回传当前 Session / Task。
- [handleAtomicSearch:1212](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryCore/src/gateway/v2-router.ts:1212) 明确不将 sessionId 加入检索过滤，因此可召回其他会话的 L1；但会加入 teamId/userId/agentId/taskId。不能因为它“跨会话”就说它“跨任何任务/用户”。
- [buildProfileIsolationScope:34](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryCore/src/core/profile/profile-scope.ts:34) 生成 `team:<teamId>|agent:<agentId>`，注释明确 L2/L3 忽略 user/session/task；[profileScopeFilter:61](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryCore/src/core/profile/profile-scope.ts:61) 用匹配的范围过滤读请求。没有 Team 时有 user fallback，但正常 Proxy 身份要求 Team，不能将兼容 fallback 当作全局记忆策略。

### 最小项目映射建议

一个逻辑仓库固定使用一组 Team / 业务 Agent。个人使用时，“一仓库一个 Team，下面一个共享开发 Agent”较直观；已有 Team 也可用不同业务 Agent 区分逻辑项目。主目录与 Orca worktree 都用相同组合，不按每个目录新建 Team 或 Agent。

每个新会话仍保留独立 Session。Task 按实际工作内容决定，不为每个目录强制新建；若不同 worktree 对应不同任务，项目长期 L2/L3 仍共享，但 L1 召回会受任务范围限制。需要共享整个项目的 L1 时，无 Task 会放宽该维度；这与所有任务私有上下文自动合并不是一回事，应先决定真实任务语义。

同一用户的两个无关仓库若都选择同一个 Team / Agent，L2/L3 不会因为 cwd 不同自动隔离。反过来，同一仓库的多个 worktree 只要选择同一组合，就不需要“迁移记忆目录”。这里业务 Agent 不是 Codex / Claude Code 应用类型：两客户端可以使用同一业务 Agent，推理供应商仍按各自 cc-Switch 应用配置工作。

## Orca 启动关联：现有预选机制与未验证部分

当前已有 [parsePresetIdentity:76](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/session/preset.ts:76)，从请求头读取预选 Team / Agent / Task。默认头名在 [config.ts:145](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/config.ts:145) 为 `x-team-id / x-agent-id / x-task-id`；这些值必须经过用户可见资源校验，见 [SessionInitConfig:298](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/types.ts:298)。例如 Claude Code 的 [init.ts:802](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/session/claude-code/init.ts:802) 可用已校验 Team+Agent 注册，无 Task 时保持较宽的 L1 召回。

因此可先手工在每个新会话选择固定项目身份；后续只需考虑启动时携带同一组身份的轻量关联，无须新增一套 worktree 记忆数据库。现有 Header 机制不证明 Orca 已经自动下发它们，也不证明所有 coding-agent 的自定义头和会话行为都完成联调。

本轮按环境事实选定 `ORCA_CLI_COMMAND=orca-ide`，执行只读 `orca-ide skills get orca-cli --json`。Windows PowerShell 拒绝执行 `\\wsl.localhost\Ubuntu-24.04\home\v-lushuai\.local\share\orca\orca-wsl-bridge.ps1`，原因是脚本未数字签名，`FullyQualifiedErrorId=UnauthorizedAccess`。没有切换其他 Orca 可执行文件、修改执行策略或恢复旧 provider 会话。因此本轮未验证当前 Orca worktree 清单、创建 hook 或身份自动继承行为。

## 实施与验证界限

此次仅更新方案状态并核对源码。GitNexus 当前索引 commit 与 HEAD `6a4bb55...` 一致，使用图查询后对照源码确认。没有修改业务实现、客户端配置、Orca 设置或任何索引数据，也未启动业务服务或进行多 worktree 真实请求验收。未合并 worktree 的事实应保留分支/提交/任务语境；共享长期记忆不代表系统已经保证了分支事实隔离或只在合并后写入长期记忆。

## Orca 创建后脚本：几种简易方案

用户提供了正在使用的 `cmem-ensure-project` 调用。本轮完整只读了 [现有脚本](/home/v-lushuai/.local/bin/cmem-ensure-project:1)，没有执行或修改它：

- 第 17–24 行优先读取 `ORCA_ROOT_PATH / ORCA_WORKTREE_PATH`，否则使用 Git common dir 归一主仓库。
- 第 60 行起把主仓库和 worktree 路径模式归入同一个项目名称；最终修改 claude-mem 的 `CLAUDE_MEM_PROJECT_ENVIRONMENTS`，不直接配置 TencentDB 的 Team/Agent。
- 脚本采用幂等写入、备份和原子替换，默认仅告警不阻断。这种 ensure 模式可复用于本任务，但不应顺带改动现有 claude-mem 脚本。

创建后脚本与后续客户端进程是不同执行阶段。仅在 setup 脚本内 export Team/Agent 变量，不会自动修改未来由 Orca 启动的进程环境；需要生成可消费的文件，或由真正的启动包装器 / session hook 在会话启动时读取。

### 当前客户端配置约束

本机 shell 可调用 `codex-cli 0.164.0` 与 `Claude Code 2.1.302`。这证明当前 PATH 中工具的版本，不证明 Orca 当前 provider 采用相同二进制或支持任意自定义启动命令。

[OpenAI Docs：Advanced Configuration](https://learn.chatgpt.com/docs/config-file/config-advanced) 明确：项目 `.codex/config.toml` 只在可信项目加载，且不能覆盖 `model_provider / model_providers / profile / profiles` 等 provider/auth 配置；支持 `http_headers / env_http_headers` 的 provider 配置应位于允许的用户层或运行时覆盖。因此不能承诺给每个 worktree 写一份 provider header 到项目配置就会生效。

[Claude Code Settings](https://code.claude.com/docs/en/settings) 描述 `.claude/settings.local.json` 的 scope，也说明部分 Git worktree 场景使用主 checkout 的 local 文件，并列出 Windows 等例外。因此同样不能无条件把每个 worktree 当作一个独立 local settings 文件位置。最终需按真实启动宿主、平台与版本验证。

本轮只读取现有 hooks 的事件名称：用户级 Codex 和 Claude 配置中都已包含 SessionStart。没有读取输出任何鉴权值、修改 hooks 或信任信息。[OpenAI Docs：Hooks](https://learn.chatgpt.com/docs/hooks) 支持 SessionStart 与 cwd/session_id 输入；这些字段能成为轻量启动 hook 的依据，但仍需核对它们与 Proxy 请求中的会话标识一致。

### 四个候选的具体区别

| 方案 | 创建阶段 | 会话阶段 | 改动与取舍 |
|---|---|---|---|
| 1. Orca 项目固定 ID，生成每 worktree 启动文件 | 项目脚本传入 Team/Agent，幂等写小启动入口 | Orca 调用生成入口，透传原客户端参数并附项目身份 | 最直观；无额外绑定服务。需要 Orca 可选择入口，每个 worktree 有生成文件 |
| 2. Git 公共目录一份绑定 + 通用包装器 | ensure `<git-common-dir>/tdai-project.json` | 包装器按 cwd 找公共目录，读取 ID，复用现有 Header 预选 | 大量 worktree 首选；不复制绑定，不写全局“当前项目”。需要验证 Orca 启动命令可配置 |
| 3. 共享绑定 + SessionStart 预绑定 | 与 2 相同，仅确保文件 | 用户级 hook 根据 cwd/session_id 向 Proxy 注册会话身份 | 保留原生 Orca 启动方式；需要新增受保护的注册入口并验证时序、会话 ID 和恢复行为 |
| 4. 全局目录映射 + 启动消费 | 类似 cmem 脚本，登记规范化根目录到 Team/Agent | hook 或包装器查表后分别使用 3 或 2 的传输方式 | 最接近现有习惯；多一个全局映射文件。多项目并发更新需加锁、原子写入并保留无关条目 |

其中“根目录绑定文件”和“全局映射”是存储选择，“包装器”和“SessionStart”是把身份送到 Proxy 的选择，不需要为这些候选增加模型路由、CodeGraph 同步或另一个 memory namespace。

方案 2 的文件形态仅为示意，不是已经创建的文件：

```json
{"team_id":"team_xxx","agent_id":"agent_xxx"}
```

文件存放在 Git 公共目录，不是每个 worktree 的 `.git` 文件中。绑定键使用规范化仓库根 / common dir，不能只靠最后一段目录名，避免不同仓库同名时串项目。API Key/OAuth 保持既有凭据来源，Task 按真实业务可选，Session 保持每会话独立。CodeGraph 已绑定在业务 Agent 上，无需在创建脚本中重新复制、索引或创建资产。

### 两种会话传输方式的实施范围

**包装器**：读取绑定并启动原客户端，把 Team/Agent 送到已有 `x-team-id / x-agent-id` 预选链。Codex 可由允许的 provider 层或命令行运行时配置映射环境请求头；不把仓库层配置当作可覆盖 provider 的入口。包装器需保留 Orca 原命令参数、标准输入/输出与信号；若原来启动 `codex app-server`，不能改成另开交互 CLI。Orca 是否允许这一路径，当前尚未验证。

**SessionStart hook**：仅把 ID 写进模型提示并不能保证预选。若选此方案，应由 hook 同步调用受鉴权保护的会话绑定入口，第一条推理请求再恢复绑定。当前 [BindingRepo](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/db/binding-repo.ts:43) 有持久化 ID 组合的内部契约，[SessionStore.getOrRecover](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/session/store.ts:324) 可恢复现有绑定；本轮相关 HTTP 路由没有找到可直接给外部 startup hook 使用的注册合同，不能把内部方法说成现成 API。

若实施 3，需增加小型认证/授权和 session binding HTTP 接口，复用既有持久化，不让 hook 直接写数据库。必须校验 user、Team/Agent 权限及客户端来源，确认 hook session_id 与实际 Proxy session key 对齐，并处理 resume/compact 时的幂等、已选 Task 和手工会话选择；不能每次 hook 都无条件覆盖已有会话。创建脚本可非阻断，但新会话绑定失败应可见，不能假称已经共享记忆。

### 当前推荐与下一步边界

优先评估 **2：共享根绑定 + 通用包装器**，因为它复用已有 Header 预选，服务端新增工作较少。若 Orca 固定原生 provider 启动方式，再评估 **3：共享根绑定 + SessionStart**。习惯集中管理多个项目时，可把绑定存储替换为 4 的全局表。

以下拟议调用只说明 Orca 脚本形态，命令尚未创建：

```sh
[ -x "$HOME/.local/bin/tdai-project" ] &&
  "$HOME/.local/bin/tdai-project" ensure || true
```

首次绑定可显式提供 Team/Agent，之后重复 ensure 只复用。现有 `cmem-ensure-project` 不需替换；二者可在 Orca 创建后脚本中顺序执行。此次只新增方案说明，没有生成包装器、修改全局配置、添加 hook、修改用户脚本或运行会话绑定接口。

> 2026-10-25 复评更新：结合两仓库源码与 `/implement-spec` skill 的自建 worktree 约束，主推由方案 2 改为**方案 5 · Proxy 侧身份解析**，见下节。方案 2 降级为不改 Proxy 时的备选。

## 复评：方案 5 · Proxy 侧身份解析（2026-10-25 追加）

用户要求结合需求与两仓库代码复评上述四方案。复评发现四个方案共享同一隐含前提——「把 `x-team-id` / `x-agent-id` 头送上客户端请求」——而 MemoryProxy 在会话初始化时本就持有完整请求体，可以在服务端解析身份，绕开全部客户端侧改造。用户确认后主推改为方案 5；随后用户补充 `/implement-spec` skill 也会自建 worktree，本节一并纳入。

### 新增源码证据

- `stageSessionInitOrchestrate`（[session-init-orchestrate.ts](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/pipeline/stages/session-init-orchestrate.ts)）是 anthropic(claude-code) / codex / workbuddy / openai-chat 四个 runner 共用的会话初始化编排入口；`parsePresetIdentity(input.config.sessionInit, input.headers)` 在此组装预选身份。追加预选来源可一处覆盖全部客户端。
- Claude Code 协议的 system prompt 在 `body.system`（[claude-code/init.ts:55](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/session/claude-code/init.ts) 注释与 `systemAppend` 契约），其中含 `<env>Working directory: ...</env>` 结构化块，仓库级 CLAUDE.md 内容也进入 system prompt；Codex 协议的 system prompt 在 `body.instructions`（[codex.ts:1390](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/pipeline/runners/codex.ts)），含 `<environment_context>` 工作目录与 AGENTS.md 内容。两者均随会话首请求出现；session-init 只在新会话触发，既有会话经 `store.getOrRecover` 恢复，不受影响。
- 预选身份租户校验 `resolvePresetIdentity`（[preset.ts:102](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/session/preset.ts#L102)）不依赖身份来源：值必须落在 kernel 返回的用户可见 `teams[]` 内，错配回退交互表单。目录 / 标记派生身份可走同一条校验链。
- 全仓检索未发现现有 cwd / environment_context 解析（仅 config.ts 涉及进程自身 cwd），也不存在按用户记忆上次选择的机制（`defaultTaskId` 只是任务级“暂时跳过”虚拟条目）。目录绑定与标记解析均为待实现功能，不是已有能力。
- 持久化范式：`BindingRepo`（[binding-repo.ts:43](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/db/binding-repo.ts#L43)）的 KV 小纸条模式。登记入口范式：mem-command 框架（[mem-command/commands/](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/mem-command/commands/)，已有 session-reset / create-task 等）。注意 `mem:` 命令从**最后一条 user message** 解析（[parser.ts](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/mem-command/parser.ts)）——它是用户输入通道，无人在场的 subagent 会话不能依赖它触发登记；这决定了临时 worktree 必须靠仓库标记信号而非登记命令覆盖。

### /implement-spec 自建 worktree 的事实

- [SKILL.md](/home/v-lushuai/.claude/skills/implement-spec/SKILL.md)：每个 implementer subagent 在**自己的 worktree 和 branch** 上工作，spec 完成后统一清理——worktree 生命周期短、由 skill 流程自动创建，没有人工登记时机。
- 布局证据：GitNexus 注册表存在 `/home/v-lushuai/Codes/.worktrees/canvas-observability-integration`（canvas_python_backend 的 worktree，对应分支 integration/task-observability）。`.worktrees` 父目录**跨仓库共享**，worktree 名来自分支描述、不含仓库名——仓库级目录前缀无法覆盖该布局，这是目录前缀单信号设计的硬缺口。
- 未验证：Claude Code subagent 的 API 请求是否携带父会话 session_id。若共享，subagent 直接继承父会话注册（父会话 cwd 在主 checkout，可被前缀命中）；若不共享，subagent 新会话必须由标记信号兜住，否则会触发交互表单——无人在场的会话无法作答表单（[preset.ts:19](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/session/preset.ts#L19) 的 HEADER_ONLY_AGENTS 注释说明了这类客户端把未知 tool_use 渲染为错误的同类风险）。列为验收项。

### 方案 5 设计（双信号 + 显式优先级）

预选优先级：**header 预选 > 仓库标记 > 目录前缀 > 交互表单**。命中且校验通过即免表单注册；未命中或校验不过回退现有表单，失败模式即现状。

- **信号一 · 仓库绑定标记**：在业务仓库的 AGENTS.md / CLAUDE.md 提交一行标记（示意 `tdai-binding: team_xxx agent_xxx`）。它随 git 检出进入任何 worktree 的 system prompt（Claude Code 经 CLAUDE.md / Codex 经 AGENTS.md），Proxy 在 session-init 解析。天然覆盖 Orca、/implement-spec、手动 `git worktree add` 及任何未来工具创建的 worktree，零登记动作、与磁盘布局无关。Team/Agent 是业务身份非凭据，写入公司仓库可接受；对外部开源仓库（如本仓库）不便提交标记，用信号二。
- **信号二 · 每用户目录前缀表**：`前缀 → {team_id, agent_id, task?}`，最长前缀匹配。Orca 布局每仓库登记一条父目录前缀（`~/orca/workspaces/<repo>`）+ 主 checkout 根；更深层前缀可对单个 worktree 覆盖 Task。存储可沿用 BindingRepo 式 KV。
- **安全模型**：标记与 cwd 同为客户端可见输入，与 header 同信任级；一切解析结果仍须通过用户可见 teams[] 校验——同事 fork 含标记的仓库时，若标记指向其不可见的 Team，自然失效回退表单，不产生越权。绑定表按用户隔离。
- **登记入口**：拟新增 `mem:bind-project`（交互会话内输入，接受可选显式路径参数），捕获客户端实际发送的 cwd 字符串，规避 WSL / 容器路径形态差异。subagent 会话不依赖它。
- **需要的 Proxy 改动（均为待实现）**：各 runner 按协议抽取标记与 Working directory 并传入编排层（编排输入现为 headers + synthesizeMessages / recoveryMessages，需扩展）；每用户前缀表存取；`SessionInitConfig` 增加 `directoryAutoSelect` / 标记开关（对齐 `headerAutoSelect` 形态，[types.ts:298](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/types.ts#L298)）；`mem:bind-project` 命令；Panel CRUD 可后补。

### 与方案 2 的取舍（用户已确认改推方案 5）

| 维度 | 方案 2（绑定文件+包装器） | 方案 5（Proxy 身份解析） |
|---|---|---|
| 客户端 / Orca 改动 | 包装器 + 用户层 header 配置；Orca 启动命令可替换性未验证 | 零 |
| /implement-spec 临时 worktree | 不经包装器启动，不覆盖 | 标记信号全覆盖（与创建工具、布局无关） |
| 手动终端打开的 worktree | 不经包装器不生效 | 全覆盖 |
| Task 粒度 | 仓库级一份，表达不了 per-worktree Task | 深前缀可覆盖（信号二） |
| 代价 | 外部集成联调（三方均未验证） | Proxy 侧功能实现 + 部署（自有代码）；每仓库一次标记提交或前缀登记 |
| 主要风险 | Orca / 包装器 / Codex 配置 | env 块与标记格式依赖（结构化、稳定，列入验收）；subagent session_id 行为待实测 |

方案 1 / 3 / 4 维持原判。方案 5 落地后 header 预选保留最高优先级，方案 2 的包装器路径仍可与其组合。

### 新增验收项（并入 PHASE 顺序）

1. 真实请求确认两客户端环境块可被正确解析（Claude Code `body.system` `<env>`、Codex `body.instructions` `<environment_context>`）。
2. 标记解析：含标记仓库的新会话免表单注册；标记指向用户不可见 Team 时回退表单，不报错不越权；header 预选优先级高于标记与目录。
3. 实测 Claude Code subagent 请求的 session_id 归属（继承父会话 or 独立），确定 /implement-spec 场景对标记信号的依赖程度。
4. 登记-重启-再开新会话的绑定持久化；多前缀最长匹配与深层 Task 覆盖。

本轮复评只读核对源码并更新文档，没有实现任何 Proxy 功能、修改客户端配置或运行会话绑定接口。
