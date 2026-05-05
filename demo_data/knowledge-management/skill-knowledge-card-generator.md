---
name: knowledge-card-generator
description: 从文章 URL 自动生成结构化知识卡片并归档
version: 1.0.0
author: Hermes Agent (auto-generated)
category: knowledge-management
tags:
  - 知识管理
  - 摘要
  - 知识卡片
  - 文章阅读
  - research
dependencies: []
difficulty: beginner
estimated_time: 2 minutes per article
---

# 知识卡片生成器

> 本技能由 Hermes Agent 根据用户的知识管理习惯自动生成。

## 何时加载此技能
当用户：
- 给出一个或多个文章链接，要求阅读、摘要或归档
- 说"帮我读一下这篇文章"、"总结一下这个链接"、"加入知识库"
- 要求批量处理多篇文章

## 执行步骤

### 第 1 步：获取文章内容
- 用 `web_extract` 提取文章正文内容
- 如果是 PDF，用 `web_fetch` 下载后提取文本
- 如果页面有付费墙或无法访问，告知用户并跳过

### 第 2 步：生成知识卡片
按以下固定模板生成 Markdown 知识卡片：

```
# {标题}

- **来源**：{作者/机构} | {发布平台}
- **日期**：{YYYY-MM-DD}
- **链接**：{URL}
- **标签**：{自动标注 3-5 个标签}
- **时效性**：{长期有效 / 时效性强 / 已过时}

## 核心观点（一句话）
{用一句话概括文章的核心主张}

## 关键要点
1. {要点1}
2. {要点2}
3. {要点3}（最多5个）

## 关键数据
- {重要数字/百分比/对比数据}

## 个人备注
{与用户已有知识的关联、待验证的问题}
```

### 第 3 步：自动分类
根据文章内容自动判断所属目录：
- AI Agent 相关 → `~/knowledge-base/cards/ai-agent/`
- 低代码相关 → `~/knowledge-base/cards/lowcode/`
- 推荐系统相关 → `~/knowledge-base/cards/recsys/`
- 电商运营相关 → `~/knowledge-base/cards/ecommerce/`
- 其他 → `~/knowledge-base/cards/misc/`

### 第 4 步：归档保存
- 文件名格式：`{YYYY-MM-DD}-{简短标题}.md`
- 保存到对应分类目录
- 更新 `~/knowledge-base/index.md` 索引文件

### 第 5 步：关联推荐
生成卡片后，检查知识库中是否有相关的已有卡片：
- 标签重叠度 > 50% 的卡片
- 同一领域但观点不同的卡片
- 如有关联，在"个人备注"中标注

## 标签规则（从用户习惯中学到的）
- 技术方向标签：ai-agent, llm, rag, mcp, lowcode, recsys
- 内容类型标签：tutorial, case-study, trend, opinion, research
- 重要程度标签：must-read, worth-reading, skim
- 行动标签：actionable, for-reference, needs-discussion

## 质量标准
- "核心观点"必须是一句完整的判断句，不能是模糊的描述
- "关键要点"每条不超过 2 句话
- "关键数据"只保留最有说服力的 3-5 个数字
- 不整段复制原文，用自己的语言重新组织

## 注意事项
- 如果文章超过 5000 字，先整体阅读再提炼，不要只读前几段
- 如果文章有多个核心论点，在"关键要点"中分别列出
- 如果文章引用了其他重要文献，在"个人备注"中标注"延伸阅读"