# MiniMax CN M3 Vision Workaround

Hermes 的 `vision_analyze` 工具无法正确路由 MiniMax CN 的视觉请求（auxiliary client 不认 minimax-cn 的 vision 能力），但 MiniMax CN M3 **确实支持多模态图片识别**。

## 直接调用方式

```bash
source ~/.hermes/.env
IMG_B64=$(base64 -i /path/to/image.jpg | tr -d '\n')
curl -s --compressed -X POST "https://api.minimax.chat/v1/text/chatcompletion_v2" \
  -H "Authorization: Bearer $MINIMAX_CN_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"MiniMax-M3\",
    \"messages\": [{
      \"role\": \"user\",
      \"content\": [
        {\"type\": \"text\", \"text\": \"请识别图片中的所有文字\"},
        {\"type\": \"image_url\", \"image_url\": {\"url\": \"data:image/jpeg;base64,$IMG_B64\"}}
      ]
    }],
    \"max_tokens\": 500
  }"
```

## 注意事项

- 接口是 `v1/text/chatcompletion_v2`，不是标准的 OpenAI-compatible `/v1/chat/completions`
- 图片用 base64 data URL 格式（`data:image/jpeg;base64,...`）
- 返回同样需要 `--compressed`
- 大图片可能导致 curl 被中断（exit 130），用 `background=true` + `notify_on_complete=true` 避免

## 被测模型

| 模型 | 结果 |
|------|------|
| `abab6.5s-chat` | ❌ 不支持图片 |
| `MiniMax-M3` | ✅ 支持 |
