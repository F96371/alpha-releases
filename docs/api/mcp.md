# Alpha MCP（让 AI 客户端读取 Alpha 数据）

Claude Code、Cursor、Cherry Studio、VS Code、Claude Desktop、Apifox 等支持 MCP 的客户端，可以通过 Alpha 桌面读取账号、会话、消息、搜索、联系人、群成员、朋友圈、公众号、收藏、年度画像和媒体链接，也可以发送文字消息（见下方「发送消息」）。不能改消息或导出文件。

## 准备

1. 在「设置 → 接口与集成 → 本机 HTTP API」设置访问令牌并启动（默认端口 5031）。详见 [HTTP API 文档](http-api.md)。
2. 同一页「让其他 AI 客户端使用 Alpha」会按当前端口生成配置，复制后把 `<访问令牌>` 换成你的令牌。

MCP 和 HTTP API 共用端口和令牌。Alpha 桌面需要保持打开；应用锁定时连接暂停。

- MCP 地址：`http://127.0.0.1:5031/mcp`
- 请求头：`Authorization: Bearer <访问令牌>`

## 客户端配置

**支持 HTTP 的客户端**（Cursor、Cherry Studio、VS Code、Claude Code 等）：

```json
{
  "mcpServers": {
    "alpha": {
      "type": "http",
      "url": "http://127.0.0.1:5031/mcp",
      "headers": { "Authorization": "Bearer <访问令牌>" }
    }
  }
}
```

**Claude Code 命令**：

```bash
claude mcp add --transport http alpha http://127.0.0.1:5031/mcp --header "Authorization: Bearer <访问令牌>"
```

**只能启动本地程序的客户端**（Claude Desktop 等）：使用安装目录里的 `resources\bin\alpha-mcp-stdio.exe` 转发到正在运行的 Alpha（设置页会显示本机实际路径）：

```json
{
  "mcpServers": {
    "alpha": {
      "command": "<安装目录>\\resources\\bin\\alpha-mcp-stdio.exe",
      "args": [],
      "env": {
        "ALPHA_MCP_DESKTOP_URL": "http://127.0.0.1:5031/mcp",
        "ALPHA_HTTP_TOKEN": "<访问令牌>"
      }
    }
  }
}
```

转发程序只接受本机回环地址，令牌不会发到其他主机。

**Apifox**（2.8.3+）：项目里新建 MCP 客户端，选 HTTP，填上面的地址；Auth 选 Bearer Token，填 Alpha 的访问令牌（不是 Apifox 自己的令牌）。

## 协议

- 传输：Streamable HTTP，只用 `POST /mcp`；响应为 `application/json`，不建立 SSE 流（`GET /mcp` 返回 405，客户端会自动改用 POST 响应），不签发会话 ID。
- 支持 `initialize`、`notifications/initialized`、`ping`、`tools/list`、`tools/call`。
- 每个工具都有 `inputSchema` 和 `outputSchema`；成功结果放在 `structuredContent`。输入参数不允许未声明的字段。
- 工具失败时 `isError=true`，错误代码和是否可重试写在 `content` 的文本 JSON 里。
- 结果是给 AI 读的精简版：联系人只列名字和 ID，朋友圈只列正文、地点、链接、点赞、评论与媒体序号，不带数据库原始列（`fields`）、原文 XML、头像链接和媒体密钥，空值和逐条重复的 `account_id` 也省掉。
- 消息只保留 `id`、`time`（本机时区）、`sender`（备注、群昵称、昵称，自己发的为“我”）、`sender_id`、`from_me`、`type`、`text` 和 `quote`。表情、引用、链接、文件、转账、聊天记录等结构化消息的 `text` 由数据所有者解析好的内容生成，例如“[表情]”“[链接] 标题 — 摘要 (网址)”；存储的 XML、CDN 地址、文件密钥和 AES 密钥都不会返回。同一会话的结果不在每条消息上重复 `conversation_id`；群成员的 `isAdmin`/`isOwner` 只在为真时出现。要看一个人的详情用 `wechat.contacts.get_contact_profile`，要看单条消息的原始字段用 `wechat.chat.get_message_raw`；需要完整原始记录时用 [HTTP API](http-api.md)，它不做精简。
- 请求体上限 1 MiB；需要整段扫描的工具（消息上下文、按日计数、年度画像、转账记录、聊天媒体等）单次最长 140 秒，其他 30 秒。

手工调用示例：

```bash
curl -s http://127.0.0.1:5031/mcp \
  -H "Authorization: Bearer $ALPHA_HTTP_TOKEN" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"wechat.chat.list_sessions","arguments":{"limit":20}}}'
```

## 参数约定

- `account`：可选。填 Alpha 账号 ID 或微信号；只有一个已选账号时可省略。
- `username`：会话的微信 ID（好友 wxid、群 `xxx@chatroom` 等），可先用 `wechat.chat.resolve_session` 或 `wechat.contacts.resolve_contact` 由名字查到。`session_id` 是它的别名。
- `limit`/`offset`：多数列表 `limit` 1–200。会话、消息、联系人、朋友圈、收藏和取证记录不传 `limit` 时每页 20 条（HTTP API 仍为 100）；`resolve_contact` 最多返回 20 个候选。
- 时间：`start_time`/`end_time` 为 Unix 秒；`date` 为 `YYYY-MM-DD`。
- 媒体工具返回的链接指向同一端口的 `/api/v1/media/...`，下载时同样带 `Authorization` 头。
- 不用令牌也能拿到媒体：结果里每个本地已核验的媒体都另存到 `<Alpha 用户数据目录>/api-media/<会话>/<images|videos|voices|files>/<sha256>.<扩展名>`，路径写在 `local_path`（同 WeFlow 的 `mediaLocalPath`），本机 AI 客户端可直接读取。图片还会作为 MCP `image` 内容直接附在结果里（每次最多 4 张，png/jpeg/gif/webp；超过 3.75 MB 的原图缩成长边 2048 的 JPEG 附上，原图仍在 `local_path`）；没附上的在 `not_inlined` 里写明原因。

## 工具列表（58 个只读工具，另有 2 个发送工具）

### 账号与目录 `wechat.core`

| 工具 | 说明 |
| --- | --- |
| `get_status` | 数据就绪状态与工具目录信息，不读消息内容 |
| `list_tools` | 列出只读工具目录，可按 `package` 过滤 |
| `list_accounts` | 账号列表 |
| `get_account_info` | 单个账号与数据库就绪信息 |

### 联系人 `wechat.contacts`

| 工具 | 说明 |
| --- | --- |
| `list_contacts` | 联系人、群、公众号；`keyword`、`include_friends/groups/officials` 过滤 |
| `resolve_contact` | 用模糊线索（`query`）找人、群或公众号，返回名字和 `username` |
| `get_contact_profile` | 一个人或群的详情（`username` 必填）：性别、签名、地区、添加方式与时间、标签、描述、共同群、好友验证；群另有群主、人数和公告。没记录的项不返回 |
| `get_group_members` | 群成员（`chatroomId` 必填），来自联系人数据库 |

### 聊天 `wechat.chat`

| 工具 | 说明 |
| --- | --- |
| `list_sessions` | 会话列表，带预览和分页 |
| `resolve_session` | 用模糊线索找会话 |
| `get_messages` | 读取一个会话的消息（`username` 必填），`order`、`cursor`/`offset` 分页 |
| `search_messages` | 全文搜索（`query` 必填），可按会话、发送人、时间、群聊/单聊过滤 |
| `list_search_senders` | 搜索结果中的发送人分布 |
| `get_message_around` | 某条消息前后的上下文（`username`、`anchor_id`，前后各最多 50 条） |
| `get_message_anchor` | 会话第一条或某一天的第一条消息（`kind`=`first`/`day`） |
| `get_daily_message_counts` | 某会话某月每天的消息数 |
| `get_message_raw` | 单条消息的原始字段 |
| `resolve_chat_history` | 展开合并转发的聊天记录（`server_id` 与所在会话 `username`） |
| `resolve_app_message` | 解析卡片、链接、小程序等消息（`server_id` 与所在会话 `username`） |

### 搜索（Alpha 扩展）

| 工具 | 说明 |
| --- | --- |
| `alpha.messages.search_page` | 跨会话分页搜索，字段与 HTTP `/api/v1/search` 相同，Base64 游标翻页 |
| `alpha.messages.resolve_search_hit` | 按搜索命中精确读取原消息及其媒体链接 |

### 朋友圈 `wechat.moments`

| 工具 | 说明 |
| --- | --- |
| `get_self_info` | 当前账号的朋友圈身份 |
| `list_timeline` | 时间线，按发布者、关键词、时间筛选；每条带 `media`（`idx`、类型、尺寸） |
| `search_moments` | 按文字或发布者搜索 |
| `list_users` | 发布者与动态数 |
| `get_media_url` | 朋友圈图片下载链接（`post_id` + `idx`） |
| `get_article_thumb_url` | 分享文章的缩略图链接 |
| `get_remote_video_url` | 朋友圈视频或实况照片链接 |
| `get_local_video_url` | 已缓存的朋友圈视频链接 |

### 公众号 `wechat.biz`

| 工具 | 说明 |
| --- | --- |
| `list_accounts` | 公众号与服务号 |
| `get_messages` | 某公众号的消息 |
| `get_pay_records` | 微信支付公众号里的转账/收付款记录 |
| `get_proxy_image_url` | 公众号图片代理链接 |

### 年度画像 `wechat.analytics`

| 工具 | 说明 |
| --- | --- |
| `get_wrapped_meta` | 年度画像目录 |
| `get_wrapped_card` | 单张年度画像卡片（`card_id`） |
| `get_wrapped_annual` | 年度画像完整数据 |

### 媒体 `wechat.media`

| 工具 | 说明 |
| --- | --- |
| `get_avatar_url` | 联系人头像 |
| `get_chat_image_url` | 聊天图片 |
| `get_chat_emoji_url` | 聊天表情 |
| `get_chat_video_thumb_url` | 聊天视频封面 |
| `get_chat_video_url` | 聊天视频 |
| `get_chat_voice_url` | 聊天语音文件（不做语音转文字） |
| `get_decrypted_resource_url` | 按 MD5 取已解密资源 |
| `get_proxy_image_url` | 远端图片代理链接 |
| `get_favicon_url` | 网站图标 |

聊天媒体工具用 `username` + `message_id`（或 `server_id`）定位消息；合并转发里的媒体另用 `record_index`/`record_index_path`。

### 移动端组合查询 `wechat.mobile`

为对话式查询准备的组合工具，一次返回较紧凑的结果：

| 工具 | 说明 |
| --- | --- |
| `get_overview` | 账号与会话概览 |
| `get_home_snapshot` | 首页快照 |
| `resolve_target` | 一个线索同时查联系人、会话、朋友圈用户、公众号 |
| `search_context` | 搜消息并附带联系人/会话信息 |
| `search_chat` | 搜聊天消息；命中上下文用 `get_chat_context` |
| `get_chat_context` | 最近消息、某条消息附近或某一天的上下文 |
| `get_session_bundle` | 会话信息、消息与可选统计 |
| `search_moments` | 搜朋友圈，附带媒体引用 |
| `get_media_links` | 聊天、朋友圈、头像、链接或表情的媒体链接 |
| `get_message_media_bundle` | 一条消息可能包含的全部媒体链接 |
| `get_analytics` | 按指标返回统计 |

### 其他（Alpha 扩展）

| 工具 | 说明 |
| --- | --- |
| `wechat.favorites.list_items` | 收藏，含嵌套内容与来源 |
| `wechat.forensics.list_records` | 取证记录表（固定白名单表，不能执行任意 SQL） |

### 发送消息

| 工具 | 说明 |
| --- | --- |
| `wechat.chat.send_text` | 给 `username`（`filehelper`、`wxid_…` 好友或 `…@chatroom` 群）发一条文字 `text`；可选 `account`、`wait_seconds`（0–120，默认 30）、`idempotency_key` |
| `wechat.chat.get_send_status` | 按 `run_id` 查询发送结果，只读取不重发 |

和聊天输入框、HTTP `/api/v1/messages/send-text` 是同一条发送流程：需要激活「发送文字消息」，并在「高级功能 → 发送文字消息」里连接一次微信；发出后写入电脑微信本地聊天记录。结果的 `state` 为 `delivered`、`sending`（稍后查询）、`failed`、`canceled`、`unknown`（结果未确认，先到微信核对，不要重复发送）或 `not-submitted`。`username` 必须是微信内部 ID（不是昵称、备注或微信号），先用 `wechat.contacts.resolve_contact` 查。失败时 `isError=true`，`structuredContent` 里 `code` 是原因、`message` 是中文说明和下一步（文本内容第一行也是这句），例如 `SEND_CONNECTION_REQUIRED`（还没连接微信）、`SEND_TARGET_NOT_FOUND`（联系人里找不到，`candidates` 给出可能的联系人）、`SEND_TARGET_UNSUPPORTED`（自定义微信 ID 的联系人暂不支持）、`MATERIAL_TEXT_SEND_RECOVERY_REQUIRED`（上一条还没结果）。未完成的结果（`sending`、`unknown` 等）也带 `message`。字段含义与 [HTTP API 文档](http-api.md#发送文字消息) 相同。

发消息会以你的微信身份对外发出，建议在 AI 客户端里把这个工具设为每次调用前确认。

表中省略了包前缀，完整名称是"包名.工具名"，例如 `wechat.chat.search_messages`。参数细节以 `tools/list` 返回的 `inputSchema` 为准。
