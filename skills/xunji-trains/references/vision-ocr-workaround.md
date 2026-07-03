# MiniMax CN Vision OCR Workaround

Hermes 的 `vision_analyze` 工具无法正确路由图片到 MiniMax CN API（`abab6.5s-chat` 不支持 image_url，`MiniMax-M3` 支持但 auxiliary client 路由不匹配）。

## 工作方法

直接用 curl 调 MiniMax CN 的 `v1/text/chatcompletion_v2` 端点：

```bash
source ~/.hermes/.env 2>/dev/null
IMG_B64=$(base64 -i <图片路径> | tr -d '\n')
curl -s --compressed -X POST "https://api.minimax.chat/v1/text/chatcompletion_v2" \
  -H "Authorization: Bearer $MINIMAX_CN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "MiniMax-M3",
    "messages": [{
      "role": "user",
      "content": [
        {"type": "text", "text": "<提问>"},
        {"type": "image_url", "image_url": {"url": "data:image/jpeg;base64,'"$IMG_B64"'"}}
      ]
    }],
    "max_tokens": 500
  }'
```

## 注意事项

- 端点：`https://api.minimax.chat/v1/text/chatcompletion_v2`（**不是** OpenAI 兼容的 `/v1/chat/completions`）
- 模型：`MiniMax-M3`（多模态，文字模型如 `abab6.5s-chat` 不支持图片）
- 图片格式：base64 data URL
- 返回格式：`choices[0].message.content`
- 对于大图片或耗时较长的 OCR，建议用 `background=true` + `notify_on_complete=true` 避免网关超时打断

## 已验证场景

- 教练手写训练卡片 OCR → 准确识别中文动作名、重量、组数
- Apple Watch 截图 → 准确提取心率、卡路里、时长等数据
