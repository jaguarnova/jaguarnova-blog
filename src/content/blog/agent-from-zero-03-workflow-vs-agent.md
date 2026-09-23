---
title: 'AI Agent 从零学（三）：Workflow 和 Agent 的边界，用一次失败实验讲清'
pubDate: 2026-09-23
description: '同一任务两种实现：固定管线在真实 API 上先跪了，自主循环也踩了协议坑。附 Agent 状态结构与重试/超时的生产化改造（data-agent v1）。'
tags: ['ai-agent', 'go', '学习笔记']
---

系列第四篇。阶段三的核心问题只有一个：**什么时候该用固定 Workflow，什么时候该用自主 Agent？** 与其背结论，我把同一个任务用两种范式各实现一遍（[源码](https://github.com/jaguarnova/agent-lab/blob/main/ch03/06_workflow_vs_agent/main.go)），让真实失败给出答案。

## 一、实验设置

任务：统计 sales 表各区域销售总额，报告最高的区域。

**Workflow 版**：三步写死的管线，无循环——
```
LLM 生成 SQL → 程序执行 → LLM 总结
```
**Agent 版**：模型带 `sql_query` 工具自主循环，最多 6 轮，直到 `finish_reason != tool_calls`。

## 二、Workflow 先跪了：管线脆在步骤边界

第一轮运行，Workflow 在**第一步**就失败了：模型返回的 `content` 是空的。原因：`deepseek-flash` 默认开启思考模式，推理内容走独立字段，被截断或移出 content 后，我拿到的是空字符串——固定管线的第二步拿着空 SQL 去查库，结果当然是 null。

修好这一处（剥离 markdown、空输出兜底报错），第二轮通过：725 tokens / 3.9s，答案正确。

**固定管线的特性暴露无遗**：任何一步的输出不符合下一步的期待，整条链就断，而且断在中间——没有自纠机会，错误只能向上抛。这印证了 Anthropic 那篇的说法：workflow 用确定性换可预测性，代价是**脆弱性集中在中继点**。

## 三、Agent 也跪了，但死法不同、修法不同

Agent 版第一轮失败得更有意思——HTTP 422。两个真实的协议坑：

1. **assistant 消息里的 tool_calls 忘了带 `type: "function"`**：服务端强校验，直接 422。回传消息的字段必须和协议严格一致，"大概齐"不行。
2. **把整个参数 JSON 当 SQL 传了**：模型给的 `arguments` 是 `{"sql": "SELECT ..."}`，我直接把这个字符串传给了数据库执行——SQLSTATE 42601 语法错误。有意思的是模型的表现：它收到错误回传后自己换了写法重试，三次都是同样的解析 bug，最后它说"工具坏了，请修复后我再统计"。**模型的自纠能力是对的，拦住它的是我代码里的 bug。**

修复后 Agent 一次通过：929 tokens / 1.35s，答案正确且带了三个区域的完整数据。

### 对照数据（任务相同，均为正确结果时）

| | Workflow | Agent |
|---|---|---|
| tokens | 725 | 929 |
| 延迟 | 3.9s | 1.4s |
| LLM 调用次数 | 2 | 1（+工具执行） |
| 首次运行 | 失败（空 SQL） | 失败（协议 422） |

样本太小不构成结论，但趋势和文献一致：简单、路径可预知的任务，workflow 更省 token、更快；agent 的优势在路径不可预知时才出现。

## 四、边界判定（本篇的真正产出）

结合实验和三篇文献，我现在的判断标准：

```
步骤固定、每步可独立校验          → Workflow（prompt chaining / routing）
路径依赖中间结果、无法预知步数     → Agent Loop
两者都有                         → Workflow 为主，局部嵌 Agent
拿不准                           → 先 Workflow（Anthropic：能用简单的就用简单的）
```

还有一个从实验里长出来的认知：**Agent 不是 Workflow 的替代，而是把"步骤间怎么走"的决定权从代码移交给模型，代价是 token 和不确定性**。所以护栏（最大轮数）不是可选项——本轮实现里 6 轮上限是第一个写进去的参数。

## 五、data-agent v1：把护栏变成代码

主项目同步升级（[v1 diff](https://github.com/jaguarnova/data-agent)）：

1. **状态结构**：`State{Rounds, Status, LastErr}`——每次运行返回所处轮数与结局（completed / max_rounds / error），不再只是黑盒答案
2. **重试**：网络错误与 5xx 指数退避重试（1s/2s/4s），**4xx 不重试**（鉴权/参数错，重试无意义）——错误分类是重试策略的全部
3. **单请求超时**：`context.WithTimeout` 每次调用独立超时，上层还有整体 ctx
4. **新增 file 工具**：SQL 之外多一个信息来源，为多工具协同做准备

失败注入用 `httptest` 做了三个单测：5xx 两次后第三次成功（重试生效）、401 一次即败（不重试生效）、永远返回 tool_calls（最大轮数护栏在第 4 轮触发）。`go test` 全绿。

## 六、下一步

阶段四：MCP 协议接入 + Context Engineering。把 `read_file` 换成 MCP 生态版体会"零实现接入"，然后开始处理上下文窗口占用问题。

---

本系列代码仓库：[agent-lab](https://github.com/jaguarnova/agent-lab)（练习）· [data-agent](https://github.com/jaguarnova/data-agent)（主项目）
