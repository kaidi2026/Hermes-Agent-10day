# Model Context Protocol (MCP) 介绍

> 原文来源：https://modelcontextprotocol.io/introduction
> 作者：Anthropic
> 发布日期：2025-11-25
> 本文为原文关键内容的结构化摘要，供离线实操使用

## 什么是 MCP？

Model Context Protocol（MCP）是一个开放标准协议，旨在为 AI 应用（特别是大语言模型）提供与外部数据源和工具交互的统一方式。

**核心问题**：当前每个 AI 应用都需要为每个外部工具单独编写集成代码。如果有 M 个 AI 应用和 N 个工具，就需要 M×N 个集成。MCP 将其简化为 M+N：每个 AI 应用实现一个 MCP Client，每个工具实现一个 MCP Server。

**核心类比**：MCP 之于 AI Agent，就像 USB 之于硬件设备 —— 统一接口标准，即插即用。

## 架构设计

### Client-Server 模型
```
┌─────────────┐        ┌─────────────┐
│  AI 应用     │        │  外部工具    │
│ (MCP Client) │◄─────►│ (MCP Server) │
│  Claude      │ JSON   │  文件系统    │
│  Hermes      │ RPC    │  数据库      │
│  Cursor      │ 2.0    │  Web API    │
└─────────────┘        └─────────────┘
```

### 通信协议
- 基于 **JSON-RPC 2.0** 标准
- 支持两种传输方式：
  - **stdio**：本地进程通信（适合命令行工具）
  - **HTTP + SSE**：远程服务通信（适合 Web 服务）

## 三大核心能力

### 1. Resources（资源）
- 允许 MCP Server 暴露数据给 LLM 读取
- 类似 REST API 的 GET 请求
- 示例：读取文件内容、获取数据库表结构

### 2. Tools（工具）
- 允许 LLM 通过 MCP Server 执行操作
- 类似 REST API 的 POST 请求
- 示例：执行 SQL 查询、发送邮件、创建文件

### 3. Prompts（提示词）
- MCP Server 可以暴露预设的提示词模板
- 帮助用户更好地使用 Server 的功能
- 示例：数据库 Server 提供"数据分析"提示词模板

## 关键数据

- 截至 2026-05，已有 **200+ 个开源 MCP Server** 可直接使用
- 支持的 AI 应用：Claude Desktop、Hermes Agent、Cursor、Continue 等
- 协议版本：**2025-06-18**（持续演进中）
- 官方 SDK：TypeScript、Python、Java、Kotlin

## 生态现状

| 类别 | 代表 Server | 说明 |
|------|------------|------|
| 文件系统 | filesystem | 读写本地文件 |
| 数据库 | sqlite, postgres | 数据库查询和操作 |
| Web | fetch, puppeteer | 网页抓取和浏览器自动化 |
| 版本控制 | git, github | 代码仓库操作 |
| 通信 | slack, email | 消息发送 |
| 生产力 | google-drive, notion | 文档和笔记管理 |

## 对 Agent 生态的影响

1. **降低集成成本**：工具开发者只需实现一次 MCP Server
2. **扩展 Agent 能力**：Agent 的能力边界从"框架自带工具"扩展到"整个 MCP 生态"
3. **标准化趋势**：可能成为 Agent 工具集成的行业标准
4. **安全考量**：需要建立 MCP Server 的安全审查机制