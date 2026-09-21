---
title: 'AI Agent 从零学（一·附）：LLM 调用协议工程参考'
pubDate: 2026-09-22
description: 'chat/completions 协议要点、SSE 流式防护、采样参数与三家实测数据——第 1 篇的工程参考手册。'
draft: true
tags: ['ai-agent', 'go', '学习笔记']
---

# 技术文档：LLM 调用协议的 Go 实现（ch01）

> 系列第 1 篇的技术附注：三个示例程序的实现要点、协议知识与实测数据汇总，当作《从零学（一）》正文的工程参考手册阅读。

## 1. 总体结构

```
ch01/
├── 01_raw/      裸 net/http 调用 chat/completions（非流式）
├── 02_stream/   SSE 流式解析 + idle timeout + 无状态重试
└── 03_sdk/      openai-go SDK 等价实现（对比用）
```

三个程序共享同一套环境变量约定，与供应商无关：

| 环境变量 | 说明 | 默认值 |
|---|---|---|
| `OPENAI_BASE_URL` | API 根地址（不带 `/chat/completions`） | `https://api.openai.com/v1` |
| `OPENAI_API_KEY` | Bearer Token | 必填 |
| `MODEL` | 模型 ID | 各程序内默认 |
| `IDLE_TIMEOUT` | （仅 02_stream）流空闲超时，如 `2s`/`30s` | `30s` |
| `TEMPERATURE` | 采样温度 | `0.7` |
| `TOP_P` | 候选截断（不设则走服务端默认） | 不发送 |
| `MAX_TOKENS` | 输出 token 上限 | `200` |

实测过的供应商：DeepSeek 官方直连、OpenRouter 聚合（z-ai/glm-5.3-flash、openai/gpt-5.6-sol）。同一份代码零改动切换，仅换 `OPENAI_BASE_URL` + `MODEL`。

## 2. 请求/响应协议要点

### 2.1 非流式（01_raw）

- `POST {base}/chat/completions`，JSON 请求体：`model` / `messages[]` / `temperature` / `max_tokens`
- `messages[]` 是 Agent 开发的核心数据结构：`role ∈ {system, user, assistant, tool}`，后续的工具调用、多轮对话、记忆全部是在这个数组上做文章
- 响应必看三字段：
  - `choices[0].finish_reason`：`stop`（自然结束）/ `length`（截断）/ `tool_calls`（要求调工具）——Agent 循环的终止判据
  - `usage`：`prompt_tokens` / `completion_tokens`——成本核算依据
  - `choices[0].message.content`——下一轮的输入原料

### 2.2 流式（02_stream）

请求加 `"stream": true`，响应变为 `text/event-stream` 分块长连接：

```
data: {"choices":[{"delta":{"content":"增量文本"}}]}\n\n   ← 每个事件以空行结尾
data: [DONE]\n\n                                          ← OpenAI 事实惯例的结束标记
```

- **`[DONE]` 不是 SSE 标准**，是 OpenAI 的私有约定被行业跟随；Anthropic/Gemini 原生协议不用它
- 健壮的结束判断三重兜底：`[DONE]` 标记 → `finish_reason` 非空 → 连接关闭
- 流式增量用 `delta` 替代非流式的 `message`；`completion_tokens` 只在部分供应商的流里返回

### 2.3 采样参数（三个程序统一支持）

| 参数 | 作用 | 设置建议 |
|---|---|---|
| `temperature` | 改变概率分布的形状（压平/收尖） | 工具调用、结构化输出用 0~0.2；创作 0.7+ |
| `top_p` | 按累计概率截断候选集（nucleus sampling） | 与 temperature 通常二选一调；默认 1.0 不动 |
| `max_tokens` | 输出上限 | 超限后 `finish_reason=length`，非自然结束 |

三个概念的关系：LLM 每步对词表全部 token 打出概率分布 → `top_p` 截掉长尾只留累计 P% 的头部 → `temperature` 调整头部候选的冒险程度 → 抽签产出下一个 token。

实现细节：

- Go struct 用 `omitempty` / SDK 用 `param.Opt`（`openai.Float` 赋值才序列化）——语义一致，都支持"不发送走服务端默认"
- 01_raw 的 `TOP_P` 默认 0 触发 `omitempty` 不发送；要显式测试设 `TOP_P=0.9`
- SDK 流式的 usage 同样需要 `StreamOptions{IncludeUsage: true}` 显式索取，usage 挂在 `[DONE]` 前最后一个 choices 为空的独立 chunk 上

## 3. 超时与流式读取的正确姿势

| 场景 | 错误做法 | 正确做法 |
|---|---|---|
| 非流式 | 无超时 | `context.WithTimeout` + `req.WithContext(ctx)`（取消可沿调用链传播） |
| 流式 | `client.Timeout`（限制总时长，长回答被掐断） | context 控制生命周期 + **idle timeout watchdog** |

**为什么流式会静默挂死**：NAT/防火墙超时清除映射、上游推理卡死、LB 丢包——连接看起来还在，`scanner.Scan()` 永久阻塞，既没有 `[DONE]` 也没有错误。

**watchdog 模式**（02_stream 核心逻辑）：

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
timer := time.AfterFunc(idleTimeout(), func() { cancel() })  // 超时触发取消
defer timer.Stop()

for scanner.Scan() {
    timer.Reset(idleTimeout()) // 每收到一行数据就续命
    ...
}
// watchdog 先触发 → cancel() → 阻塞中的 Read 以 context.Canceled 立刻醒来
```

正常流式的 chunk 间隔是亚秒级，30s（`IDLE_TIMEOUT` 可调）无数据必然异常。这是"流超过空闲上限无新数据"错误的来源。

**注意**：`bufio.Scanner` 默认单行上限 64KB，长 JSON chunk 会报 `token too long`，需 `scanner.Buffer(..., 1<<20)` 放宽。

## 4. 无状态重试

SSE 无法续流（没有恢复握手），但 LLM 调用是无状态的：**`messages` 就是完整状态**，断线后整体重发即可。

```
重试策略：最多 3 次，指数退避 1s → 2s → 4s
错误分类：
  ├── 4xx（鉴权/参数错）      → 不可恢复，直接失败
  ├── 5xx / 网络错误 / RST   → 可重试
  ├── idle timeout           → 可重试
  └── 收 [DONE] 前连接关闭    → 可重试（部分实现不发 [DONE]，接多供应商时放宽为 finish_reason 判断）
```

## 5. SDK（openai-go）抽象对照

| 手写层 | SDK 层 | 被封装的内容 |
|---|---|---|
| 拼 URL + Header | `option.WithBaseURL / WithAPIKey` | 鉴权、路由 |
| bufio 逐行扫描 + JSON 分帧 | `stream.Next()` 迭代器 | SSE 解析、断帧容错 |
| 区分 HTTP 错误 / 解析错误 | 统一 `stream.Err()` | 错误归一 |

结论：SDK 值得在业务中使用，但必须先手写一遍——调试时才能看懂 SDK 吞掉的错误，才能判断第三方兼容网关的行为偏差。

## 6. 实测数据（2026-09-21/22）

同一 prompt（"用 3 句话解释 SSE 协议"），同一份代码：

| | DeepSeek 官方 | z-ai GLM-5.3-flash | OpenAI gpt-5.6-sol |
|---|---|---|---|
| 非流式延迟 | 199ms | 3.33s | 1.75s |
| 流式 | 102 chunk / 2.3s / 458 字 | 42 chunk / 13.6s / 486 字 | 60 chunk / 4.9s / 425 字 |
| 响应模型名 | `deepseek-flash`（底层模型名，≠请求名） | 原样返回请求名 | 原样返回请求名 |

延迟差距 10 倍以上——Agent 循环多轮工具调用会放大这个差距，选型不能只看模型能力。

## 7. 踩坑实录（已验证的坑）

1. **响应 `model` 字段 ≠ 请求 `model`**：DeepSeek 请求 `deepseek-chat` 返回 `deepseek-flash`——请求的是产品名，响应是实际底层模型。不要对 model 字段做字符串等值断言。
2. **Go `json.Unmarshal` 静默丢弃未知字段**：请求 `deepseek-reasoner` 时响应含思维链字段 `reasoning_content`，struct 未定义则无声丢失。对接新协议先看原始 JSON；关键字段用 `json.RawMessage` 保底。
3. **推理模型的 token 计量不同**：思考过程计入 `completion_tokens`（同 prompt 用量 60 → 206），成本预算必须按模型区分。
4. **`client.Timeout` 会掐断长流**：它覆盖"发请求到 body 读完"的全时长。
5. **`[DONE]` 缺失是真实场景**：部分实现直接关连接，结束判断要三重兜底（见 §2.2）。
6. **供应商地域墙**：同一聚合网关，部分上游模型按出口 IP 拒绝服务（403 + `routing_funnel` 元数据），换出口即恢复。多供应商架构天然缓解此问题。

## 8. 遗留与后续

- [ ] 结束判断放宽为 `finish_reason` 优先（接更多供应商前）
- [ ] token/latency 落盘记录（当前仅 stdout）
- [ ] 三个练习合并为统一 CLI Client（阶段一产出物要求）
- [x] 采样参数（temperature/top_p/max_tokens）env 可配，三程序统一（2026-09-22）
- [ ] 阶段二：Tool Calling + Structured Output，在此 Client 上迭代
