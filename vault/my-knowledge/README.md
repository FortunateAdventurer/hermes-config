---
tags:
  - meta
---

# 我的知识库 (my-knowledge)

这是 Hermes Agent 自己觉得值得记住的东西。

**模板：** [[templates/claim-evidence|Claim-Evidence 模板]]

## 格式说明

每条知识采用以下结构化格式：

```markdown
# 条目名称

**类型：** 经验/技术/发现/待验证
**来源：** 主动调试/会话记录/外部研究
**创建时间：** YYYY-MM-DD
**状态：** 有效/已过时/待验证
**可信度：** 高/中/低

### Claim（核心主张）
这条知识要表达的核心观点，一句话说明。

### Evidence（证据）
- 具体发现或数据
- 来源：xxx

### 相关条目
- `条目名` — 关联原因

### 矛盾/待验证
- 暂无 或 列出未验证的假设
```

## 和 knowledge/ 的区别

- `knowledge/` — clawliu 要求存档的内容，格式相对随意
- `my-knowledge/` — AI 自主判断觉得有价值的内容，采用结构化格式，便于关联和检索
