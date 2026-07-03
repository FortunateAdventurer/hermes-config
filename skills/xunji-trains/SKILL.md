---
name: xunji-trains
description: "Use when user asks to read/write/organize 训记 (Xunji) App training data. Reads/writes training records via Open API v2, with caching, rate-limiting, and pre-write confirmation. Movements use Chinese names only."
version: 1.3.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [xunji, training, fitness, api, 训记]
---

# xunji-trains — 训记训练数据 Open API

> **触发条件：** 用户明确要求读取、整理、写回 训记 训练数据时才调用。
> **API：** 训记 Open API v2（`https://trains.xunjiapp.cn`）

---

## 何时使用

- 用户说"读一下 2026-04-02 的训练"、"把今天的训练写回训记"、"整理一下训记里的数据"
- 需要把外部 OCR/session .md 里的动作、重量、次数回写到 训记
- 需要从 训记 拉取某天的训练做对比/校对

## 何时不要用

- 用户没要求时主动调用
- 与 训记 无关的训练数据（教练手写卡片、Apple Watch 本地数据、OCR 草稿）—— 那些继续走 `projects/exercise/` 工作流
- 任何"探测性"读取（先问再读）

---

## 鉴权

- 请求头 `Authorization: Bearer <key>`
- 也兼容 `x-api-key: <key>`
- **不支持** 把 key 放在 body / query / URL 里
- key 不进日志、不进 commit message、不进 commit diff
- key 不在 skill 文件里——通过环境变量 `XUNJI_API_KEY` 注入
- 用户拿到 key 后，提示存到 1Password / macOS keychain，不要明文落盘

### curl 注意事项

- 服务端返回 **gzip 压缩**数据，curl 必须加 `--compressed`，否则拿到二进制乱码
- 所有 API 调用示例均需含 `--compressed`

---

## 接口

### 读：`POST /api_trains_for_llm_v2`

```json
{
  "schema_version": "train_open_api_v2",
  "datestr": "2026-04-02",
  "include_full_data": false
}
```

- 成功：`{ success: true, res: { trains: [...] } }`
- 训练在 `res.trains`；每条保留 `localid`、`start`、`end`
- 默认 `include_full_data: false`（轻量，接近 v1）
- 需要以下字段时传 `true`：
  - 未完成组（`done: false`）
  - RPE
  - 备注 / 完成感受
  - 左右侧重量
  - 实练秒数 / 每组休息秒数
- 有氧 / 计时 / Tabata / 苹果健康等记录型动作：在 `sets[].metrics` 返回 `distance` / `kcal` / `calories` / `workoutTime` / `avgHeartRate` / `maxHeartRate`
- 苹果健康训练的 `name` 返回运动类型（如 `Running`）；老数据会从训练标题推断
- 超级组 / 递减组：在 `sets[].items[]` 返回子动作；每子项 `set` 里有 `weight` / `unit` / `reps` / `time` / `metrics`
- 服务端会返回 `movementCatalogUrl`（如 `/api_movement_catalog_for_llm_v2`），那是服务端动作 catalog；GitHub 名表是离线文档版

### 写：`POST /api_upsert_trains_for_llm_v2`

```json
{
  "schema_version": "train_open_api_v2",
  "client_request_id": "xunji-{epoch_ms}-{nonce}",
  "dry_run": false,
  "include_full_data": false,
  "res": [
    {
      "datestr": "2026-04-02",
      "localid": 123456,
      "title": "胸部训练",
      "start": 1744010000000,
      "end": 1744013600000,
      "movements": [
        { "name": "杠铃卧推", "sets": [
          { "done": true, "weight": "60", "unit": "kg", "reps": "10" }
        ] }
      ]
    }
  ]
}
```

**关键约束：**

- 动作只传中文 `name`，**不传 `key`**；服务端按中文名匹配并回填内部 key
- `res` 数组或 `{ "trains": [...] }` 都行；单次最多 4 条训练，必须同一天
- 每训练最多 15 个动作（write 限制）；每动作最多 20 组（超限服务端拒绝）
- 有 `localid` → 更新；无 `localid` → 新建；**列表里没出现 ≠ 删除**
- 更新旧训练时保留 `localid` / `start` / `end`，除非用户明确要改时间
- 组至少包含 `weight` / `weight_kg` / `reps` / `time` / `duration_s` / `selfWeight` 之一
- **默认 `done: true`（2026-06-23 政策）**：写回时所有 set 默认 `done=true`，不再单独确认完成状态。例外：用户明确说"未完成 / done: false"时按用户要求
- 未完成组用 `done: false`；**不要** 把完整模式读到的未完成组擅自删掉
- 写回成功后用服务端返回的标准化 `res` 覆盖缓存
- **服务端没有 delete 端点**；写错的记录只能用户去训记 App 里手动删

---

## RPE 与动作完成难度

RPE（自感用力程度）写在具体组上，动作完成难度写在动作对象上。

### RPE

- 字段：`movements[].sets[].rpe`
- **合法值（必须是字符串）**：`"6"`、`"6.5"`、`"7"`、`"7.5"`、`"8"`、`"8.5"`、`"9"`、`"9.5"`、`"10"`
- **清空 RPE 用 `""`（空字符串），不要写 `0`**
- 超级组 / 递减组子项的 RPE 写在 `sets[].items[].set.rpe`
- RPE 数据只有 `include_full_data: true` 时才返回——改 RPE 前必须用完整模式读取原训练
- 写回时保留原训练其它动作、组和 note 元数据，不要因为只改 RPE 就丢掉别的字段
- 写回涉及 RPE 时，建议请求里传 `include_full_data: true`，方便服务端返回完整标准化数据

### 动作完成难度

- 字段：`movements[].difficulty`
- **合法值（英文）**：`easy`、`normal`、`hard`
- **不要**把中文"简单/正常/困难"写进 `difficulty` 字段
- 改难度前用 `include_full_data: true` 读取原训练；写回时保留其它元数据

**示例 — 同时改 RPE + difficulty：**

```json
{
  "include_full_data": true,
  "res": [
    {
      "datestr": "2026-04-02",
      "localid": 123456,
      "movements": [
        {
          "name": "杠铃卧推",
          "difficulty": "hard",
          "sets": [
            { "done": true, "weight": "60", "unit": "kg", "reps": "10", "rpe": "8.5" }
          ]
        }
      ]
    }
  ]
}
```

---

## 历史训练颜色

训练历史卡片的背景颜色存在训练 `note` 对象里。

- **字段路径：** `note.trainColor`（**不是**顶层的 `color`）
- **格式：** CSS 十六进制字符串，如 `#FF7A00`
- **清空自定义颜色：** 传 `""`（空字符串）

### 改颜色时的注意事项

- 改颜色前**必须先读取原训练**（保留原始 `localid`、`datestr`、`start`、`end`、`title`、`movements` 等字段不变，只改 `trainColor`）
- 如果 `note` 是 JSON 字符串：先解析成对象，合并 `trainColor` 后再写回
- **不要覆盖** `note` 里的其他字段：`text`（备注文字）、`heartRate`、`customTitle`、`personalworkout_*` 等
- 示例 — 仅改颜色，保留备注文字：

```json
{
  "localid": 123456,
  "datestr": "2026-04-02",
  "note": {
    "text": "今天状态不错",
    "trainColor": "#FF7A00"
  }
}
```

---

## 工作流

### 读取

1. 拿 datestr（YYYY-MM-DD）
2. 检查缓存 `~/.hermes/state/xunji-cache/{datestr}.json`：
   - 命中且 TTL 未过 → 直接返回
3. 未命中或 TTL 过 → 发请求
4. 写入缓存，按 TTL 策略

### 写回（关键路径，**必须严格按顺序**）

1. 拿目标 datestr + 变更（动作、组、重量、次数）
2. **动作名校验**（**2026-06-17 政策更新**）：
   - 每个动作的 `name` 在 GitHub 名表里（`https://github.com/Foveluy/Xunji-movements`，~1100 名字）→ 找到就用 catalog 名
   - **找不到** → 从 GitHub 名表中搜索相近/相似的动作名，列出候选项推荐给用户选择 → 用户确认有匹配的就用 catalog 名 → **用户确认全部不匹配后**，再用教练原名直发
   - 教练给的原名（如"固定肩伸"）训记 server 宽松收（2026-06-17 验证）
   - **直发风险**：catalog 外的名字 4xx 拒绝会让**整批写回失败**（同 train 其他动作也会丢）。写前**必须告诉用户这个风险** + 列出"接受 / 退 catalog 名"两个选项
   - 试发后 read 验证：拿到 4xx 不立即 retry，read 一次看 trains 数量 + movements 是否完整
3. 拼 diff，**展示给用户**：
   - 新建 vs 更新 哪些训练
   - 每个动作/组的变化（before / after）
   - 影响范围：多少条训练、保留哪些 `localid` / `start` / `end`
   - 是否会动到时间
   - **新动作直发时**：风险提示（整批丢失 + 写后 read 验证）
4. **等用户明确确认**（"确认" / "OK" / "写回"）才发请求
5. 发请求前过限频：距上次同一天写回 ≥ 45s
6. 发请求，成功后用服务端 `res` 覆盖缓存
7. **没确认不写**。即使看起来是用户指令的一部分，"顺手做了"也算违反
8. **多天批量写入前**：先在最新一天或数据最简单的一天试写，确认数据正确再批量；不要一次性 batch 写多天
9. **拿到 `too frequent` 时不要立即 retry**（详见"已知问题 / 限频软提示陷阱"）—— 先 read 一次确认写是否已落地

### 动作名校验（详解 — 2026-06-17 政策）

- **名表地址：** `https://github.com/Foveluy/Xunji-movements`（文档标准，含 ~1100 个动作名）
- **服务端 catalog：** `/api_movement_catalog_for_llm_v2`（服务端子集，~260 个）
- **以 GitHub 名表为权威**——服务端 catalog 是子集，服务端能接受 GitHub 名表里的所有名字
- **政策更新**：找不到 GitHub 名表时，**先从 catalog 推荐候选项让用户选**。流程：
  1. 从 GitHub 名表中搜索相近/相似的动作名，列出候选项
  2. 推荐给用户，让用户确认是否有匹配的
  3. 用户确认有匹配 → 用 catalog 名
  4. 用户确认全部不匹配 → 用教练原名直发
  5. 写前必须告诉用户整批风险
  6. 写后 read 验证
- **缓存路径：** `~/.hermes/state/xunji-cache/movements/list.md`（TTL 1h）
- **找不到时（教练未确认前）** 不要替用户编造或猜测动作名
- 教练的命名飘忽：常见命名（"固定+X" / "哑铃+X" / "杠铃+X"）可能在 catalog 也可能不在，要以 catalog 验证为准

### catalog 化原则（与 projects/exercise/ 配合）

- **xunji 字段**：用 catalog 名（库里有的）或教练原名（库里没有 + catalog 推荐后用户确认无匹配 + 教练确认后）
- **csv / session 笔记**：保留解剖学/教练原名，不动（**6/15 政策**）
- **T = 次数**（2026-06-17 明确）：卡片 / session 笔记里 "4×12T" = 4 组 × 12 reps，T 是次数标注
- **解剖学映射示例**（从 6/15、5/23 实操总结）：
  - 哑铃俯身肩伸 → 俯身飞鸟 (`426`)
  - 哑铃侧平举（单只）→ 单臂哑铃侧平举 (`471`)
  - 固定推肩（单边）→ 悍马机坐姿推举 (`435`) + 备注"实为单边"
  - 固定肩屈（纽泰克机）→ 悍马机坐姿推举 (`435`) + 备注"机器实为纽泰克"
  - 哑铃肩伸（单只）→ 俯身飞鸟 (`426`)
  - 固定肩伸 → **库里没有** + 教练确认 → 用教练原名 `固定肩伸` 直发
  - 史密斯推肩 → 史密斯机推举 (`432`)
  - 哑铃肩屈 → 前平举 (`425`)

---

## 限频

| 操作 | TTL | 触发条件 |
|---|---|---|
| 读（轻量，`include_full_data: false`） | 15s | 重复读同一天 |
| 读（`include_full_data: true`） | 30s | 重复 full 读同一天 |
| 写 | 45s | 重复写同一天 |

- **拿到 `too frequent` 时不要机械 sleep + retry**（详见 已知问题 / 限频软提示陷阱）—— 服务端的 rate-limit 是"软"提示，写大概率已经落地
- 服务端返回 `too frequent` → **先 read 当前 datestr**，看 trains 数量是否已经 +1；确认已写就跳过 retry
- 限频时间戳存 `~/.hermes/state/xunji-cache/ratelimit.json`

## 错误处理

| 服务端返回 | 应对 |
|---|---|
| `too frequent` | **先 read 验证写是否落地**（详见已知问题）；如未落地再等 ≥ 60s 后 retry |
| `apikey missing` / `apikey invalid` | 让用户回 App 复制新 key；不要在 chat 里贴明文 |
| `仅VIP可用` | 告知当前账号需要会员权限；不自动尝试绕过 |
| 4xx 校验错误（含动作名不在 catalog） | 停下来报告具体字段，**不要**"修一下再发"。如果用户已确认用教练原名直发，写后 read 验证；如果没确认就停下问 |
| 5xx / 网络错误 | 退避重试 1 次，仍失败则报告并停下 |

---

## 已知问题 / Lessons Learned

### ⚠️ `dry_run: true` 在 v2 API 里**不被尊重**

虽然接口 spec 把 `dry_run` 列为参数（默认 `false`），**实测传 `true` 也会真写**（2026-06-15 验证：批量 17 天 dry_run 测试，全部被真写入；后续又重复触发了一次写入，导致部分日期出现重复记录）。

**应对策略：**

- **不要**依赖 `dry_run` 做"试运行"
- 写回前的所有验证都必须在客户端完成（动作名校对、payload 完整性、限频检查）
- 多天批量写入前，**先单独写 1 天**确认无误再批量（已经写错的数据需要用户去 App 里手动删，因为服务端没有 delete 端点）
- 提示：批量写前先缓存本地 payload 到 `/tmp/xj-payloads/` 之类的目录，便于重放和排查

### ⚠️ 服务端无 delete 端点

`/api_upsert_trains_for_llm_v2` 只支持创建和更新，**不支持删除**。要清理错误记录必须：
- 训记 App 里手动删，或
- 用 `localid` "更新"成空训练（但 record 还在，只是内容空）

写回前的 diff 展示步骤因此**极其重要**——一旦确认错误数据已落地，清理成本高。

### ⚠️ 限频对 dry_run 也生效

dry_run 请求也吃 45s 写限频。批量测试时按真实写入节奏等。

### ⚠️ `too frequent` rate-limit 软提示陷阱（2026-06-16 新发现）

服务端返回 `success: false, "too frequent, retry after Xs"` 时，**实际写操作仍然落库**——只是把响应改成了"软"失败标记（同一族 bug：dry_run 也是返回标记但真写）。

实测证据（2026-06-16）：
- 6/15 力量 写回：1st attempt 拿到 `success: false, retry after 36s`，但 server 落了库（localid 1781578113899001 epoch 对应 1st write 时间）
- client sleep 40s 后 retry 又写了一次 → 重复（localid 1781578176402001）

实测证据（2026-06-17）：
- 5/23 力量 写回：1st attempt 拿到 `success: false, retry after 37s`，但 server 落了库（localid 1781702064097001 epoch 对应 1st write 时间）。read 验证 trains count 从 1 → 2，新 train 5 moves 全部到位

**应对策略：**

- 拿到 `too frequent` 时**不要立即 retry**——写大概率已经成功
- 先 read 一次当前 datestr，看 trains 数量是否已经 +1（或 +cardio 那条）
- 如果确认已经写了，**跳过 retry**
- 如果 read 后没看到新 train，再等 ≥ 60s 后 retry
- 多天批量写入时**任何一条撞 rate-limit 都要停下来 read 验证**，不能机械 sleep + retry
- 配合 dry_run 那条一起看：服务端返回的"成功/失败"标志都不可信，写回前 diff + 写后 read 双保险

### ⚠️ catalog 外名字 server 宽松收（2026-06-17 新发现）

`/api_upsert_trains_for_llm_v2` 对**不在 GitHub 名表**里的中文动作名是**宽松收**的。证据：
- 5/23 #5 `固定肩伸` 不在 GitHub 名表（~1106 名字）也不在 server catalog（~260 名字）
- 直发 → server 收，localid 1781702064097001 的 5 moves 全部落库
- read 验证 movements 数组里 name 原样是 `固定肩伸`（无自动转换）

**政策落地（2026-06-17，2026-07-01 更新）：**
- 库里有的 → catalog 名（不变）
- 库里没有 → 先从 catalog 推荐候选项给用户 → 用户确认无匹配 → 教练确认后直发
- csv / session 笔记保留解剖学 / 教练原名（不动）
- **写前必须告诉用户整批丢失风险**（catalog 外的名字 4xx 拒绝会带丢同 train 其他动作）

**应对策略：**
- 写后**必须** read 一次（不只是 200 OK 看响应，read 验证 trains 数 + movements 完整）
- 拿到 4xx 不立即 retry：先 read 看部分数据是否落地，没落地等 ≥ 60s 后 retry；落地了就停下报告（部分成功）

### ⚠️ 环境变量名安全提醒

注意：形如 `XUNJI_API_KEY` / `XXX_SECRET` / `XXX_TOKEN` 的环境变量名在 skill 文件中应避免使用 `$` 前缀（如 `$XUNJI_API_KEY`），因为某些自动化工具可能会误把变量名当作 secret 值进行 redact。skill 文件中用纯文字描述（如"通过环境变量 XUNJI_API_KEY 注入"）即可。

---

## 安全与隐私

- **API key 不写进 skill 文件 / git tracked 路径 / 任何 commit**
- 真实 key 通过环境变量 `XUNJI_API_KEY` 注入
- skill 文件、缓存文件、日志里**绝不出现明文 key**
- 缓存里可能有 `localid` / `start` / `end` 等内部标识——可本地存但不展示给第三方
- 写回 `client_request_id` 用本地生成（`epoch_ms` + 随机 nonce），便于服务端日志去重

---

## 缓存位置

- 读缓存：`~/.hermes/state/xunji-cache/{datestr}.json`
- 动作名表：`~/.hermes/state/xunji-cache/movements/list.md`（TTL 1h）
- 限频时间戳：`~/.hermes/state/xunji-cache/ratelimit.json`

---

## 完整训练数据工作流

标准流程（教练卡片 → 训记）：

```
教练卡片（拍照）→ Vision OCR → strength-log.csv（原始数据，教练原名）
    → sessions/YYYY-MM-DD.md（详细 session + 动作名校验映射表）
    → 用户确认动作名 → 训记 API 写回
```

### 步骤详解

1. **OCR 识别**：用户拍照发教练卡片 → OCR 识别日期、肌群、强度、动作/重量/组数次数
   - **优先用 MiniMax CN M3 直接 curl**（Hermes vision 工具对 minimax_cn 路由有问题，需手动绕过）
   - curl 命令：`IMG_B64=$(base64 -i <图片> | tr -d '\n') && source ~/.hermes/.env && curl -s --compressed -X POST "https://api.minimax.chat/v1/text/chatcompletion_v2" -H "Authorization: Bearer $MINIMAX_CN_API_KEY" -d '{"model":"MiniMax-M3","messages":[{"role":"user","content":[{"type":"text","text":"请识别这张健身训练卡片上的所有文字"},{"type":"image_url","image_url":{"url":"data:image/jpeg;base64,$IMG_B64"}}]}]}'`
   - OCR 后用户会纠正误读（手写卡片常见：月份数字、动作名、lbs/kg 混淆）
2. **写入 CSV**：原始数据先落 `ForHermes/projects/exercise/strength-log.csv`（字段：`date,muscle_group,intensity_pct,order,exercise,weight,sets_reps,notes`），动作名保留教练卡片原名
3. **写 session 笔记**：在 `ForHermes/projects/exercise/sessions/YYYY-MM-DD.md` 记录完整 session，含动作名校验映射表（教练原名 → catalog 名）、课前状态、OCR 纠错记录、写回 localid
4. **动作名校验**：按「动作名校验」流程逐动作匹配，推荐候选项给用户确认
5. **写回训记**：确认后通过 `api_upsert_trains_for_llm_v2` 写回，记录 `localid` 到 session 笔记

### CSV 格式

```csv
date,muscle_group,intensity_pct,order,exercise,weight,sets_reps,notes
2026-07-02,胸,75,1,杠铃卧推,22.5 kg,4×10,已确认
```

- `exercise` — 教练卡片原名（未匹配 catalog 前）
- `notes` — 确认状态 + 纠错记录（`✅ 已确认` / `OCR 纠错：xxx`）
- 教练原名在 CSV 永不动（**6/15 政策**）

### 数据源关系

- CSV / session .md 在 `ForHermes/projects/exercise/`（Obsidian vault）
- 训记 API 在 `xunji-cache/`（Hermes 本地缓存）
- 训记是其中一个数据源，不是唯一源
- 发现训记和 CSV 不一致时，停下来问用户以哪个为准
- 不要把训记数据自动覆盖到 CSV，反之同理

### ForHermes Vault 结构

```
ForHermes/
├── 欢迎.md              ← 索引入口
├── projects/exercise/   ← 健身训练跟踪
│   ├── strength-log.csv ← 原始训练数据
│   ├── sessions/        ← 每次训练详细记录
│   └── data/            ← 导出数据
├── diary/               ← 每日会话记录（AI 自动写）
├── my-knowledge/        ← AI 自主结构化知识（Claim-Evidence）
└── templates/           ← 笔记模板（diary / knowledge / claim-evidence / project-session）
```
