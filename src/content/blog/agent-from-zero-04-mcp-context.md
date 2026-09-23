---
title: 'AI Agent 从零学（四）：MCP 接入与上下文工程'
pubDate: 2026-09-23
description: '手写 MCP Client 对接官方 filesystem server（14 个工具零实现接入），再手写 token 估算、历史裁剪与配对保护——上下文不再靠运气。'
draft: true
tags: ['ai-agent', 'go', '学习笔记', 'mcp']
---

系列第五篇，两个主题一套逻辑：**Agent 的能力边界（工具从哪来）和资源边界（上下文怎么管）**。代码在 [agent-lab/ch04](https://github.com/jaguarnova/agent-lab/tree/main/ch04) 与 [data-agent v2](https://github.com/jaguarnova/data-agent)。

## 一、MCP：一份 schema，两边通用

MCP（Model Context Protocol）用一句话说清：**把"工具怎么声明"标准化，让工具的"怎么实现"可以被复用**。它的工具定义（`inputSchema`）和 OpenAI 的 `parameters` 是同一个 JSON Schema 格式——这就是"同构"的含义：Agent 核心根本不关心工具来自本进程还是 MCP server。

手写一个最小 Client（[源码](https://github.com/jaguarnova/agent-lab/blob/main/ch04/07_mcp_client/main.go)，约 200 行，零依赖）只要四步：

```
① spawn server（stdio 子进程）           ← 传输层：stdin/stdout 各一条管道，一行一条 JSON
② initialize → initialized              ← 握手：协商协议版本（日期命名，如 2024-11-05）
③ tools/list                            ← 发现：server 报告自己有哪些工具
④ tools/call {name, arguments}          ← 执行：转发调用，拿回 content[].text
```

协议 underneath 就是 JSON-RPC 2.0：请求带自增 id，响应用同 id 配对，通知（如 `initialized`）没有 id。**没有鉴权、没有重连、没有压缩**——协议本身极薄，全部复杂度在 server 侧。

## 二、实测：14 个工具，零行实现

接官方 `@modelcontextprotocol/server-filesystem`（`npx` 拉起，一条命令）：

```
✓ 握手完成，server: secure-filesystem-server 0.2.0
✓ 发现 14 个工具：read_text_file / write_file / edit_file / directory_tree
                    / search_files / move_file / get_file_info ...
✓ 调用 read_file → module github.com/jaguarnova/agent-lab ...
```

我的 Client **没有实现任何文件逻辑**。安全边界由 server 执行：启动时限定目录，越界访问被 server 拒绝。

然后接入 data-agent v2：MCP 工具适配进统一的 `agent.Tool`（schema 直接透传），与本地 `sql_query` 并列。实测问题"销售最高区域 + 读 go.mod"：

```
--- MCP: 接入 14 个工具 ---
4 轮、3 次工具调用全成功：模型自己混用了本地 sql_query 和 MCP 的
list_directory / read_text_file——它不区分工具来源，只看 description。
```

**选型结论更新**：通用工具（文件、网页、GitHub）用生态版；带业务语义的工具（只读 SQL + 表结构知识 + 安全约束）自己写。两钟来源在同一个 Agent 里共存，因为对模型来说它们长得一样。

## 三、Context Engineering：messages 不会自己变短

多轮 Agent 的上下文是单调递增的：每轮追加 assistant、tool 结果（SQL 结果动辄几 KB），tokens 线性上涨直到超窗或烧钱。data-agent 实测一次 4 轮任务就 10634 tokens。

手写两个工程手段（[源码](https://github.com/jaguarnova/data-agent/blob/main/internal/agent/context.go)）：

### 1. Token 估算 + usage 校准

精确分词需要 tokenizer（Go 没有官方 tiktoken），但"逼近上限就裁剪"不需要精确——按字符数估算（初始 2.5 字符/token）误差 ±20% 完全够用，且**每次 API 响应的真实 `usage.prompt_tokens` 会反向校准估算系数**（指数平滑 0.7/0.3），跑得越久估得越准。

### 2. 滑动窗口裁剪，带配对保护

裁剪不是简单砍掉最旧的消息——**tool 结果必须和发起它的 assistant(tool_calls) 一起保留**，否则服务端 400（id 配对断裂，上一篇踩过的坑）。裁剪算法：

```
从尾部保留最近 N 条
  ↓ tool 消息被保留时，连带保留它配对的 assistant 消息
  ↓ 首条 system 消息始终保留（工具声明和人格都在里面）
```

三个单测锁定行为：不超限不裁、超限保留 system+最近消息、**配对不拆散**。data-agent 每轮请求前自动检查，`MAX_CONTEXT_TOKENS` 环境变量设预算（默认 8000），运行结束打印 `truncations=N`。

### 3. 没做的：摘要（Summarization）

被裁掉的内容其实是"摘要替换"的好素材（用一次廉价 LLM 调用换信息保留），但当前阶段不值得——先用最便宜的截断，等真实任务里出现"裁掉的信息被再次需要"的 case 再上摘要。**上下文策略也是要按证据演进的，不是一次做全。**

## 四、踩坑实录

1. **MCP stdio 的响应可能混入非 JSON 行**（server 日志），解析要跳过而不是报错
2. **握手前工具不可用**：`initialized` 通知不发，`tools/list` 会挂——MCP 是有状态的会话，不像 HTTP 无状态
3. **估算器的初始比例影响首轮裁剪**：校准需要真实 usage，而校准发生在请求后——第一轮只能靠先验假设，预算要留余量
4. **MCP 工具的 description 质量参差**（Anthropic 那篇提过）：filesystem server 自己标了 `read_file DEPRECATED`——发现工具后要过滤弃用项，否则模型会优先选它

## 五、下一步

阶段五：Eino。把手写的循环、工具编排迁移到框架，同时保留现有 MCP 工具与 Context 策略——验证"框架之下是什么"的时刻到了。

---

本系列代码仓库：[agent-lab](https://github.com/jaguarnova/agent-lab)（练习）· [data-agent](https://github.com/jaguarnova/data-agent)（主项目）
