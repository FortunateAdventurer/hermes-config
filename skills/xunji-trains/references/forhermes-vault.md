# ForHermes Vault — 知识仓库结构

> AI-native 个人知识库，由 clawliu 和 Hermes Agent 共同维护。
> 位置：`~/Documents/ForHermes`（Obsidian vault）
> GitHub：`https://github.com/FortunateAdventurer/hermes-config`

## 目录结构

```
ForHermes/
├── 欢迎.md              ← 索引入口 + 维护约定
├── inbox/               ← 快速捕获，待整理
├── diary/               ← 每日会话记录（AI 自动写）
│   └── YYYY-MM-DD.md
├── knowledge/           ← clawliu 指定存档的参考资料
├── my-knowledge/        ← AI 自主判断的结构化知识（Claim-Evidence 格式）
├── memory/              ← 系统自动输出，只看不改
├── projects/            ← 活跃项目
│   └── exercise/        ← 健身训练跟踪
│       ├── README.md
│       ├── strength-log.csv
│       ├── sessions/
│       └── data/
├── archive/             ← 已完成项目（移入时加索引）
├── logs/                ← 运行日志
└── templates/           ← 笔记模板
    ├── diary.md
    ├── knowledge.md
    ├── claim-evidence.md
    ├── inbox-note.md
    └── project-session.md
```

## 分工原则

| 目录 | 负责人 | 规则 |
|------|--------|------|
| `memory/` | 系统 | 只看不改 |
| `knowledge/` | AI（按 clawliu 指令） | 用户说"记下来"才写 |
| `my-knowledge/` | AI 自主 | Claim-Evidence 结构化格式 |
| `diary/` | AI | 每次会话写一条 |
| `inbox/` | 任意 | 快速捕获，定期分流 |
| `archive/` | AI | 项目完成后移入并加索引 |

## 写作规范

- 所有笔记建议加 frontmatter（tags + date/created + status）
- 新笔记使用 `templates/` 对应模板
- 跨目录引用用 wikilink `[[路径/文件名]]`
- `my-knowledge/` 条目使用 Claim-Evidence 格式

## 工具集成

- `xunji-trains` / `xunji-diet` / `xunji-body` — 训记 API
- `obsidian` — 读写 Obsidian vault
- `apple-notes` / `apple-reminders` — Apple 生态
