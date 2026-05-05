# 个人知识管理上下文

## 用户信息
- 角色：技术团队成员
- 关注领域：AI Agent、低代码/无代码、推荐系统、电商运营
- 知识库路径：~/knowledge-base/
- 偏好语言：中文（技术术语保留英文）

## 知识库结构
```
~/knowledge-base/
├── cards/                    # 知识卡片（每篇文章一个 .md 文件）
│   ├── ai-agent/            # AI Agent 方向
│   ├── lowcode/             # 低代码方向
│   ├── recsys/              # 推荐系统方向
│   └── ecommerce/           # 电商运营方向
├── reports/                  # 研究报告
├── weekly-digest/            # 每周知识摘要
└── index.md                  # 知识索引（自动维护）
```

## 知识卡片模板
每张知识卡片使用以下固定格式：
```
# {标题}
- **来源**：{作者/机构} | {发布平台}
- **日期**：{YYYY-MM-DD}
- **链接**：{URL}
- **标签**：{tag1}, {tag2}, {tag3}
- **时效性**：{长期有效 / 时效性强 / 已过时}

## 核心观点（一句话）
{用一句话概括文章的核心主张}

## 关键要点
1. {要点1}
2. {要点2}
3. {要点3}

## 关键数据
- {重要数字/百分比/对比数据}

## 个人备注
{读后感、与其他知识的关联、待验证的问题}
```

## 标签体系
- 技术方向：ai-agent, llm, rag, mcp, lowcode, recsys
- 内容类型：tutorial, case-study, trend, opinion, research
- 重要程度：must-read, worth-reading, skim
- 行动相关：actionable, for-reference, needs-discussion

## 知识整理规则
- 每周五自动生成「本周知识摘要」
- 新卡片自动加入 index.md 索引
- 超过 6 个月未回顾的卡片标记为"待回顾"