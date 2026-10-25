# cc-Switch 路由与 MemoryProxy 组合调研

调研日期：2026-10-25。本文是方案证据与边界记录；没有启动服务、修改客户端配置或发送模型请求。

## 结论

cc-Switch 本身支持 Codex 官方 ChatGPT 登录经过本地路由，因此上一轮“任何 Proxy 都无法使用官方订阅”的表述不成立。但 **cc-Switch 能转发官方 OAuth，不等于 MemoryProxy 已支持把记忆身份与上游 OAuth 同时传递**。原生 Codex 的官方路由和 Claude Code 页面的 `Codex OAuth` 供应商，还是两种不同的凭据机制。

建议把主要方案明确标成“需要最小适配、未经运行验证”：

```text
Codex（保留官方 OAuth；另外携带记忆 user_key）
  → MemoryProxy（拟新增双凭据校验；注入/归档记忆）
  → cc-Switch 本地路由 /v1/responses（官方 Codex provider）
  → ChatGPT Codex 官方服务（实际订阅模型请求）
```

另一条 API Key 路线可先作为现有配置能力的组合候选：客户端 → cc-Switch 自定义 MemoryProxy 供应商 → MemoryProxy → OpenAI 官方 API。它使用 API Key 计费，不能当作订阅方案。两服务串联仍需联调，不能称为已经可用。

这条 API 备选的路径拼接有源码依据：[handlers.rs:1067](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/handlers.rs#L1067) 将来访路径规范成 `/responses`；[codex.rs:1136](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/providers/codex.rs#L1136-L1165) 对自定义前缀地址直接追加该路径。因此 cc-Switch 自定义 Codex 供应商采用原生 Responses，上游 `base_url=http://<MemoryProxy>:8096/codex/default`，供应商 Key 填记忆业务用户的 `sk-mem-...`，会请求 `/codex/default/responses`；base URL 带 `/v1` 时会请求 `/codex/default/v1/responses`。两者均是 MemoryProxy 文档列出的入口，仍须真实接口验收。

## 当前源码与工作区事实

- 主要依据：本地只读源码 `/home/v-lushuai/Codes/cc-switch`，`main`，HEAD `de0a235ad6389ce4afa333b472cdee615a3f3987`；核对时无 staged、unstaged、untracked 变更。
- 该 commit 与调研时官方 GitHub `farion1231/cc-switch` 的 `main` 一致。没有在此仓库运行构建、修改代码或变更配置。
- 未发现该仓库内 `AGENTS.md` / `CLAUDE.md` 或 GitNexus runner。本文仅增加 TencentDB-Agent-Memory 下的新研究文档。
- 主代理只读核对了当前 Codex 的非敏感配置：`model_provider=custom`，provider 名称 `OpenAI`，`requires_openai_auth=true`，`supports_websockets=false`，`wire_api=responses`，`base_url=http://127.0.0.1:15721/v1`，没有 profile。它已与旧会话“没有 base_url”的快照不同；尚未证明监听该端口的进程身份、运行状态或真实上游。
- TencentDB-Agent-Memory 下 `deploy/global-images/.env` 尚不存在，不能把研究中的环境变量样例当作已部署状态。

## 一手资料及文档差异

用户提供的 [教程入口](https://ccswitch.io/zh/tutorials) 和 [文档入口](https://ccswitch.io/zh/docs?section=getting-started) 是前端渲染页面。公开抓取可见的页面壳链接到 [官方 GitHub 仓库](https://github.com/farion1231/cc-switch)。因此具体行为以同一 commit 的用户手册和本地源码互相印证。

当前 [应用路由手册](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/docs/user-manual/zh/4-proxy/4.2-routing.md) 明确把 Codex OpenAI Official 列为官方账号可以经过本地路由的例外。相对较旧的 [保留官方登录攻略](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/docs/guides/codex-official-auth-preservation-guide-zh.md) 的 FAQ 仍然不建议官方供应商经过路由，属于当前仓库内的文档冲突。当前源码存在官方路由实现与对应测试，所以不能用旧 FAQ 推导“完全不支持”。

## 路由模式究竟做什么

| 能力 | 当前证据 | 对本方案的含义 |
|---|---|---|
| 本地 HTTP 路由默认监听 `127.0.0.1:15721` | [本地路由服务手册](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/docs/user-manual/zh/4-proxy/4.1-service.md) | 端口示例与当前 Codex 配置一致，但还要确认实际进程 |
| Claude 与 Codex 分别维护路由目标 | [应用路由手册](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/docs/user-manual/zh/4-proxy/4.2-routing.md) | 两客户端的供应商配置不能互相代替 |
| 路由模式按选定供应商转发，可选故障转移 | 同上；[故障转移手册](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/docs/user-manual/zh/4-proxy/4.3-failover.md) | 关闭故障转移时只向选定目标发请求 |
| 聚合模式按 `/model` 模型前缀分流，不做故障转移；OpenAI Official 只能作默认供应商 | [聚合模式手册](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/docs/user-manual/zh/4-proxy/4.6-aggregation.md) | “路由”和“聚合”是不同决策，不应混为任意多供应商自动路由 |
| 常见 API Key 路由接管会把 live token 换成占位符，再由路由注入真实 Key | [应用路由手册](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/docs/user-manual/zh/4-proxy/4.2-routing.md) | 不能把这个占位符规则套到官方 OAuth 路线 |
| 可只启动服务，不接管客户端配置 | [commands/proxy.rs:31](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/commands/proxy.rs#L31-L36) | 可以设计 MemoryProxy 在前，cc-Switch 仅作为后方路由；仍需固定目标和联调 |

## 两条 OAuth 路径不能混用

### 1. 原生 Codex 的 OpenAI Official 路由

- [codex.rs:301](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/providers/codex.rs#L301-L346) 识别官方 ChatGPT 卡片；带有明确第三方上游或存储 API Key 的卡片不会仅因名称为官方而走此路线。
- [codex.rs:1025](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/providers/codex.rs#L1025-L1028) 对官方卡片返回固定 ChatGPT Codex base URL。修改卡片的普通 `base_url` 不能把 MemoryProxy 插到它后面。
- [forwarder.rs:1413](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/forwarder.rs#L1413-L1459) 校验客户端传入的官方凭据。若绑定托管账号，还会检查 `chatgpt-account-id` 和客户端 token 与所选账号匹配。
- [forwarder.rs:170](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/forwarder.rs#L170-L214) 拒绝缺失授权和 `PROXY_MANAGED` 占位授权。
- [forwarder.rs:2471](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/forwarder.rs#L2471-L2482) 让客户端 `Authorization` 原样通过。此分支不会从授权中心临时替客户端补一把 OAuth token。
- [forwarder.rs:3307](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/forwarder.rs#L3307-L3314) 把官方 route 错误归为不可重试，防止把某个官方账号的入站凭据拿去重试其他供应商；[provider_router.rs:77](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/provider_router.rs#L77-L87) 也把选定官方账号保持为单一目标。

### 2. Claude Code 里的 Codex OAuth 供应商

- [Claude 供应商预设:2651](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src/config/claudeProviderPresets.ts#L2651-L2680) 定义 `Codex` 预设，`providerType=codex_oauth`，上游格式为 Responses。
- [ClaudeAdapter:780](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/providers/claude.rs#L780-L785) 返回 `AuthStrategy::CodexOAuth` 占位策略；[forwarder.rs:2155](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/forwarder.rs#L2155-L2206) 随请求动态取得有效 managed token，并附加 workspace 身份。
- [ClaudeAdapter:718](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/providers/claude.rs#L718-L720) 同样固定官方目标，不能靠改预设地址在其后串 MemoryProxy。
- 因此不能拿此路径的“路由会自动注入 token”，论证原生 Codex 的 OpenAI Official 路径也会这样做。

## 主推方案的可行边界

### 双凭据是待新增设计

拟在 Codex 的独立 provider/profile 中保留 `requires_openai_auth=true`，让 `Authorization: Bearer <OAuth>` 继续来自官方登录；另外使用 `http_headers` 发送 `x-tdai-user-key: sk-mem-...`。**该 header 在此处是拟新增契约，不能假称 MemoryProxy 现已支持。**

MemoryProxy 需增加的职责：

1. 仅在明确开启的 Codex 双凭据模式读取并校验 `x-tdai-user-key`，将其用于记忆 Team/Agent/Task 身份；上游 OAuth 不作为 memory user_key。
2. 记忆鉴权完成后剥除 `x-tdai-user-key`，防止它进入 cc-Switch 或官方服务；日志也不记录两类凭据正文。
3. 原样保留官方 `Authorization`、`chatgpt-account-id` 及实际所需的会话标识；不能用静态 `.env` 参数保存会过期的 OAuth token。
4. 上游固定指向可信的本机 cc-Switch；禁用会覆盖 key/model/URL 的 MemoryProxy 实例路由、pool 或 cost guard 分支，避免身份或模型被其他路由改写。

cc-Switch 的独立服务模式源码可以解释目标选择：

- [provider_router.rs:45](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/provider_router.rs#L45-L53) 从 `provider_in_use` 获取当前目标。
- [mode/current.rs:63](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/mode/current.rs#L63-L88) 在没有代理模式时回落到直连当前 provider。所以可以把直连当前目标设成 OpenAI Official，仅启动路由服务，由前置 MemoryProxy 调用它；这是源码推导，仍需真实接口验证。
- [server.rs:330](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/server.rs#L330-L372) 暴露 Models、Responses、compact 入口。

客户端入口建议用专用 profile 或运行参数隔离。cc-Switch 自动接管会把 live `base_url` 写回自己的端口，绕过前置 MemoryProxy；应选择“仅启动服务、不接管客户端”，或清楚验证 profile 的覆盖优先级和退出恢复行为。当前已有会话可能固定了旧 provider，验收必须新建会话。

### 不应标成现成可用的组合

| 组合 | 结论与原因 |
|---|---|
| Codex → MemoryProxy（sk-mem Bearer）→ cc-Switch Official | 当前直接串联不可当成订阅方案：官方路由需要客户端 OAuth，sk-mem 不是该凭据 |
| Codex → cc-Switch Official → MemoryProxy → 官方 | 不能只填卡片 base URL 达成：官方 adapter 固定目标 |
| 把 OAuth token 手工填进 `PROXY_UPSTREAM_API_KEY` | 缺少刷新、账号匹配及双身份契约，不推荐作为方案 |
| cc-Switch 前后都把对方设成同一应用的上游 | 可能形成请求循环，不能使用一个可变路由指针同时承担入口和出口 |
| 保留 Codex 官方登录 + 走第三方模型 | 官方登录态不证明模型或计费仍在官方；必须看实际转发目标 |

## 实施前的验证清单

1. 先确认本机实际 cc-Switch 版本与上述 commit 的行为一致，确认 15721 的监听进程、Windows/WSL/Docker 网络可达地址。不能把容器内 `127.0.0.1` 当宿主机。
2. 未接管客户端时，向路由发真实请求，验证其实际选中 OpenAI Official；不能只凭静态配置得出成功。
3. 先通过官方直连路由证明 OAuth、所选模型和账号可用，再加 MemoryProxy；两条链路以同一账号、新会话比较。
4. 验证首次 Session Init 表单、第二轮记忆注入、工具调用往返、SSE、compact 和辅助 Models 请求。不要只把首轮 200 当作通过。
5. 核对代理日志中的真实目标、模型及官方账号用量；客户端显示名称、保留的登录图标和被 mask 的 model 名不能单独作为模型来源证据。
6. 明确错误要原样返回，不自动改用第三方供应商；验证额度耗尽、token 刷新、账号不匹配和 router 停止时的行为。

## 本次文档变更验证

创建前对新文档路径执行 GitNexus `impact`：目标尚不存在，返回 `risk=UNKNOWN`；随后用文件存在性检查确认是新文档，并非现有函数或模块修改。没有把 UNKNOWN 解读为低风险，也没有修改任何已存在的业务符号。研究中所有“可行”均指源码推导或配置候选，没有运行验收结果。

## 补充：Claude Code 可以使用现有鉴权机制前置 MemoryProxy

以下组合在源码上具备现有配置基础，无需为它实现原生 Codex 的双凭据；尚未串联运行验证：

```text
Claude Code（Anthropic Messages；入站 sk-mem 记忆身份）
  → MemoryProxy /claude-code/default（记忆注入与归档）
  → cc-Switch /v1/messages（选中 Claude 页真实供应商）
  → API Key 供应商，或 Claude 页 Codex OAuth 供应商
```

证据与条件：

- MemoryProxy [anthropic.ts:276](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/pipeline/runners/anthropic.ts:276) 从 `x-api-key` 或 Bearer 读取入站 key，并在 [anthropic.ts:664](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/pipeline/runners/anthropic.ts:664) 作为记忆的 `callerUserKey`。Claude Code 传 `sk-mem` 可以满足既有记忆身份机制。
- [resolve-target-full.ts:59](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/pipeline/stages/resolve-target-full.ts:59) 支持按 `claude-code` 配置上游 URL 和空 `apiKey`；该 agent entry 会切断全局服务端 key 回退。[anthropic.ts:387](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/pipeline/runners/anthropic.ts:387) 对空出站 key 保留客户端认证，因而入站 `sk-mem` 可先到可信的本机路由，再由后者替换。
- [guard-adapter.ts:430](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/guard-adapter.ts:430) 的 `joinUrl` 能把 `http://<router-host>:15721/v1` 与 `/messages` 拼成 `/v1/messages`。需要使用实际的 WSL/Windows/Docker 可达地址。
- cc-Switch [forwarder.rs:2471](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/forwarder.rs#L2471-L2491) 会在非原生 Codex Official 分支，用所选供应商的认证头替换入站 `Authorization`/`x-api-key`，而非把 `sk-mem` 发给真正的 LLM。对于 Claude 页 Codex OAuth，凭据由 [forwarder.rs:2155](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/forwarder.rs#L2155-L2206) 动态取得。
- MemoryProxy 可通过自定义 YAML 设置 `upstream.agents.claude-code.url` 为 cc-Switch 路由地址、`apiKey: ""`，同时关闭会改目标/model/key 的其他分支；cc-Switch Claude 页选择实际模型供应商，避免把 MemoryProxy 再设成该应用的出站目标形成循环。
- 默认 [start-proxy.sh:20](/home/v-lushuai/Codes/TencentDB-Agent-Memory/deploy/global-images/start-proxy.sh:20) 强制 `PROXY_UPSTREAM_API_KEY` 非空，且 [start-proxy.sh:46](/home/v-lushuai/Codes/TencentDB-Agent-Memory/deploy/global-images/start-proxy.sh:46) 会生成并覆盖 YAML。因此此方案需要自定义配置/启动方式，或随后显式适配启动脚本；**不需要因此修改业务鉴权代码，但不能宣称现有 `.env` 脚本原封可用**。

Claude Code 与 Codex 可以连接同一 Memory Core，并选择同一 Team/Agent 来共享知识和记忆；是否共享应由业务身份和资产归属决定，而不是两边恰好使用同一个模型。这里 Claude 页 Codex OAuth 会把 Anthropic 请求转换到 Responses，并由路由注入 managed token；原生 Codex Official 则需要保留客户端 OAuth。两者不能共用一个未经区分的认证配置说明。

## 补充：Codex 与 Claude Code 同时使用不同供应商

本节只读追踪本地 cc-Switch `de0a235ad6389ce4afa333b472cdee615a3f3987`，该仓库仍无工作区改动。用户的目标是同一时刻 Codex 使用 GPT 官方订阅、Claude Code 使用 GLM；这符合 cc-Switch 已有的按应用类型隔离机制，不需要为了并发运行而切换一个全局供应商。

### 同一个端口，通过请求入口确定应用

| cc-Switch HTTP 入口 | Handler 固定的应用类型 | 本方案对应的供应商 |
|---|---|---|
| `POST /v1/messages` | `AppType::Claude`，字符串 `claude` | Claude 页选中的 GLM |
| `POST /v1/responses` | `AppType::Codex`，字符串 `codex` | Codex 页选中的 OpenAI Official |

入口注册见 [server.rs:305](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/server.rs#L305-L306)、[server.rs:339](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/server.rs#L339-L349)；固定应用类型见 [handlers.rs:245](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/handlers.rs#L245-L250)、[handlers.rs:1011](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/handlers.rs#L1011-L1016)。路由并不根据 `model` 名或 Bearer 内容猜测应用。

所以 MemoryProxy 可让两条分支共用同一个 router origin 和 `/v1` base，但保留各自协议和 endpoint；不需要另增 cc-Switch 应用标识 header：

```text
Codex → MemoryProxy /codex/default → router /v1/responses → GPT 官方订阅
Claude Code → MemoryProxy /claude-code/default → router /v1/messages → GLM
```

这里 MemoryProxy 的客户端来源名 `claude-code` 与 cc-Switch 内部应用名 `claude` 不同，HTTP endpoint 完成二者的对应关系，不能把 `/claude-code/default` 当作 cc-Switch 接口。两条路径的上游地址应是实际可达的 `http://<router-host>:15721/v1`，例如容器通过宿主可达地址访问 Windows 路由服务；不能直接假定容器的 `127.0.0.1` 是宿主。

### 目标选择不是一个共享的 current_provider

当前请求沿以下层次选择供应商，所有读取均带当前应用类型：

1. 代理模式下读取该应用的 `proxy_route`；没有有效代理模式或目标时回落到该应用的直连供应商。见 [mode/current.rs:65](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/mode/current.rs#L65-L97)。
2. 直连选择优先取设备本地 settings 中的 `current_provider_claude`、`current_provider_codex` 等独立字段；验证该供应商在当前应用内存在后使用它。见 [settings.rs:1357](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/settings.rs#L1357-L1365)、[settings.rs:1402](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/settings.rs#L1402-L1427)。
3. 本地指针缺失或失效时，才回落到数据库中这个 `app_type` 的 `is_current`。见 [providers.rs:197](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/database/dao/providers.rs#L197-L205)。

数据库 `providers` 的主键是 `(id, app_type)`，不是单独的 `id`，见 [schema.rs:27](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/database/schema.rs#L27-L42)；设置当前项时也只更新指定 `app_type`，见 [providers.rs:475](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/database/dao/providers.rs#L475-L489)。模式状态则保存在共享状态文件的 `apps[app]` 下，每个 app 有独立 `proxy_route`，见 [mode/state.rs:155](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/mode/state.rs#L155-L170)、[mode/state.rs:365](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/mode/state.rs#L365-L377)。这也是不能只读私有数据库 `is_current` 就声称知道实际路由目标的原因。

### 并发请求、热切换和模型映射

- 每个请求在 [handler_context.rs:128](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/handler_context.rs#L128-L203) 取得该应用的供应商及候选列表，并保存供本次转发使用的副本。切换 Claude 页目标后，后续 Claude 请求会看到新目标；Codex 的供应商不会因此改变，已经取得供应商快照并正在传输的请求也不会被切到新目标。涉及客户端模型目录或配置变化时，是否重启客户端仍按对应手册处理。
- 路由目标切换的内部命令显式接收 `app_type` 和 `provider_id`，见 [commands/proxy.rs:108](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/commands/proxy.rs#L108-L118)。这是说明隔离机制的源码证据，不是建议 MemoryProxy 调用的公共 HTTP 接口。
- 故障队列、应用级故障配置及熔断器 key 也按 app 隔离，熔断器 key 为 `app_type:provider_id`。见 [provider_router.rs:68](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/provider_router.rs#L68-L109)。官方 Codex 账号仍保持单一目标，不自动重试到第三方供应商。
- 普通路由先按应用选择供应商，再读取这家供应商自己的模型映射。见 [forwarder.rs:1459](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/forwarder.rs#L1459-L1475)、[model_mapper.rs:21](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/model_mapper.rs#L21-L54)、[model_mapper.rs:129](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/model_mapper.rs#L129-L154)。例如 Claude 的角色模型名可映射到 GLM；这不会改变 Codex 页的官方模型选择。
- 聚合模式的模型标识可以显式选择其他供应商，但解析和供应商读取仍受当前 app 限定。见 [handlers.rs:179](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/handlers.rs#L179-L204)、[mode/stack.rs:679](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/mode/stack.rs#L679-L698)。本次双客户端需求并不要求启用聚合。

### 集成边界：HTTP 数据面与状态观测

MemoryProxy 的首选集成契约是 cc-Switch 已有的 HTTP 请求转发接口。让 cc-Switch 自己管理按 app 隔离的供应商、模型、认证和切换；MemoryProxy 不建立另一套供应商指针，不跨语言 import Rust，不直接读取私有数据库或调用内部 Tauri 命令。`MEMORY_LLM_*` 继续配置记忆加工模型，不应自动跟随某个 coding-agent 的供应商。

可以使用 `GET /health` 观察路由服务健康，`GET /status` 查看整体计数和状态；注册见 [server.rs:302](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/server.rs#L302-L303)，实现见 [handlers.rs:62](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/handlers.rs#L62-L75)。但应特别区分：

- HTTP `/status` 直接复制 `state.status`；其中 `current_provider` 与 `current_provider_id` 由每次转发尝试写入，是全局最近尝试目标，见 [forwarder.rs:600](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/forwarder.rs#L600-L607)。并发运行时可由另一 app 覆盖，不能据此给 MemoryProxy 选择上游。
- 内部 `ProxyServer::get_status()` 另从按 app 的 HashMap 计算 `active_targets`，见 [server.rs:265](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/server.rs#L265-L282)。HTTP handler 没有调用该方法，不能把内部 Tauri 状态接口的丰富程度当成 HTTP `/status` 的保证。
- 即使内部 UI 的 `active_targets` 有值，也可能由热切换预先更新，不代表该目标已处理成功请求，见 [server.rs:287](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/server.rs#L287-L295)。因此状态观测不能代替逐请求日志及实际模型调用验证。

以下只是请求契约示意，不含真实凭据，且本轮没有执行：

```http
POST /v1/responses
Authorization: Bearer <Codex 客户端 OAuth>
chatgpt-account-id: <官方账号 workspace>
Content-Type: application/json

{"model":"<GPT 模型>","input":[...],"stream":true}
```

```http
POST /v1/messages
Authorization: Bearer <MemoryProxy 转发到本机路由的入站凭据>
Content-Type: application/json

{"model":"<Claude 角色别名或 GLM 模型>","messages":[...],"max_tokens":4096,"stream":true}
```

第一条由 cc-Switch 原生 Codex Official 分支校验并透传客户端 OAuth；第二条由 Claude 分支把入站认证替换成所选 GLM 供应商的 Key。前者仍需要本文前述 MemoryProxy 双凭据适配；按 app 分开路由并不会自动解决这个记忆身份问题。

### 辅助端点不能全部只按路径识别应用

`POST /v1/responses/compact` 在 [handlers.rs:1240](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/handlers.rs#L1240-L1244) 固定使用 `AppType::Codex`。但 `GET /v1/models` 由两个客户端共用：[handlers.rs:134](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/handlers.rs#L132-L145) 检查请求是否有 `anthropic-version` header；没有该 header 时，query 含 `limit` 且不含 `client_version` 也判为 Claude 模型发现。否则进入 Codex 目录分支。

Claude 模型发现返回已发布的 Claude 聚合模型，见 [handlers.rs:147](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/handlers.rs#L147-L168)；Codex 分支读取 live `config.toml` 当前指向且归 cc-Switch 所有的模型目录文件，没有有效指向时返回 `models: []`，见 [handlers.rs:91](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/handlers.rs#L91-L125)。这不是根据当前供应商实时向上游拉取模型的接口。前置 MemoryProxy 应保留相关 query 和协议 header，并单独验证目录及 compact 请求；不能把两条主推理请求的固定 app 结论扩大为所有辅助请求都天然可识别。

### app_type、业务 Agent 与多个终端的区别

cc-Switch 的 `claude` / `codex` 是 coding-agent **应用类型**，不是 Memory Core 的业务 Agent ID，也不是每个终端或会话的 ID。因此：

- 同时运行 Codex→GPT 和 Claude Code→GLM，已有按 app 隔离满足目标。
- 两个 Claude Code 终端默认共享 `claude` 当前供应商，切换后会一起影响各自后续普通路由请求；两个 Codex 终端同理。
- 同一应用类型的多个会话若要同时用不同供应商，需要明确采用聚合模型标识、不同路由实例或另行设计会话级隔离，不能把现有 app 级指针说成会话级指针。
- MemoryProxy 的 `upstream.agents["claude-code"]` / `["codex"]` 也是按客户端来源覆盖上游的配置项，与 Memory Core 选中的业务 Agent 不是同一概念。业务记忆是否共享，仍由 user/team/agent/task 及资产权限决定。

本章追加前再次对研究文档路径执行 GitNexus `impact`，因文档尚未进入符号索引返回 UNKNOWN；随后核对它是本任务此前创建的未跟踪研究文档，编辑范围仅为本文件的追加内容。没有将 UNKNOWN 作为现有业务代码可安全修改的证据。

## MemoryProxy 的可实施接入范围与独立记忆模型

用户已明确本轮继续补齐方案与 HTML，不实施代码或变更客户端配置。推荐在 MemoryProxy 增加可选的 `cc-switch` 编码上游模式，使用 cc-Switch 已有 HTTP 数据面；当前 direct 模式保持兼容。拟新增配置字段和 adapter 尚未存在，不能把 HTML 草图当作现有配置说明。

### 编码请求与记忆加工分别配置

- 编码请求只由 coding 上游 adapter 确定 cc-Switch 协议入口：Codex → `/v1/responses`，Claude Code → `/v1/messages`。供应商与模型映射由 cc-Switch 按应用决定；MemoryProxy 不读取私有数据库，也不根据 HTTP `/status` 的最近供应商重新选路。
- [start-memory-core.sh](/home/v-lushuai/Codes/TencentDB-Agent-Memory/deploy/global-images/start-memory-core.sh:182) 已将 `MEMORY_LLM_BASE_URL / API_KEY / MODEL` 写入 Core 的独立 `llm` 配置。
- [start-memory-hub.sh](/home/v-lushuai/Codes/TencentDB-Agent-Memory/deploy/global-images/start-memory-hub.sh:103) 明确启用 Knowledge 的 `LLM_MODE=custom`，通过 `MEMORY_LLM_*` 注入协议、端点、Key 和模型。切换 cc-Switch 的 Claude / Codex 供应商不应修改这些配置。
- Proxy 自身的任务草稿需要补齐：[resolveTaskDraftConfig](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/routes/session-task.ts:262) 按字段选取 `MEMORY_LLM_*` 优先值，缺字段会回落到主推理上游和客户端当次 model。任务命令最终复用此解析链，见 [create-task.ts](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/mem-command/commands/create-task.ts:164)。因此“Core / Knowledge 独立”不能推出“Proxy 内部所有记忆辅助调用已独立”。
- [start-proxy.sh](/home/v-lushuai/Codes/TencentDB-Agent-Memory/deploy/global-images/start-proxy.sh:153) 当前 `docker run` 没有注入 `MEMORY_LLM_*`，脚本也只强制校验 `PROXY_UPSTREAM_*`。新增模式需完整注入并校验独立记忆组；Proxy 草稿 LLM 缺配置时返回明确配置错误，不借用编码 OAuth / GLM Key。任务实体是否沿用现有上层降级行为，应保持既有业务契约，不能借本次路由接入暗改。

### 不能遗漏的现有请求处理

1. [Codex runner](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/pipeline/runners/codex.ts:338) 在当前请求链执行 `applyMainModel`；Anthropic runner 还可进入 pool / model gate / cost-guard。cc-Switch 模式应在这些编码模型和路由决策之前分支，并阻止 instance override 等再次覆盖 model、URL、Key；记忆注入、会话初始化和归档仍保留。
2. [modelsHandler.ts](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryProxy/src/modelsHandler.ts:45) 当前校验记忆 Key 后，从本地价格表和模型池构建目录。接入 cc-Switch 时还需处理模型发现，不能仅改两条主推理出口；Codex 的记忆鉴权也要统一支持独立 Header。
3. `/v1/models` 需保留 query 与 `anthropic-version` 等判定信息；cc-Switch 返回的是发布目录，不一定是实时上游目录。仅启动后方服务、前方独立 profile 下的目录行为须单独验证。不能把空目录误当成真实模型调用不可用，也不能把目录展示当成订阅权限证据。
4. 客户端有效入口必须保留在 MemoryProxy。cc-Switch 只提供后方 HTTP 路由，或以可验证的 profile / 启动参数隔离接管；重启和切换后读回配置，确认未绕过记忆层。

### 本轮追加的验收场景

同时发起 Codex 官方订阅请求与 Claude Code GLM 请求，分别核对应用类型、供应商和真实结果；切换 Claude 目标，观察已开始请求的快照和后续请求，同时确认 Codex 目标与独立记忆组不变。并行触发 Core / Knowledge 记忆加工与 Proxy 任务草稿，核对它们调用独立记忆端点。验证模型目录、compact、工具调用、SSE、登录刷新，以及 router 不可达时的明确失败，避免任何未配置的编码模型或凭据兜底。

本节与 HTML 均为源码推导和待实现设计，没有启动业务服务、修改认证实现、调用真实模型或确认运行验收成功。
