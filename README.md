# Phicomm R1 Voice Assistant (multi-source news)

基于 [ring1012/r1-dummy](https://github.com/ring1012/r1-dummy) 斐讯 R1 音箱 AI 方案的自部署分支,
新增**新闻多源**支持。包含两个子项目:

| 目录 | 说明 |
|------|------|
| `r1-py/` | Cloudflare Python Worker(AI 工具服务)。`playNews` 支持多新闻源,新增 `/r1/admin/news` 管理 API |
| `r1-admin-page/` | EdgeOne 管理后台。服务配置页新增 **新闻** 卡片,可视化管理新闻源 |

## 新闻多源的工作方式

音箱语音请求 → r1-py(LLM function calling)→ `playNews` 工具,按以下优先级解析新闻源:

1. 语音指定的源名称(如"播放经济之声新闻")→ KV `providers` 的 `news` 列表
2. `x-r1-news` 请求头(base64 JSON `{"provider": "..."}`,与音乐源机制一致)
3. 设备 KV `device:{serial}` 的 `newsConfig.endpoint` 默认源
4. 内置央广中国之声(chanId=64,开箱即用)

新闻源两种形态:

- `chanId` 型:复用内置央广接口,只需登记对应频道 chanId;
- `endpoint` 型:自建/第三方媒体服务,实现 `GET {endpoint}/search?keyword=新闻`,
  返回 `{"count": N, "musicinfo": [{"id","title","artist","url"}]}`,`url` 为可直接播放的音频。

## 部署

### r1-py

```bash
cd r1-py
uv sync
npx wrangler kv namespace create R1     # 并把 id 填入 wrangler.jsonc
npx wrangler secret put ADMIN_TOKEN     # 管理 API 鉴权密钥
npx wrangler deploy
```

然后设置音箱 AI endpoint 指向你的 Worker(恩山帖 / 后台 AI 配置页)。

### r1-admin-page

```bash
cd r1-admin-page
npm install && npm run build
# 按 README 部署到 EdgeOne Pages
```

后台 → 服务配置 → **新闻** 标签:填 Worker 地址 + ADMIN_TOKEN,即可添加/删除新闻源、设置默认源。

详细 API 用法见 `r1-py/README.md` 的"新闻多源配置"一节。
