---
title: '为什么你的 Agent 需要一个显式的工具调用循环'
pubDate: 2026-09-20
description: '从手写 ReAct 循环聊起：记忆、工具重试与上下文管理的一次踩坑复盘。'
---

AI Agent 开发系列的第一篇。这里记录实战经验：多轮对话记忆怎么存、工具调用的失败重试策略、context window 管理。

## 下一步

- [ ] 一篇解决一个具体问题，标题写清楚"怎么做 X"

## 代码示例

```python
def run_agent(goal: str) -> str:
    """最小 ReAct 循环骨架"""
    messages = [{"role": "user", "content": goal}]
    while True:
        reply = llm(messages, tools=TOOLS)
        if not reply.tool_calls:
            return reply.content
        messages += [reply, *execute(reply.tool_calls)]
```

关注 RSS 订阅更新。
