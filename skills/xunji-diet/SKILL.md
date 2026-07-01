---
name: xunji-diet
description: "Use when user asks to read/search/write 训记 (Xunji) App diet/food data. Query diet records, search foods, create custom foods, and apply meal templates via Open API."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [xunji, diet, food, nutrition, api, 训记]
---

# xunji-diet — 训记饮食数据 Open API

> **触发条件：** 用户明确要求读取、搜索、写回 训记 饮食/食物数据时才调用。
> **API：** 训记饮食 Open API（`https://eatings.xunjiapp.cn` / `https://api.xunjiapp.cn`）

---

## 何时使用

- 用户说"读一下今天的饮食"、"搜一下鸡蛋的营养"、"把这餐写回训记"
- 需要创建自定义食物（搜索不到官方食物时）
- 需要查询或套用饮食模板

## 何时不要用

- 用户没要求时主动调用
- 与训记无关的饮食数据（MyFitnessPal、手写记录等）——那些另走独立工作流
- 任何"探测性"读取（先问再读）

---

## 鉴权

**两套 Key，两个 Base URL：**

| 用途 | Base URL | 环境变量 | 请求头 |
|------|----------|---------|--------|
| 饮食记录 / 自定义食物 / 模板 | `https://eatings.xunjiapp.cn` | `XUNJI_FOOD_API_KEY` | `Authorization: Bearer <key>` |
| 食物搜索 | `https://api.xunjiapp.cn` | `XUNJI_FOOD_SEARCH_KEY` | `Authorization: Bearer <key>` |

- 饮食记录接口也兼容 `x-api-key`；食物搜索也兼容 `x-agent-key` 或 `x-api-key`
- **不支持**把 key 放在 body / query / URL 里
- key 不进日志、不进 commit、不进 diff
- key 不在 skill 文件里——通过环境变量注入

### curl 注意事项

- 所有接口路径以 `_gzip` 结尾，服务端返回 **gzip 压缩**数据
- curl 必须加 `--compressed`，否则拿到二进制乱码

---

## 接口总览

| 操作 | 方法 | 路径 | Base URL |
|------|------|------|----------|
| 查询饮食记录 | POST | `/open/food/query_gzip` | eatings |
| 写回饮食记录 | POST | `/open/food/upsert_gzip` | eatings |
| 创建/更新自定义食物 | POST | `/open/food/custom/upsert_gzip` | eatings |
| 查询饮食模板 | POST | `/open/food/templates/list_gzip` | eatings |
| 套用饮食模板 | POST | `/open/food/templates/apply_gzip` | eatings |
| 搜索官方食物 | POST | `/open_agent/food/search_gzip` | api |

---

## 查询饮食记录

```http
POST https://eatings.xunjiapp.cn/open/food/query_gzip
Authorization: Bearer <XUNJI_FOOD_API_KEY>
Content-Type: application/json

{
  "start_date": "2025-06-12",
  "end_date": "2026-09-12",
  "include_detail": true
}
```

- 查询范围：**过去一年 ~ 未来 3 个月**。用户要更大范围时解释限制并拆分请求
- `include_detail: true` 返回完整食物信息（`ntr`、`units` 等）
- 只读取用户明确需要的日期范围；不要为模糊问题扫全量
- 缓存按 `{start_date}_{end_date}` 组合键

---

## 搜索食物

```http
POST https://api.xunjiapp.cn/open_agent/food/search_gzip
Authorization: Bearer <XUNJI_FOOD_SEARCH_KEY>
Content-Type: application/json

{
  "keyword": "鸡蛋",
  "limit": 8
}
```

- 搜索结果优先使用 `res.foods`：其中 `ntr` 是每 100g 营养，`units` 是可选单位换算，`uniquekey` 写回时必须带上
- `res.d` 是客户端同款压缩数组，格式 `[id, name, cal, carb, fat, protein, foodpic, uniquekey, units]`
- **不确定食物匹配、单位或份量时先让用户确认**——不要只凭相似名称直接写回
- 搜索不到 → 创建自定义食物（见下文）

---

## 创建自定义食物

```http
POST https://eatings.xunjiapp.cn/open/food/custom/upsert_gzip
Authorization: Bearer <XUNJI_FOOD_API_KEY>
Content-Type: application/json

{
  "client_request_id": "xjfood-{epoch_ms}-{nonce}",
  "dry_run": false,
  "food": {
    "name": "用户确认的食物名",
    "ntr": {
      "cal": 165,
      "protein": 31,
      "fat": 3.6,
      "carb": 0,
      "foodpic": "",
      "foodUnit": [{ "unit": "份", "count": "1", "gram": 100 }]
    },
    "units": [{ "unit": "份", "count": "1", "gram": 100 }]
  }
}
```

**触发条件：**
- 搜索不到合适官方食物
- 用户明确要创建私有食物（包装食品、餐厅食物等）

**创建流程：**
1. 搜索官方食物 → 搜不到或用户不满意
2. Agent 自行查公开营养信息（网页、包装成分表）获取每 100g 营养
3. **展示给用户确认**：食物名 + 每 100g 热量/蛋白质/脂肪/碳水 + 来源
4. 用户确认后发请求
5. 创建成功后，用返回的 `res.food`（`name`、`uniquekey`、`unit/units`、`ntr`）再写回饮食记录

**注意事项：**
- `ntr` 数值均为**每 100g**：`cal`、`protein`、`fat`、`carb`
- 可带 `foodpic`（空字符串即可）
- `units` 和 `ntr.foodUnit` **必须一致**
- 没有明确份量单位时传空数组 `[]`，默认按克记录
- **不确定营养数据时追问用户，不要估算**

---

## 写回饮食记录

```http
POST https://eatings.xunjiapp.cn/open/food/upsert_gzip
Authorization: Bearer <XUNJI_FOOD_API_KEY>
Content-Type: application/json

{
  "client_request_id": "xjfood-{epoch_ms}-{nonce}",
  "dry_run": false,
  "foods": [
    {
      "date": "2026-06-12",
      "meal_type": "lunch",
      "name": "鸡胸肉",
      "amount": 150,
      "unit": "g",
      "uniquekey": "搜索或自定义食物返回的 uniquekey",
      "ntr": { "cal": 165, "protein": 31, "fat": 3.6, "carb": 0 }
    }
  ]
}
```

**写回规则：**

- 官方食物：写前先搜索 → 带上 `uniquekey`、`units`、`ntr`
- 自定义食物：用创建接口返回的 `res.food` 作为写回来源
- 写回前**必须已有 `ntr`**——接口不会搜食物或猜营养
- `meal_type` 合法值：`breakfast`、`lunch`、`dinner`、`snack`
- 有服务端 id → 更新；无 id → 新建
- 不确定餐次、数量、单位或食物匹配时**先追问用户**

---

## 饮食模板

### 查询模板

```http
POST https://eatings.xunjiapp.cn/open/food/templates/list_gzip
Authorization: Bearer <XUNJI_FOOD_API_KEY>
Content-Type: application/json

{}
```

### 套用模板

```http
POST https://eatings.xunjiapp.cn/open/food/templates/apply_gzip
Authorization: Bearer <XUNJI_FOOD_API_KEY>
Content-Type: application/json

{
  "template_id": "xxx",
  "date": "2026-06-12"
}
```

- 套用模板属于写回操作，**必须先展示摘要并等用户确认**

---

## 工作流

### 读取饮食

1. 拿日期范围（YYYY-MM-DD）
2. 检查缓存 `~/.hermes/state/xunji-cache/diet/{start_date}_{end_date}.json`
3. 命中且 TTL 未过 → 直接返回
4. 未命中 → 发请求，写缓存

### 写回饮食（关键路径）

1. 收集目标日期、餐次、食物、数量、单位
2. 每个食物搜官方库 → 有则带 `uniquekey` + `ntr`；无则走创建自定义食物流程
3. 拼 diff，**展示给用户**：
   - 日期 + 餐次
   - 每个食物：名称、数量、单位、热量/蛋白质/脂肪/碳水
   - 新建 vs 更新哪些记录
4. **等用户明确确认**（"确认" / "OK" / "写回"）才发请求
5. 发请求前过限频：距上次同类请求 ≥ 15s
6. 成功后用服务端返回数据覆盖缓存

### 搜索食物流程

1. 搜官方库
2. 找到 → 展示候选（名称 + 每 100g 营养），让用户选
3. 找不到 → 问用户是否创建自定义食物 → 走创建流程

---

## 限频

| 操作 | TTL |
|------|-----|
| 饮食记录（查询/写回/自定义/模板） | 15s |
| 食物搜索 | 15s |

- 限频时间戳存 `~/.hermes/state/xunji-cache/ratelimit-food.json`

---

## 错误处理

| 服务端返回 | 应对 |
|---|---|
| `too frequent` | 等提示的 retry 时间后重试 |
| `apikey missing` / `apikey invalid` | 让用户回 App 复制新 key；不要在 chat 里贴明文 |
| `仅VIP可用` | 告知当前账号需要会员权限；不自动尝试绕过 |
| 4xx 校验错误 | 停下报告具体字段，**不要**"修一下再发" |
| 5xx / 网络错误 | 退避重试 1 次，仍失败则报告并停下 |
| 搜索无结果 | 提示用户可创建自定义食物 |

---

## 安全与隐私

- **API key 不写进 skill 文件 / git tracked 路径 / 任何 commit**
- 饮食记录 key 通过 `XUNJI_FOOD_API_KEY` 注入
- 食物搜索 key 通过 `XUNJI_FOOD_SEARCH_KEY` 注入
- skill 文件、缓存文件、日志里**绝不出现明文 key**
- 写回 `client_request_id` 用本地生成（`epoch_ms` + 随机 nonce）

---

## 缓存位置

- 饮食记录缓存：`~/.hermes/state/xunji-cache/diet/{start_date}_{end_date}.json`
- 食物搜索结果：`~/.hermes/state/xunji-cache/diet/search/{keyword}.json`（TTL 1h）
- 限频时间戳：`~/.hermes/state/xunji-cache/ratelimit-food.json`

---

## 与 xunji-trains 的关系

- **独立 skill**，不同数据域（饮食 ≠ 训练）
- 共享 `~/.hermes/state/xunji-cache/` 目录，但不同子目录/文件名
- 共享限频文件分离（`ratelimit-food.json` vs `ratelimit.json`）
- 用户可能同时操作训练和饮食，不要混用接口
