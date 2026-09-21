---
title: 'AI Agent 从零学（一）：用 Go 手撕 LLM 调用协议'
pubDate: 2026-09-21
description: '不用 SDK 调通 chat/completions 与 SSE 流式：idle timeout watchdog、断线重试、采样参数与 openai-go 对比。'
tags: ['ai-agent', 'go', '学习笔记']
---

系列第二篇。读完三篇奠基文献（见[上一篇](../agent-from-zero-00-reading-notes/)）后，动手前的第一课：把"一次 LLM 调用"的全部细节变成肌肉记忆。所有代码在 [agent-lab](https://github.com/jaguarnova/agent-lab) 仓库，每个程序百行以内，可以照着跑。

## 一、裸 HTTP：chat/completions 到底是什么

剥掉 SDK，一次 LLM 调用就是一个普通的 HTTP POST（[源码](https://github.com/jaguarnova/agent-lab/blob/main/ch01/01_raw/main.go)）：

```
POST {base_url}/chat/completions
Authorization: Bearer sk-xxx
Content-Type: application/json

{
  "model": "deepseek-chat",
  "messages": [
    {"role": "system", "content": "你是一个简洁的中文技术助手"},
    {"role": "user", "content": "用一句话解释什么是 LLM 的 Token"}
  ],
  "temperature": 0.7,
  "max_tokens": 200
}
```

`messages` 数组是整个 Agent 开发里最重要的数据结构——后面所有概念（工具调用、多轮对话、记忆）都是往这个数组里加不同 role 的消息。响应里三个字段值得盯住：

| 字段 | 含义 | 为什么重要 |
|---|---|---|
| `choices[0].finish_reason` | `stop` / `length` / `tool_calls` | Agent 循环的终止判断依据 |
| `usage.total_tokens` | 本次调用消耗 | 成本控制的第一手数据 |
| `choices[0].message.content` | 回复内容 | 喂回下一轮的原料 |

实测（DeepSeek，首次调用）：HTTP 200，198ms，prompt=30 / completion=30 / total=60 tokens。

## 二、SSE 流式：`stream: true` 改变了什么

请求体加一个 `"stream": true`，响应就从"一次性 JSON"变成一条 `text/event-stream` 文本流。逐行解析的规则只有三条（[源码](https://github.com/jaguarnova/agent-lab/blob/main/ch01/02_stream/main.go)）：

```go
scanner := bufio.NewScanner(resp.Body)
for scanner.Scan() {
    line := scanner.Text()
    if !strings.HasPrefix(line, "data: ") { continue }  // ① 只关心 data: 行
    payload := strings.TrimPrefix(line, "data: ")
    if payload == "[DONE]" { break }                     // ② 结束标记
    // ③ 每行是一个 JSON，choices[].delta.content 是增量而非全量
    ...
}
```

实测：458 字的回复拆成 **102 个增量 chunk**，总耗时 2.3s——如果不用流式，用户要干等 2 秒才看到第一个字；流式后第一个字几百毫秒就出现。这就是所有 AI 应用都"打字机"的原因。

两个后端视角的坑：

- **不能用 `http.Client.Timeout`**：它限制总时长，而流式总时长不可预估。正确做法是用 context 承担生命周期 + **idle timeout watchdog**：

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
timer := time.AfterFunc(30*time.Second, func() { cancel() })  // 超时触发取消
defer timer.Stop()

for scanner.Scan() {
    timer.Reset(30 * time.Second)  // ★ 每收到一行数据就续命
    ...
}
// watchdog 先触发 → cancel() → 阻塞中的 Read 以 context.Canceled 立刻醒来
```

  为什么要它：NAT 超时清映射、上游推理卡死、LB 丢包——连接看起来还在，`scanner.Scan()` 会**永久阻塞**，既没有 `[DONE]` 也没有错误。正常流式 chunk 间隔是亚秒级，30s 无数据必然异常。watchdog 让挂死的流最多拖 30s。
- **断线就整体重试**：SSE 没有续流握手，但 LLM 调用是无状态的——`messages` 就是完整状态，断了把已生成的半截丢掉、整个请求重发即可（最多 3 次，指数退避 1s/2s/4s）。错误要分类：4xx 重试也不会好，直接失败；5xx、网络错误、idle timeout 才值得重试。
- **bufio.Scanner 默认单行上限 64KB**，超长 JSON chunk 会报 `token too long`，要手动放宽 buffer。
- **流式下 usage 默认不返回**，要显式加 `"stream_options": {"include_usage": true}`——服务端在 `[DONE]` 前发一个 `choices` 为空数组的独立 chunk 携带 usage，解析代码不用特殊处理。

## 三、SDK 对比：openai-go 帮你做了什么

用官方 [openai-go](https://github.com/openai/openai-go) 重写同样功能（[源码](https://github.com/jaguarnova/agent-lab/blob/main/ch01/03_sdk/main.go)），三处抽象一目了然：

| 裸 HTTP | SDK | 藏了什么 |
|---|---|---|
| 手拼 header、URL | `option.WithBaseURL / WithAPIKey` | 鉴权细节 |
| bufio 逐行扫 SSE、拼 JSON | `stream.Next()` 迭代器 | 全部分帧解析 |
| 分别处理 HTTP 错误 / 解析错误 | 统一 `stream.Err()` | 错误分类 |

SDK 版实测与裸 HTTP 一致（86 chunk / 1.05s）。**结论**：SDK 值得用在业务里，但先手写一遍的价值在于——调试时你能看懂 SDK 吞掉的报错、能判断第三方兼容网关的行为是否符合预期。Anthropic 那篇说的"不理解底层抽象的框架依赖是常见错误来源"，就是这个意思。

## 四、采样参数：temperature / top_p / max_tokens

| 参数 | 作用 | 设置建议 |
|---|---|---|
| `temperature` | 改变概率分布的形状（压平/收尖） | 工具调用、结构化输出用 0~0.2；创作 0.7+ |
| `top_p` | 按累计概率截断候选集（nucleus sampling） | 与 temperature 通常二选一调，默认 1.0 不动 |
| `max_tokens` | 输出上限 | 超限后 `finish_reason=length`，非自然结束 |

生成链路一句话：LLM 每步对词表全部 token（10 万+）打出概率分布 → `top_p` 截掉长尾只留累计 P% 的头部 → `temperature` 调整头部的冒险程度 → 抽签产出下一个 token。temperature=0 也**不等于可复现**，只是统计上最稳。

三个程序里这些参数都做成了环境变量可配（`TEMPERATURE` / `TOP_P` / `MAX_TOKENS`），做对比实验不用改代码。实现上有个 Go 细节：裸 HTTP 用 `omitempty`、SDK 用 `param.Opt`（`openai.Float` 赋值才序列化）——语义一致，都支持"不发送走服务端默认"。

## 五、两个意外发现（踩坑实录）

这是教程不会告诉你、跑起来才有的东西：

1. **struct 会静默丢弃未知字段**。切到推理模型请求 `deepseek-reasoner` 时，响应里其实有思维链字段 `reasoning_content`，但我的 struct 没定义它——Go 的 `json.Unmarshal` 直接扔掉了，没有任何报错。程序"正常工作"，数据悄悄丢失。教训：对接协议时先看原始 JSON，再定义 struct；关键字段用 `json.RawMessage` 保底。
2. **`finish_reason` 语义随模型变**。同一套代码，非推理模型 `stop` 结束，推理模型会把思考也算进 `completion_tokens`（同一段 prompt，token 用量 60 → 206）。做 Agent 的成本预算时必须考虑这一点。

## 六、下一步

下一项是第一个多轮对话程序，然后进入重头戏：手写工具调用循环（Agent Loop）。

---

本系列代码仓库：[jaguarnova/agent-lab](https://github.com/jaguarnova/agent-lab)
