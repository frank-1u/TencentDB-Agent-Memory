# cc-Switch 聚合模式承接记忆加工 API：源码调研

日期：2026-10-25。主要依据：只读本地 `/home/v-lushuai/Codes/cc-switch`，HEAD `de0a235ad6389ce4afa333b472cdee615a3f3987`；核对时工作区 clean。本轮仅新增本文，没有修改 cc-Switch、客户端配置或服务，没有读取真实凭据或调用模型。

## 结论与适用范围

**记忆加工 LLM 请求可以考虑经过 cc-Switch 聚合路由，但不是把 `MEMORY_LLM_BASE_URL` 改成其端口就自动获得独立供应商。**必须同时匹配 HTTP 输入协议、所属应用的聚合 namespace、完整的聚合模型 ID，以及后台调用者可用的供应商认证方式。

可行的优先候选是：记忆加工使用 cc-Switch 中某个 **API Key 供应商的显式聚合模型 ID**，避免跟随 coding app 的默认供应商。若 MemoryCore / Knowledge 实际发送 OpenAI Chat Completions，就使用 cc-Switch 的 `/v1/chat/completions`，模型填真实登记的 `ccs-<key>/<model>`；该接口使用 **Codex 聚合 namespace**，不是一个独立的 memory namespace。所选上游必须原生支持 Chat Completions。

这允许如下并发候选：

```text
Codex 主推理（无聚合前缀） → /v1/responses → Codex 聚合默认 OpenAI Official
Claude Code 主推理        → /v1/messages  → Claude 默认 GLM
记忆加工 Chat 请求        → /v1/chat/completions，ccs-<memory-provider-key>/<model>
                         → Codex 聚合中的指定 API Key 供应商
```

`MEMORY_LLM_*` 的职责仍独立；只有请求运输和供应商凭据管理交给 cc-Switch。记忆加工不是 coding app 主推理的隐式子请求，不应复用其 OAuth、默认模型或故障队列。本轮已核对 MemoryCore / Knowledge 的实际文本 LLM 调用使用 Chat Completions，细节见下节；静态源码结论仍不等于运行验证。

## 当前 MemoryCore / Knowledge 的协议证据

本仓当前 HEAD 为 `6a4bb55ea54834d8265ce52024d3e3c75db30d38`。本节引用当前本地代码，工作区文件是事实依据：

- Core 的 [StandaloneLLMRunner.run:340](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryCore/src/adapters/standalone/llm-runner.ts:340) 使用 `provider.chat(this.model)`；offload 的 [callLlm:58](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryCore/src/offload/local-llm/llm-caller.ts:58) 创建 OpenAI-compatible provider，在 [llm-caller.ts:66](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryCore/src/offload/local-llm/llm-caller.ts:66) 同样调用 `.chat(config.model)`。因此统一入口应是 Chat Completions，不能把 base URL 改到 `/responses` 假称协议已切换。
- Knowledge 的 [ingest-v2/llm.ts:114](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/engines/wiki/ingest-v2/llm.ts:114) 按 protocol 创建 provider；当前共同方案采用 `openai`，在 [llm.ts:142](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/engines/wiki/ingest-v2/llm.ts:142) 使用 `.chat(config.model)`。Knowledge 单独有 Anthropic 配置能力，不代表 Core 共同配置也可以直接切成 Anthropic。
- Core 应保持 `llm.provider=openai` 或相应默认。当前 [llm-provider-resolver.ts:37](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryCore/src/adapters/standalone/llm-provider-resolver.ts:37) 对非 `proxy` 原样保留；若改成 `proxy`，会在 [llm-provider-resolver.ts:75](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryCore/src/adapters/standalone/llm-provider-resolver.ts:75) 拼接 `/proxy/<instanceId>/v1` 并可能替换为记忆系统用户 key。这不是 cc-Switch 的接口契约。
- Knowledge 保持 `LLM_MODE=custom`。此处 `custom` 表示使用给定端点，并不表示某家供应商或 MemoryProxy；其配置注释及实例 binding 的覆盖优先级见 [config.ts:20](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryKnowledge/src/config.ts:20)。部署脚本 [start-memory-hub.sh:109](/home/v-lushuai/Codes/TencentDB-Agent-Memory/deploy/global-images/start-memory-hub.sh:109) 已为该模式传入 `MEMORY_LLM_*`，正式配置前还须检查已有实例绑定是否覆盖全局设置。
- 远程 Embedding 由独立服务调用 `${baseUrl}/embeddings`，见 [embedding.ts:520](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryCore/src/core/store/embedding.ts:520)；工厂同时支持远程 OpenAI-compatible 和本地模型，见 [embedding.ts:620](/home/v-lushuai/Codes/TencentDB-Agent-Memory/MemoryCore/src/core/store/embedding.ts:620)。cc-Switch 当前没有相应入口，所以 **Embedding 不在本次 Chat 聚合范围**，应保留本地或另配远程 embedding 服务。

据此主推应明确命名为“**记忆 API 聚合配置候选，待真请求验证**”。它与整套 coding 接入的完成状态分开：文本记忆加工存在配置组合基础；原生 Codex 经 MemoryProxy 保留官方 OAuth 的双凭据支持仍需代码适配。

## 一手文档与源码定位

- [官方聚合模式手册](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/docs/user-manual/zh/4-proxy/4.6-aggregation.md)：区分默认供应商、`ccs-` 模型选择、无故障转移及客户端重启要求。
- [聚合模式攻略](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/docs/guides/aggregation-mode-guide-zh.md)：产品侧的配置和模型发布步骤。
- 下文链接都固定在上述 commit，避免把不同版本的命名和模式行为混用。源码比手册更精确地揭示了 Chat Completions 入口、前缀拼错时的普通模型分支，以及模型列表变化时的边界。

## 精确的模型 ID 与供应商选择

| cc-Switch 应用 | 聚合模型 ID 格式 | 示例，key 仅作示意 |
|---|---|---|
| Codex | `ccs-<key>/<model>` | `ccs-memory-api/deepseek-chat` |
| Claude Code | `ccs-claude-<key>--<model>`；可带客户端 1M 标记 | `ccs-claude-memory-api--deepseek-chat` |

定义与编码见 [stack.rs:33](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/mode/stack.rs#L33-L39)、[stack.rs:118](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/mode/stack.rs#L118-L125)。`key` 是注册到具体供应商 ID 的稳定标识，不等于供应商显示名称，也不能在文档里随意猜一个后直接使用。

[allocate_key:61](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/mode/stack.rs#L61-L95) 首次从图标/名称派生 key、处理冲突；已有 key 则保留，移除或删除的 key 不回收。两家供应商提供同名模型时，完整 ID 中的 key 负责区分。供应商显示名称变化不需要改变旧 key；删除重建同名卡片不能保证旧 ID 继续有效。

普通不带前缀的模型请求会走该应用的默认供应商，见 [聚合手册:76](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/docs/user-manual/zh/4-proxy/4.6-aggregation.md#L76-L88)。后台记忆请求若想固定供应商，应复制实际发布的完整 ID，并在配置校验时验证其 namespace 与格式。

### 明确报错与意外走默认的边界

[decode:163](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/mode/stack.rs#L163-L201) 与 [resolve:679](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/mode/stack.rs#L679-L727) 有以下差别：

| 配置/状态 | 当前行为 |
|---|---|
| 当前应用能识别的完整聚合 ID，但 key 不存在 | 返回聚合 ID 不存在，不走默认 |
| 完整聚合 ID，当前应用退出聚合模式 | 返回 `StackOff`，不走默认 |
| 对应供应商从名单移除或删除 | 返回 `Removed` / `Deleted`，不走默认 |
| 当前应用能识别的前缀但分隔后 key/model 为空 | 返回 malformed/unknown 错误 |
| Codex 请求的 `ccs-foo` 没有 `/` | **被当成普通模型**，继续当前默认路由；不是聚合错误 |
| Claude 请求误用 Codex 的 `ccs-foo/model` | **没有 Claude 的保留前缀**，被当成普通模型；可能走 Claude 默认 |
| 普通未加前缀的模型名 | 按设计走当前默认，默认切换会影响它 |

因此“聚合不故障转移”和“所有错误模型 ID 都不会走默认”不是同一承诺。集成端应严格验证完整 ID 的应用前缀和分隔符，不能只关闭 failover 就声称模型来源固定。源码对真正的聚合 miss 会在 [handlers.rs:179](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/handlers.rs#L179-L212) 返回 HTTP 400。

## 三种输入协议都可以解析聚合，但不能任意互换

| HTTP 输入接口 | 所属应用 namespace | 聚合解析 | 对记忆加工的含义 |
|---|---|---|---|
| `/v1/chat/completions` | Codex | 支持 `ccs-<key>/<model>` | 可作为原生 OpenAI Chat SDK 的候选入口；目标上游仍需支持 Chat |
| `/v1/responses` | Codex | 支持 `ccs-<key>/<model>` | 适合 Responses caller，可按供应商配置做 Responses→Chat/Anthropic 转换 |
| `/v1/messages` | Claude | 支持 `ccs-claude-<key>--<model>` | 适合 Anthropic caller；可使用 Claude 供应商的格式转换与 managed OAuth |

注册见 [server.rs:305](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/server.rs#L305-L328)、[server.rs:339](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/server.rs#L339-L349)。Chat handler 在 [handlers.rs:916](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/handlers.rs#L916-L948) 显式以 `AppType::Codex` 调用聚合解析；Responses 和 Messages 也分别在 [handlers.rs:1040](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/handlers.rs#L1040-L1067)、[handlers.rs:307](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/handlers.rs#L307-L340) 调用相同解析器。

**Chat 输入并不自动享有 Responses 输入的协议转换。**[codex.rs:76](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/providers/codex.rs#L76-L88)、[codex.rs:202](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/providers/codex.rs#L202-L210) 将转换条件限制在 Responses/compact endpoints；没有把普通 `/chat/completions` 输入转换到 Responses 或 Anthropic 的通用桥。因此不能把只支持 Anthropic 或 Responses 的供应商配置加入聚合后，就断言 Chat 类型记忆调用也可用。请求及返回格式、JSON/structured output、工具调用、非流式响应，都要按实际 memory caller 验收。

上述路由不是完整 OpenAI API 服务；本轮路由表没有发现 `/embeddings` 入口。若实际记忆服务另需 embedding API，必须单独确认路径与能力，不能仅依据统一 base URL 推断支持。

## 静态本机占位 Key 能换成什么凭据

### API Key 供应商：候选可行

后台 caller 可带一个不含真实厂商凭据的本机占位 Bearer。聚合解析先选定供应商；[forwarder.rs:2466](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/forwarder.rs#L2466-L2488) 用供应商 adapter 提供的认证头替换入站认证。所选卡片必须保存可用的真实上游 API Key，输入协议也必须匹配上游。

例如以下是配置形态示意，不是已部署配置，`memory-api` 也不是保证存在的注册 key：

```text
MEMORY_LLM_BASE_URL=http://<router-host>:15721/v1
MEMORY_LLM_API_KEY=local-memory-caller
MEMORY_LLM_MODEL=ccs-memory-api/deepseek-chat
MEMORY_LLM_PROTOCOL=openai
```

仅当实际 caller 使用 Chat Completions、Codex 聚合已生效且包含对应 API Key 卡片、目标 Chat endpoint 可用时，这个形态才是可验证候选。后端直接调用 cc-Switch，不应再指向 MemoryProxy 的用户记忆入口，否则会把内部摘要请求重新当成 coding 会话甚至形成循环。

### 原生 Codex OpenAI Official：占位 Key 不可替代 OAuth

[controller.rs:1224](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/mode/controller.rs#L1224-L1257) 禁止官方账号成为普通聚合成员；Codex OpenAI Official 只能作默认供应商。[forwarder.rs:1413](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/forwarder.rs#L1413-L1459) 要求客户端官方凭据，原样透传 OAuth，而非由路由把占位 Key 换成订阅 token。把后台 Chat caller 指向该默认目标既不提供所需 OAuth，也不能假定官方 Codex backend 支持 Chat 输入。因此这不是现成可用的记忆后台订阅路线。

### Claude 页 Codex OAuth：不同机制，需 Anthropic 输入

Claude 页的 `Codex OAuth` 供应商预设是 `category=third_party`、`providerType=codex_oauth`，见 [claudeProviderPresets.ts:2651](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src/config/claudeProviderPresets.ts#L2651-L2680)。它可按 Claude 聚合成员的检查规则参与；[ClaudeAdapter:780](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/providers/claude.rs#L780-L785) 选择 managed 策略，再由 [forwarder.rs:2155](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/forwarder.rs#L2155-L2206) 动态取有效 token。

因此后台 Anthropic Messages caller 配 Claude namespace 的显式聚合 ID，在认证机制上有候选基础；它走的是 **Claude→Responses 转换和 managed token**，不是原生 Codex Official 的客户端 OAuth 透传。不要将两者写成一个可互换的供应商入口，也不要把订阅计量或账号条款推导成已验证结论。只需统一管理记忆加工 API 时，优先选用独立 API Key 供应商，减少对订阅与协议转换的依赖。

## 聚合模式与后台 HTTP 服务的生命周期耦合

后台 caller 不需要真的启动 Codex/Claude Code 进程；聚合解析在 HTTP handler 内完成。但当前实现检查该应用 `mode==Proxy && stack.enabled`，见 [state.rs:405](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/mode/state.rs#L405-L410)。仅“启动本地 HTTP 服务”不等于启用后台聚合 registry。

产品的聚合启用流程调用 [controller.rs:796](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/mode/controller.rs#L796-L825)，写 `stack.enabled`，并通过 `write_proxy` 以 `attached=true` 投影客户端配置。没有独立的公共 HTTP API 可只为 memory 建立一个不影响 coding 应用的聚合 namespace。因此：

- 可以依赖已经生效的 coding 应用聚合状态，让后台直接发送 HTTP 请求。
- 开启/退出聚合、修改成员仍与这个 coding 应用的 live 配置及模型目录关联。前置 MemoryProxy 方案必须继续使用明确的 profile/运行参数隔离入口，并验证 cc-Switch 重写 live 配置后不会绕过 MemoryProxy。
- 不能一面宣称“cc-Switch 完全不接管 coding 配置”，一面假设其既有聚合功能可独立启用；后者需要现有产品操作或新增独立服务契约。
- 如果要求后台记忆持续工作且完全不受 coding app 的路由/聚合模式变化影响，需要明确的独立后台 namespace/registry 或等价隔离设计，属于新增能力。不能靠直接改私有状态文件或读私有数据库维持这种契约。

## 切换、重命名与删除的影响

| 操作 | 对显式、正确格式的 memory 聚合 ID 的影响 |
|---|---|
| coding app 默认供应商切换，仍为聚合模式且目标成员保留 | 完整 ID 仍选原 key 对应的供应商，不跟随默认 |
| 供应商显示名称变化 | 既有 key 不变，ID 不应因此改变 |
| 供应商 Key、endpoint、模型实际服务发生变化 | 仍是同一 provider ID，但每次请求读取当前卡片，后台会使用修改后的配置 |
| 把供应商从聚合名单移除/删除供应商 | 直接聚合错误，不走默认 |
| 路由↔聚合切换，或退出代理模式 | 非聚合状态下完整聚合 ID 报 `StackOff`；重新启用且成员/key保留后才可恢复 |
| 从卡片发布列表移除或重命名某个模型，但保留供应商成员 | **不会保证在本机拒绝**；当前 resolver 保留旧 ID 携带的模型名发给原供应商，由上游决定是否接受 |
| memory 配置把 ID 改成没有被当前 app 识别为聚合的普通模型名 | 走默认，可能随 coding 默认变化 |

模型删除边界来自 [stack.rs:705](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/mode/stack.rs#L705-L727)：Claude 会优先匹配当前发布模型，匹配不到则保留 ID 中旧模型；Codex 直接使用 ID 中的模型名，并不执行“模型仍在卡片目录里”的白名单校验。后台需要自己维护固定 ID 及可接受模型合同，不能把 UI 删除模型当作访问撤销。

真正的聚合请求会强制关闭 failover，并仅携带所选 provider；见 [handler_context.rs:128](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/handler_context.rs#L128-L147)、[handler_context.rs:267](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/handler_context.rs#L267-L293)。[forwarder.rs:509](https://github.com/farion1231/cc-switch/blob/de0a235ad6389ce4afa333b472cdee615a3f3987/src-tauri/src/proxy/forwarder.rs#L509-L512) 还保留 resolver 已确定的模型，不再套默认角色映射。SDK 自己是否重试由实际 memory caller 配置决定；这些机制不等于所有类型的重复请求都被禁止。

## 方案矩阵

“现有机制可组合”均为源码层候选，尚未真实联调。

| 方案 | 判断 | 原因与边界 |
|---|---|---|
| Memory API Key 直连，coding 推理仍经 cc-Switch | 现有基线，适合优先保持后台独立 | 供应商切换不由 cc-Switch 统一管理；不满足所有推理都走 cc-Switch 的完整目标 |
| Memory Chat → cc-Switch Codex 聚合 → 显式 API Key 成员 | **现有机制可组合，优先验证** | alias 固定供应商；目标须支持原生 Chat；共享 Codex 聚合生命周期 |
| Memory Anthropic → cc-Switch Claude 聚合 → 显式 API Key 成员 | **现有机制可组合** | 需实际 caller 支持 Anthropic；共享 Claude 聚合生命周期 |
| Memory Responses → cc-Switch Codex 聚合 → 显式成员 | **协议机制可组合** | 需 actual caller 支持 Responses；不能把 Chat SDK 的 base URL 改写当成协议迁移 |
| Memory Anthropic → Claude 聚合 Codex OAuth 成员 | 可进一步评估，非默认推荐 | 走 managed token 与协议转换；与原生 Codex Official 机制不同，仍需端到端验证 |
| Memory 静态占位 Key → Codex OpenAI Official 默认 | **现有认证不满足，不推荐** | 缺客户端 OAuth，且 Chat 输入不对应官方 Codex Responses 协议 |
| Memory 普通模型名 → coding app 默认供应商 | 配置最少，**不推荐长期使用** | 默认切换会改变后台模型/计费；可能落到需要 OAuth 的官方目标 |
| Memory → MemoryProxy coding入口 → cc-Switch | **不推荐** | 后台请求再次经过记忆注入/归档与初始化，可能自我记忆、循环或错归任务 |
| cc-Switch 新增独立 memory namespace/后台聚合 registry | **需产品/源码改造** | 可把后台生命周期与 coding app 解耦，但不是当前现成配置，也不能只加一条模型 alias 就声称完成 |

## 最小验证与配置门槛

1. 选定实际 memory 协议，核实模型 endpoint、非流式返回、结构化输出等真实需求；先用独立 API Key 成员验证，不把 SDK 名称等同于输入协议。
2. 取得真正发布/登记的完整模型 ID；校验 endpoint 所属 app、前缀和分隔符。避免普通模型名误入默认。
3. 同时发起三类真实请求：Codex 官方、Claude GLM、memory 显式聚合成员；对照每条请求的实际 provider、上游模型和用量归属，客户端显示名不是证明。
4. 切换默认供应商后确认 memory 仍命中固定成员；退出聚合、移除成员、未知key应按预期失败，不能静默切其他供应商。
5. 验证模型改名/移除后旧 ID 仍可能发往旧模型的边界，确定后台自己的变更与回滚程序。
6. 验证前置 MemoryProxy 与 cc-Switch live 配置重写的共存方式、服务退出/重启、Windows/WSL/Docker 网络可达性。不要把静态方案写成部署证明。

创建本文前执行了 GitNexus `impact`：新文档未索引，返回 UNKNOWN；随后确认目标文件尚不存在且本轮只创建该文档，没有把 UNKNOWN 当作业务符号安全修改的证明。
