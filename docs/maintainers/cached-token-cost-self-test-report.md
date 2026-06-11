# Cached Token 成本跟踪自测报告

## 1. 文档说明

本文档用于记录本次 Cached Token 成本跟踪改动的人工自测方案，面向测试人员或研发自测执行，不是单元测试清单。

本次改动主要覆盖以下能力：

- OpenAI-compatible usage 中 Cached Prompt Token 的解析
- 支持 cached input 的差异化成本计算
- legacy alias pricing 与 `cost.rates` 的优先级统一
- provider 标识统一为 `<type>.<alias>`
- Gateway 与 WebSocket 路径下按 turn 聚合 token 与 cost
- 成本记录持久化与汇总查询正确性

## 2. 变更范围

涉及模块：

- `crates/zeroclaw-providers/src/compatible.rs`
- `crates/zeroclaw-runtime/src/agent/cost.rs`
- `crates/zeroclaw-runtime/src/agent/loop_.rs`
- `crates/zeroclaw-runtime/src/agent/agent.rs`
- `crates/zeroclaw-config/src/cost/types.rs`
- `crates/zeroclaw-config/src/cost/tracker.rs`
- `crates/zeroclaw-gateway/src/lib.rs`
- `crates/zeroclaw-gateway/src/ws.rs`
- `crates/zeroclaw-channels/src/orchestrator/mod.rs`

## 3. 测试环境

| 项目 | 建议配置 |
|---|---|
| 分支 | 包含本次改动的功能分支 |
| 运行环境 | 本地开发环境或独立测试环境 |
| 成本跟踪 | `cost.enabled = true` |
| 定价配置 | 至少配置一组 `cost.rates.providers.models` |
| Provider 覆盖 | 至少准备一个会返回 usage 的 OpenAI-compatible provider |
| Cached usage 覆盖 | 一组返回 `prompt_tokens_details.cached_tokens`；一组返回 `prompt_cache_hit_tokens` |
| 多 alias 覆盖 | 同一 provider type 下至少配置两个 alias，例如 `deepseek.work` 和 `deepseek.personal` |
| 验证输出 | WebSocket done 帧、gateway 响应、日志、`state/costs.jsonl`、成本汇总接口 |

## 4. 核查重点

| 核查项 | 验证重点 |
|---|---|
| usage 解析 | `input_tokens`、`output_tokens`、`cached_input_tokens` 是否归一化正确 |
| 成本计算 | cached input 是否按缓存价格或正确回退价格计费 |
| 定价优先级 | 同时存在时是否由 `cost.rates` 覆盖 legacy pricing |
| provider 标识 | 计费 lookup 是否在需要的路径上使用 `<type>.<alias>` |
| turn 聚合 | 单轮多次 LLM 调用时是否累计为一个正确总值 |
| 落盘结果 | `costs.jsonl` 是否包含完整且正确的字段 |
| 并发隔离 | 多 session 同时运行时是否互不污染 |
| 异常容错 | cached token 字段异常时是否不影响主流程 |

## 5. 人工自测用例

### 5.1 Usage 解析

| 用例 ID | 测试场景 | 前置条件 | 测试步骤 | 预期结果 | 执行结果 |
|---|---|---|---|---|---|
| TC-01 | 解析 OpenAI 风格 cached token | Provider 返回 `usage.prompt_tokens_details.cached_tokens` | 1. 发起一次普通对话请求。 2. 抓取 provider usage 返回。 3. 查看日志或落盘 usage。 | `cached_input_tokens` 来自 `prompt_tokens_details.cached_tokens`，且 input/output token 正常。 | 待执行 |
| TC-02 | 解析 DeepSeek 风格 cached token | Provider 返回 `usage.prompt_cache_hit_tokens` | 1. 发起一次对话请求。 2. 查看日志或落盘 usage。 | `cached_input_tokens` 来自 `prompt_cache_hit_tokens`。 | 待执行 |
| TC-03 | 同时存在两种 cached 字段时优先取显式 cache-hit 字段 | Provider 同时返回 `prompt_cache_hit_tokens` 和 `prompt_tokens_details.cached_tokens` | 1. 发起一次请求。 2. 对比归一化后的 cached token 与原始返回。 | `cached_input_tokens` 优先采用 `prompt_cache_hit_tokens`。 | 待执行 |
| TC-04 | cached token 为数字字符串 | Provider 返回如 `"2.5e2"` 的 cached token | 1. 使用 mock 或 provider 返回该场景。 2. 查看归一化 usage。 | 系统可容错解析为合法 cached token 数值。 | 待执行 |
| TC-05 | cached token 为非法值 | Provider 返回负数或非数字字符串 | 1. 发起请求。 2. 确认主流程成功。 3. 查看 usage 记录。 | 主流程成功；非法 cached token 被忽略；input/output usage 仍正常。 | 待执行 |
| TC-06 | usage 缺失 | Provider 完全不返回 `usage` | 1. 发起请求。 2. 查看日志和成本输出。 | 主流程正常，不因 usage 缺失产生异常。 | 待执行 |

### 5.2 成本计算

| 用例 ID | 测试场景 | 前置条件 | 测试步骤 | 预期结果 | 执行结果 |
|---|---|---|---|---|---|
| TC-07 | 无 cached token 的普通计费 | 配置 input/output 价格 | 1. 发起一次普通请求。 2. 查看 `costs.jsonl`。 | 成本 = input 费用 + output 费用。 | 待执行 |
| TC-08 | 有 cached input 的差异化计费 | 配置 input、output、cached-input 价格 | 1. 发起一次带 cached token 的请求。 2. 查看落盘记录。 | uncached input 按 input rate 计费，cached input 按 cached-input rate 计费，output 单独计费。 | 待执行 |
| TC-09 | cached 价格缺失时回退标准 input 价格 | 不配置 cached-input 价格，但配置 input 价格 | 1. 发起一次带 cached token 的请求。 2. 查看成本。 | cached input 不按 0 计费，而是回退到标准 input 价格。 | 待执行 |
| TC-10 | cached token 大于总 input token 的容错 | 使用 mock 或 provider 返回 cached input > input | 1. 发起请求。 2. 查看落盘记录和成本。 | cached input 被裁剪到不大于 input，总成本合法，无负数。 | 待执行 |
| TC-11 | 零 token 场景 | 构造 input/output 均为 0 的 usage | 1. 执行请求。 2. 查看落盘和汇总。 | 不产生无意义的成本记录。 | 待执行 |

### 5.3 定价来源与优先级

| 用例 ID | 测试场景 | 前置条件 | 测试步骤 | 预期结果 | 执行结果 |
|---|---|---|---|---|---|
| TC-12 | `cost.rates` 覆盖 legacy alias pricing | 同一模型同时配置 legacy `pricing` 与 `cost.rates` | 1. 发起请求。 2. 对比实际成本与两套价格。 | 最终成本以 `cost.rates` 为准。 | 待执行 |
| TC-13 | 仅 legacy alias pricing 时仍可计费 | 不配置对应 `cost.rates` | 1. 仅保留 alias pricing。 2. 发起请求。 | 成本仍可由 legacy alias pricing 正常计算。 | 待执行 |
| TC-14 | legacy pricing 在多 alias 间隔离 | 同一 provider type 下两个 alias 配置不同价格 | 1. 通过 alias A 发起请求。 2. 通过 alias B 发起请求。 3. 对比成本。 | 各 alias 使用各自价格，不互相串值。 | 待执行 |
| TC-15 | channels 路径使用 type-level pricing | 从 channel/orchestrator 路径触发对话 | 1. 通过 channel 入口发消息。 2. 查看成本记录。 | channels 路径能够正确命中 type-level pricing。 | 待执行 |

### 5.4 Provider 标识与定价 lookup

| 用例 ID | 测试场景 | 前置条件 | 测试步骤 | 预期结果 | 执行结果 |
|---|---|---|---|---|---|
| TC-16 | `process_message` 使用 composite provider ref | agent 配置为 `<type>.<alias>` | 1. 从 daemon 或 channel 入口触发消息。 2. 查看日志与成本输出。 | 定价 lookup 使用 `<type>.<alias>`，不会因 bare provider type 造成价格丢失。 | 待执行 |
| TC-17 | Gateway 非流式路径使用正确 provider ref | gateway 已开启成本跟踪 | 1. 从 gateway chat 或 webhook 入口发请求。 2. 查看 observer 和成本输出。 | provider 归因与成本 lookup 都命中实际的 `<type>.<alias>`。 | 待执行 |
| TC-18 | WebSocket 流式路径使用正确 provider ref | WebSocket 会话可用 | 1. 建立流式会话。 2. 发送一条消息。 3. 查看 done 帧和成本记录。 | done 帧和成本记录都使用正确 provider 标识，并命中正确价格。 | 待执行 |

### 5.5 Turn 级聚合

| 用例 ID | 测试场景 | 前置条件 | 测试步骤 | 预期结果 | 执行结果 |
|---|---|---|---|---|---|
| TC-19 | 单轮只有一次 LLM 调用 | 使用不会触发工具的简单问题 | 1. 发送简单请求。 2. 对比 done 帧和 `costs.jsonl`。 | turn 总量与该次单调用 usage/cost 一致。 | 待执行 |
| TC-20 | 单轮包含多次 LLM 调用 | 使用会触发工具的复杂问题 | 1. 发送触发工具的请求。 2. 统计每次 usage。 3. 查看最终 done 帧。 | 最终 turn 总量等于该轮所有 LLM 调用 usage 的累计值。 | 待执行 |
| TC-21 | 流式场景 usage 聚合 | 流式 provider 路径能返回 usage | 1. 执行一次流式对话。 2. 观察 usage 事件。 3. 查看最终 done 帧。 | done 帧中 usage/cost 只累计一次，不重复、不遗漏。 | 待执行 |
| TC-22 | 落盘失败时 turn 总量仍保留 | 人为制造 `costs.jsonl` 不可写或落盘失败 | 1. 发起一次请求。 2. 查看客户端返回。 3. 查看日志中的落盘告警。 | 即使持久化失败，turn 级 usage/cost 仍能在客户端响应路径中看到。 | 待执行 |
| TC-23 | 并发 session 之间隔离 | 同时存在两个 gateway 或 WS session | 1. 并发运行两个 session。 2. 对比两个 turn 的结果。 | 每个 session 只统计自己的 usage/cost，无串扰。 | 待执行 |

### 5.6 落盘与汇总校验

| 用例 ID | 测试场景 | 前置条件 | 测试步骤 | 预期结果 | 执行结果 |
|---|---|---|---|---|---|
| TC-24 | 落盘记录包含完整 token 字段 | 已开启成本跟踪 | 1. 执行一条带 usage 的请求。 2. 打开 `state/costs.jsonl`。 | 记录包含 model、input tokens、cached input tokens、output tokens、total tokens、cost、timestamp。 | 待执行 |
| TC-25 | billable input 可正确推导 | 至少产生一条带 cached input 的记录 | 1. 执行一次 cached token 请求。 2. 查看落盘记录。 | 从语义和数值上，billable input = `input_tokens - cached_input_tokens`。 | 待执行 |
| TC-26 | 汇总结果与落盘记录一致 | 可访问成本汇总接口或摘要输出 | 1. 连续执行多次请求。 2. 查询汇总。 3. 对比落盘记录。 | 总成本、按模型汇总、请求数与落盘记录一致。 | 待执行 |
| TC-27 | 按 agent 归因正确 | 开启 `track_per_agent`，并使用多个 agent | 1. 用不同 agent 发起请求。 2. 查询成本汇总。 | 落盘记录和汇总中的 agent 归因正确。 | 待执行 |

### 5.7 回归验证

| 用例 ID | 测试场景 | 前置条件 | 测试步骤 | 预期结果 | 执行结果 |
|---|---|---|---|---|---|
| TC-28 | 普通聊天能力不受影响 | 任意支持的 provider 可用 | 1. 发送普通聊天问题。 2. 查看响应。 | 聊天流程正常，无新增用户可见错误。 | 待执行 |
| TC-29 | 工具调用链路不受影响 | 至少启用一个工具 | 1. 发送会触发工具的请求。 2. 查看工具执行与最终回答。 | 工具调用、后续模型调用、最终回复均正常。 | 待执行 |
| TC-30 | 无定价配置时仍非致命 | 不配置价格或关闭定价 | 1. 发送请求。 2. 查看响应与日志。 | 主流程成功，即使最终成本为 0 或仅有缺失价格告警。 | 待执行 |
| TC-31 | 不返回 cached usage 的 provider 仍兼容 | 使用仅返回 prompt/completion token 的 provider | 1. 发送请求。 2. 查看落盘 usage。 | 成本跟踪仍正确，`cached_input_tokens` 为空或 0。 | 待执行 |

## 6. 执行说明

| 项目 | 说明 |
|---|---|
| 执行状态建议值 | 建议使用 `通过`、`失败`、`阻塞`、`未执行` |
| 证据建议 | 建议保留请求参数、provider usage 原始返回、done 帧、`costs.jsonl` 片段、汇总接口输出 |
| 高优先级用例 | TC-08、TC-12、TC-16、TC-20、TC-23、TC-27 |
| 环境注意事项 | 部分流中断相关测试可能受本地沙箱或 socket 权限影响，这类场景以人工运行时验证为准 |

## 7. 自动化验证补充

以下内容是评审过程中已观察到的自动化验证结果，可作为研发侧辅助证据，不替代人工自测结果。

| 命令 | 状态 | 说明 |
|---|---|---|
| `cargo test -p zeroclaw-runtime cost -- --nocapture` | 通过 | runtime crate 下与成本跟踪相关的单元与集成风格测试通过 |
| `cargo test -p zeroclaw-gateway ws -- --nocapture` | 通过 | gateway WebSocket 相关测试通过 |
| `cargo test -p zeroclaw-providers compatible -- --nocapture` | 部分通过 | 新增 usage 解析相关覆盖通过，但有一个既有 stream-abort 测试在沙箱内因 `Operation not permitted` 失败 |

## 8. 最终结论

| 项目 | 内容 |
|---|---|
| 人工自测结论 | 待执行 |
| 是否阻塞发布 | 待 QA 确认 |
| 主要剩余风险 | 多 alias 定价解析、单轮多次 LLM 调用聚合、持久化失败时的前端可见行为 |
| 建议 | 合并或发版前优先执行高优先级用例 |
