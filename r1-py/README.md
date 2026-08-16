# LangChain Cloudflare Worker Example

This example demonstrates how to use `langchain-cloudflare` with Cloudflare Python Workers, using the Workers AI, Vectorize, and D1 bindings directly for optimal performance.

## 新闻多源配置(本分支新增)

`playNews` 工具支持多种新闻源,解析优先级:

1. 语音指定的源名称(如"播放经济之声新闻")→ 匹配 KV `providers` 中的 `news` 列表
2. `x-r1-news` 请求头(base64 JSON `{"provider": "..."}`,与音乐源机制一致)
3. 设备 KV `device:{serial}` 中的 `newsConfig.endpoint` 默认源
4. 内置央广中国之声(chanId=64,开箱即用)

新闻源有两种形态,登记在 KV `providers` 键中:

```json
{
  "news": [
    {"provider": "中国之声", "chanId": "64"},
    {"provider": "经济之声", "chanId": "xx"},
    {"provider": "澎湃新闻", "endpoint": "https://your-news-service"}
  ]
}
```

- `chanId` 型:复用内置央广接口,只需填央广 App 接口对应频道的 chanId。
- `endpoint` 型:自建/第三方媒体服务,需实现统一搜索协议
  `GET {endpoint}/search?keyword=新闻`,返回
  `{"count": N, "musicinfo": [{"id", "title", "artist", "url"}]}`,`url` 必须是音箱可直接播放的音频地址。

### 管理 API

```bash
# 设置鉴权密钥(部署后执行一次)
npx wrangler secret put ADMIN_TOKEN

# 查看已配置的新闻源与某设备的默认源
curl -H "x-r1-admin-token: XXX" https://your-worker/r1/admin/news
# 指定设备: 再加 -H "x-r1-serial: 设备序列号"

# 添加/删除新闻源
curl -X POST -H "x-r1-admin-token: XXX" -H "Content-Type: application/json" \
  -d '{"action":"add","provider":"澎湃新闻","endpoint":"https://your-news-service"}' \
  https://your-worker/r1/admin/news
curl -X POST -H "x-r1-admin-token: XXX" -H "Content-Type: application/json" \
  -d '{"action":"remove","provider":"澎湃新闻"}' https://your-worker/r1/admin/news

# 设置/清除某台设备(默认 DEFAULT)的默认新闻源
curl -X POST -H "x-r1-admin-token: XXX" -H "Content-Type: application/json" \
  -d '{"action":"set_default","provider":"澎湃新闻","endpoint":"https://your-news-service"}' \
  https://your-worker/r1/admin/news
```

配套的 [r1-admin-page](https://github.com/ring1012/r1-admin-page) 后台在"服务配置"页新增了 **新闻** 卡片,填入 Worker 地址和 ADMIN_TOKEN 即可视化管理。

## Features

- Basic chat completion with Workers AI
- Structured output with Pydantic models
- Tool calling
- Multi-turn conversations
- `create_agent` pattern (requires langchain>=0.3.0)
- Vectorize operations (insert, search, delete)
- D1 database operations

## Prerequisites

1. Cloudflare account with Workers, AI, Vectorize, and D1 access
2. Python 3.12+
3. [uv](https://docs.astral.sh/uv/) package manager
4. [pywrangler](https://pypi.org/project/workers-py/) for local development

## Setup

1. Install dependencies:
   ```bash
   uv sync
   ```

2. Create a Vectorize index (if needed):
   ```bash
   npx wrangler vectorize create langchain-test-persistent --dimensions 768 --metric cosine
   ```

3. Create a D1 database (if needed):
   ```bash
   npx wrangler d1 create test-db
   ```

4. Update `wrangler.jsonc` with your database ID

## Running Locally

```bash
uv run pywrangler dev
```

## Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/` | GET | API documentation |
| `/chat` | POST | Basic chat completion |
| `/structured` | POST | Structured output extraction |
| `/tools` | POST | Tool calling |
| `/multi-turn` | POST | Multi-turn conversation |
| `/agent-structured` | POST | Agent with structured output |
| `/agent-tools` | POST | Agent with tools |
| `/vectorize-insert` | POST | Insert into Vectorize |
| `/vectorize-search` | POST | Search Vectorize |
| `/vectorize-delete` | POST | Delete from Vectorize |
| `/vectorize-info` | GET | Vectorize index info |
| `/d1-health` | GET | D1 health check |
| `/d1-create-table` | POST | Create D1 table |
| `/d1-insert` | POST | Insert into D1 |
| `/d1-query` | POST | Query D1 |
| `/d1-drop-table` | POST | Drop D1 table |

## Example Usage

```bash
# Chat
curl -X POST http://localhost:8787/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello!"}'

# Structured output
curl -X POST http://localhost:8787/structured \
  -H "Content-Type: application/json" \
  -d '{"text": "Acme Corp announced a partnership with TechGiant."}'

# Tool calling
curl -X POST http://localhost:8787/tools \
  -H "Content-Type: application/json" \
  -d '{"message": "What is the weather in San Francisco?"}'
```
