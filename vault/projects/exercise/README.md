---
tags:
  - project
  - exercise
status: active
---

# 健身训练跟踪

## 目标

系统化记录每次训练，支持教练卡片 OCR → 结构化数据 → 纵向对比分析。

## 工作流

```
教练卡片（拍照）→ Vision OCR → strength-log.csv（原始记录）
    → sessions/YYYY-MM-DD.md（详细 session 笔记 + 动作名校验）
    → 用户确认动作名 → 训记 API 写回（xunji-trains skill）
```

## 文件说明

| 文件 | 用途 |
|------|------|
| `strength-log.csv` | 原始训练数据（教练卡片 OCR → 结构化 CSV） |
| `sessions/YYYY-MM-DD.md` | 每次训练的详细 session 记录 |
| `data/` | 导出的结构化数据 |

## CSV 字段

```
date,muscle_group,intensity_pct,order,exercise,weight,sets_reps,notes
```

- `exercise` — 教练卡片原名（未匹配 catalog 前）
- `notes` — 确认状态 + 纠错记录

## 工具链

- **Vision OCR** — MiniMax CN 识别教练卡片
- **xunji-trains** — 动作名校验 + 训记 API 写回
- **xunji-diet** — 饮食记录
- **xunji-body** — 身体数据（体重/体脂）

## 最近训练

（训练后自动更新）
