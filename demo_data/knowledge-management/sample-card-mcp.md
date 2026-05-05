# Model Context Protocol (MCP) —— LLM 与外部工具的标准交互协议

- **来源**：Anthropic 官方
- **日期**：2025-11-25
- **链接**：https://modelcontextprotocol.io/introduction
- **标签**：mcp, ai-agent, tutorial, must-read
- **时效性**：长期有效（协议标准，持续演进中）

## 核心观点（一句话）
MCP 是一个开放标准协议，定义了 LLM 应用如何统一地连接和调用外部数据源与工具，解决了当前 AI Agent 生态中"每个工具一套接口"的碎片化问题。

## 关键要点
1. **解决的问题**：目前每个 AI 应用（如 Claude、ChatGPT、Hermes）都需要为每个外部工具单独写集成代码。MCP 提出一个统一协议，让工具接入变成"写一次、到处用"
2. **架构模型**：采用 Client-Server 架构。AI 应用是 MCP Client，外部工具是 MCP Server。通过 JSON-RPC 2.0 通信
3. **三大能力**：Resources（暴露数据给 LLM 读取）、Tools（暴露可执行的操作）、Prompts（暴露预设的提示词模板）
4. **核心类比**：MCP 之于 AI Agent，就像 USB 之于硬件设备 —— 统一接口标准，即插即用
5. **采用现状**：Claude Desktop、Hermes Agent、Cursor 等主流 AI 工具已支持 MCP

## 关键数据
- 截至 2026-05，已有 200+ 个开源 MCP Server 可直接使用
- 支持两种传输方式：stdio（本地进程）和 HTTP+SSE（远程服务）
- 协议版本：当前为 2025-06-18 版本

## 个人备注
- MCP 和 Hermes Agent 的 Skill 系统有互补关系：Skill 解决"怎么做"，MCP 解决"用什么工具做"
- 需要关注：MCP 生态的 Server 质量参差不齐，实际使用时需要验证安全性
- 待验证：MCP 在高并发场景下的性能表现如何？