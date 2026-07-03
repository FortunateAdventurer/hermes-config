# OCR 工作流 — 训记训练卡片

## 整体流程

```
教练卡片拍照 → OCR → strength-log.csv → session .md → 动作名校验 → 训记写回
```

## OCR 方案

### MiniMax CN M3（当前可用）

MiniMax CN 的 M3 模型支持图片识别，但 Hermes 的 auxiliary_client 没有正确路由
MiniMax CN 的 vision 请求。临时方案：直接用 curl 调 API。

```bash
source ~/.hermes/.env
IMG_B64=$(base64 -i <image.jpg> | tr -d '\n')
curl -s --compressed -X POST "https://api.minimax.chat/v1/text/chatcompletion_v2" \
  -H "Authorization: Bearer $MINIMAX_CN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "MiniMax-M3",
    "messages": [{
      "role": "user",
      "content": [
        {"type": "text", "text": "识别这张训练卡片上的文字..."},
        {"type": "image_url", "image_url": {"url": "data:image/jpeg;base64,'"$IMG_B64"'"}}
      ]
    }],
    "max_tokens": 500
  }'
```

注意：使用 `/v1/text/chatcompletion_v2` 端点（不是 OpenAI 兼容的 `/v1/chat/completions`）。

### 其他试过的方案

| 方案 | 结果 |
|------|------|
| Hermes vision_analyze (MiniMax CN abab6.5s-chat) | 不支持图片 |
| Hermes vision_analyze (DeepSeek v4-pro) | API 拒绝 image_url 格式 |
| Hermes vision_analyze (MiniMax CN MiniMax-M3) | auxiliary_client 路由问题 |
| curl 直调 MiniMax CN M3 | 可用 |

## OCR 常见问题

- 手写数字易误读：月份 6 vs 2/1，重量 lbs vs kg
- 重量列 vs REPS 列容易读花
- 动作名 OCR 后需要人工确认（如"窄手划船"→"单手划船"）
