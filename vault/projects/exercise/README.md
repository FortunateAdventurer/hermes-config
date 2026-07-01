---
tags:
  - project
  - exercise
status: active
---

# 健身训练跟踪

## 目标

系统化记录每次训练，支持教练卡片 OCR → 结构化数据 → 纵向对比分析。

## 工具链

- **训记 App** — `xunji-trains` / `xunji-diet` / `xunji-body` skills
- **OCR** — vision 工具（MiniMax CN）识别教练卡片
- **数据源** — 训记 API + 教练手写卡片 + 体脂秤

## 数据流

```
教练卡片（拍照）→ Vision OCR → 结构化（动作/重量/次数）
    → 动作名校验（GitHub catalog）
    → 展示 diff → 用户确认 → 写回训记 API
    → 缓存到 xunji-cache/
```

## 目录

- `sessions/` — 每次训练的 session 记录
- `data/` — 导出的结构化数据（CSV/JSON）
