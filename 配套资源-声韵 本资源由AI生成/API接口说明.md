# 腾讯元器平台 API 接口说明

## 概述

「声韵」智能体基于腾讯元器平台（yuanqi.tencent.com）开发，使用其提供的标准化API接口进行通信。

## API 基础信息

| 项目 | 值 |
|------|------|
| API地址 | `https://yuanqi.tencent.com/openapi/v1/agent/chat/completions` |
| 请求方式 | POST |
| 认证方式 | Bearer Token (AppKey) |
| 数据格式 | JSON |
| 流式响应 | SSE (Server-Sent Events) |

## 认证

### 请求头

```
Content-Type: application/json
Authorization: Bearer <AppKey>
```

其中 `<AppKey>` 为腾讯元器平台的 AppKey。

## 请求格式

### 请求体结构

```json
{
    "assistant_id": "2049083180719277120",
    "user_id": "web_xxxxxx",
    "stream": true,
    "messages": [
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": "用户输入的文字"
                }
            ]
        }
    ]
}
```

### 参数说明

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| assistant_id | string | 是 | 助手ID（填AppID） |
| user_id | string | 是 | 用户ID，可自定义 |
| stream | bool | 否 | 是否流式返回，默认false |
| messages | list | 是 | 会话内容，最多40条 |
| messages[n].role | string | 是 | 'user' 或 'assistant' |
| messages[n].content | list | 是 | 内容列表，支持text和image_url |

## 响应格式

### 流式响应 (stream: true)

SSE格式，每条消息格式：
```
data: {"choices":[{"delta":{"content":"文本片段"},"index":0}]}
data: [DONE]
```

### 非流式响应 (stream: false)

```json
{
    "choices": [
        {
            "message": {
                "role": "assistant",
                "content": "完整的回复文本"
            }
        }
    ]
}
```

## SSE 流式解析示例

### JavaScript

```javascript
async function chat(message) {
    const response = await fetch(API_URL, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${APP_KEY}`
        },
        body: JSON.stringify({
            assistant_id: APP_ID,
            user_id: USER_ID,
            stream: true,
            messages: [{ role: 'user', content: [{ type: 'text', text: message }] }]
        })
    });

    const reader = response.body.getReader();
    const decoder = new TextDecoder();

    while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        const chunk = decoder.decode(value);
        const lines = chunk.split('\n');

        for (const line of lines) {
            if (line.startsWith('data: ')) {
                const data = line.slice(6);
                if (data === '[DONE]') return;

                const parsed = JSON.parse(data);
                const content = parsed.choices?.[0]?.delta?.content;
                if (content) {
                    console.log(content); // 打字机效果
                }
            }
        }
    }
}
```

### Python

```python
import requests
import json

def chat(message):
    headers = {
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {APP_KEY}'
    }

    data = {
        'assistant_id': APP_ID,
        'user_id': 'user_001',
        'stream': True,
        'messages': [
            {'role': 'user', 'content': [{'type': 'text', 'text': message}]}
        ]
    }

    response = requests.post(API_URL, headers=headers, json=data, stream=True)

    for line in response.iter_lines():
        if line.startswith('data: '):
            data = line[6:]
            if data == '[DONE]':
                break

            parsed = json.loads(data)
            content = parsed.get('choices', [{}])[0].get('delta', {}).get('content', '')
            if content:
                print(content, end='', flush=True)
```

## Socket.IO 协议 (WebSocket方式)

### 连接地址
```
wss://wss.yuanqi.tencent.com/v2/chat/conn/?language=zh-CN&EIO=4&transport=websocket
```

### 协议格式
- 消息前缀 `42` 表示带事件名的消息
- 格式：`42["event_name", {data}]`

### 发送消息
```javascript
const payload = {
    "Type": "message",
    "MessageId": "usr_xxx",
    "ConversationId": "",
    "BotAppKey": APP_ID,
    "Content": {"Type": "text", "Text": "用户消息"}
};
socket.send('42' + JSON.stringify(["message", payload]));
```

### 接收事件
- `text.delta` - 流式文本片段
- `response.completed` - 回复完成

## 注意事项

1. **CORS限制**：浏览器直接调用可能遇到跨域问题，可使用代理服务器
2. **Token有效期**：AppKey和Token可能有有效期限制
3. **并发限制**：平台可能有API并发限制
4. **内容审核**：AI输出可能经过内容安全审核

## 错误处理

常见错误码：

| 错误码 | 说明 | 处理方式 |
|--------|------|----------|
| 401 | 认证失败 | 检查AppKey是否正确 |
| 403 | 无权限 | 检查AppID是否有效 |
| 429 | 请求过于频繁 | 降低请求频率 |
| 500 | 服务器错误 | 重试或联系平台支持 |

## 获取API密钥

1. 登录腾讯元器平台 (https://yuanqi.tencent.com)
2. 进入智能体管理后台
3. 在设置中找到 AppID 和 AppKey
4. 更新前端代码中的配置

## 免责声明

本API文档仅供参考，腾讯元器平台可能会更新API规范。建议开发者关注官方文档获取最新信息。
