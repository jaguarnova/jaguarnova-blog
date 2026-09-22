---
title: 'AI Agent 从零学（二）：Tool Calling 是怎么跑起来的'
pubDate: 2026-09-22
description: '手写 Go 工具调用循环：struct 生成 JSON Schema、tool_calls 协议、错误回传自纠、结构化输出三层防线，主项目 v0 同步启动。'
draft: true
tags: ['ai-agent', 'go', '学习笔记']
---

系列第三篇。上一篇把单次 LLM 调用拆到了 HTTP 层，这一篇让模型动起手来：Tool Calling。所有代码在 [agent-lab/ch02](https://github.com/jaguarnova/agent-lab/tree/main/ch02)，主项目也同步启动了：[data-agent](https://github.com/jaguarnova/data-agent) v0。

## 一、先纠正最大的误解：模型不执行工具

Tool Calling 的"调用"是**请求**不是**执行**。完整流程是：

```
① 你在请求里声明工具（name + description + JSON Schema 参数）
② 模型返回 tool_calls：它想调哪个工具、参数是什么（JSON 字符串）
③ 你的代码解析参数、真正执行工具
④ 把结果以 role=tool 消息回传（带上 tool_call_id）
⑤ 模型看到结果：要么再发起 tool_calls（继续循环），要么给最终回答
```

模型全程只干一件事：**决定调什么、传什么参数**。执行永远发生在你的进程里——这也是权限控制、审计、沙箱的挂载点。

## 二、工具怎么声明：Go struct → JSON Schema

工具的参数定义用 Go struct 描述，反射生成 JSON Schema（[源码](https://github.com/jaguarnova/agent-lab/blob/main/ch02/04_tools/main.go)，手写迷你版约 30 行）：

```go
type SQLQueryArgs struct {
    SQL string `json:"sql" desc:"要执行的只读 SQL（仅 SELECT）"`
}

// 生成结果
{"type":"object","properties":{"sql":{"type":"string","description":"..."}},"required":["sql"]}
```

description 字段值得多花心思——它就是工具的 API 文档，模型完全靠它决定怎么用。Anthropic 那篇说的 ACI（Agent-Computer Interface）第一课：把 description 当写给初级同事的 docstring 来写。

## 三、循环骨架：这就是 Agent 的原型

```go
for iter := 1; iter <= maxIterations; iter++ {
    resp := llm(messages, tools)
    messages = append(messages, resp.AssistantMessage)  // assistant 消息必须回传

    if len(resp.ToolCalls) == 0 {
        return resp.Content  // 没有工具调用 = 最终回答
    }
    for _, tc := range resp.ToolCalls {
        result := execute(tc)  // 出错也转成文本回传
        messages = append(messages, Message{Role: "tool", ToolCallID: tc.ID, Content: result})
    }
}
```

三个实现细节：

1. **assistant 消息（含 tool_calls）必须原样回传**，服务端靠它和后面的 tool 结果配对，漏了直接报错
2. **并行工具调用**：一次响应可能带多个 tool_calls（实测 DeepSeek 一轮同时发起了 SQL 查询和文件读取），逐个执行逐个回传即可
3. **错误也是数据**：工具失败不要 panic，把错误文本作为 tool 结果回传，模型自己会纠正（比如修正 SQL 语法）——"让模型自纠"是 Agent Loop 最重要的设计决策

安全边界提示：SQL 工具的只读校验（拒绝非 SELECT）必须写在**工具执行代码里**，不要指望 prompt 约束模型。

## 四、结构化输出：三层防线

工具的下游是代码不是人，输出格式必须严格可控（[源码](https://github.com/jaguarnova/agent-lab/blob/main/ch02/05_structured/main.go)）。三层防线全部手写验证过：

1. **Prompt 约束**：JSON Schema 贴进 system prompt，明确"只输出 JSON"
2. **语法校验**：`json.Unmarshal` 失败 → 带错误信息重试
3. **业务校验**：数值范围、必填字段 → 带错误重试

```go
if err := json.Unmarshal([]byte(content), &report); err != nil {
    messages = append(messages,
        assistant(content),
        user(fmt.Sprintf("你的输出不是合法 JSON：%v。请重新输出。", err)))
    continue  // 带着错误信息重试，模型修正率很高
}
```

实测：正常情况一次通过；故障注入非法输出后，第 2 次尝试即修正成功。`temperature: 0` 是结构化输出的标配——确定性优先。

## 五、主项目 v0：data-agent

学习仓库之外，主项目 [data-agent](https://github.com/jaguarnova/data-agent) 同步启动，从此所有新能力都沉淀进这个持续演进的项目：

```
用户问题 → LLM → Tool Calling → sql_query → PostgreSQL(sales 表) → 回答
```

实测："数据库里销售额最高的区域是哪个？"

```
--- stats: 工具调用 1 次（成功 1，成功率 100%）| tokens=987 | 耗时 1.38s ---
**结论：销售额最高的区域是「华南」，总销售额 6620.00，商品种类数 3 种。**
```

和练习版相比，主项目把工具循环抽成了 `internal/agent` 包：`Tool` 接口、`Stats`（Tool Success Rate / tokens 计量）、可注入的 `http.Client`（测试用假服务器）。v1 会在此之上加 ReAct 循环、重试与状态管理。

## 六、指标：Tool Success Rate 从第一天就开始记

Agent 质量不能靠感觉。v0 起每次运行都打印：

```
工具调用 N 次（成功 M，成功率 X%）| tokens=T | 耗时=D
```

后续评测体系（阶段六）会把"断言正确性"加进来，但调用成功率和 token 成本从第一天就有基线——改 prompt、换模型后对比这组数字，比肉眼感觉可靠得多。

## 七、下一步

阶段三：Agent Core——ReAct 循环、Workflow 与 Agent 的边界、`context.Context` 贯穿的超时/取消/重试。data-agent 升级为 v1 单 Agent。

---

本系列代码仓库：[agent-lab](https://github.com/jaguarnova/agent-lab)（练习）· [data-agent](https://github.com/jaguarnova/data-agent)（主项目）
