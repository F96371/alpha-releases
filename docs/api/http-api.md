# Alpha 本机 HTTP API

Alpha 桌面可以在本机开一个 HTTP 端口，让你自己的脚本、ChatLab 或其他工具读取聊天、联系人、朋友圈数据，订阅新消息，把会话导出成文件，或发送文字消息。同一端口还提供 MCP 入口，供 AI 客户端使用，见 [MCP 文档](mcp.md)。

## 开启

1. 打开「设置 → 接口与集成 → 本机 HTTP API」。
2. 填端口（默认 `5031`，可选 1024–65535）和访问令牌（32–256 个可见 ASCII 字符，可随机生成；加密保存，之后可在这里查看和复制）。
3. 点「保存并启动」。需要实时推送时，再打开同一页的「主动推送」并应用；下方的「消息推送」会给出推送地址和会话过滤。

说明：

- 只监听 `127.0.0.1`，局域网和外网都访问不到；Alpha 桌面需要保持打开。
- 应用锁定时服务暂停，解锁后自动恢复。
- 端口被占用时启动失败并提示，不会换端口。WeFlow 的 API 默认也用 `5031`，两个同时开时请给其中一个换端口。
- 「停止服务」会立即关闭端口和现有连接。

## 调用约定

**基础地址**：`http://127.0.0.1:5031`（端口以设置为准）

**鉴权**：除健康检查外，所有请求都要在请求头带上令牌：

```
Authorization: Bearer <访问令牌>
```

客户端没法设置请求头时，也可以像 WeFlow 一样把令牌接在地址后面：`?access_token=<访问令牌>`（所有接口都认；地址里的令牌容易留在日志和历史记录里，能用请求头就用请求头）。不支持放在请求体里。带 `Origin` 头的浏览器跨站请求，以及 Host 不是 `127.0.0.1`/`localhost` 的请求都会被拒绝。

**示例**

```powershell
$h = @{ Authorization = "Bearer $env:ALPHA_HTTP_TOKEN" }
Invoke-RestMethod 'http://127.0.0.1:5031/api/v1/accounts' -Headers $h
Invoke-RestMethod 'http://127.0.0.1:5031/api/v1/messages?talker=wxid_xxx&limit=100' -Headers $h
```

```bash
curl -H "Authorization: Bearer $ALPHA_HTTP_TOKEN" "http://127.0.0.1:5031/api/v1/sessions?limit=20"
```

**参数**

- GET 的参数放在查询字符串里；同名 POST 接口接收 JSON 请求体，字段相同。中文等字符请做 URL 编码。
- 账号用 `account` 或 `account_id`，取值为 `/api/v1/accounts` 返回的账号 ID，也可以用微信号。只有一个已选账号时可以省略；有多个账号时必须指定。
- 别名（如 `query`/`keyword`、`username`/`talker`）只传其中一个，同时传会报错。
- 时间参数接受 Unix 秒、Unix 毫秒或 `YYYYMMDD`（按本机时区；结束日期包含当天全天）。起止都包含边界。
- 字段名保持原样，不做 camelCase/snake_case 转换。

**等待时间**：会话、消息、搜索、群成员、朋友圈、发送消息和 MCP 最长 150 秒；会话导出 180 秒；朋友圈导出 480 秒；其他 45 秒；推送流不限时。超时返回 504，只表示不再等待，导出类操作可能仍在写文件。请求体不超过 1 MiB。

**错误**：返回 JSON，包含错误 `code`。端口本身返回的错误：

| HTTP | code | 含义 |
| --- | --- | --- |
| 401 | `HTTP_API_UNAUTHORIZED` | 没带令牌或令牌不对 |
| 403 | `HTTP_API_ORIGIN_DENIED` / `HTTP_API_HOST_DENIED` | 浏览器跨站请求或非本机 Host |
| 404 | `HTTP_API_ROUTE_NOT_FOUND` | 路径不存在或未开放 |
| 405 | `HTTP_API_METHOD_INVALID` | 方法不支持（只支持 GET/POST，`/mcp` 只支持 POST） |
| 400 | `HTTP_API_PUSH_ACCOUNT_REQUIRED` / `HTTP_API_PUSH_QUERY_INVALID` / `HTTP_API_PUSH_FILTER_RESERVED` | 订阅推送没带 `accountId`、查询参数重复，或自带了过滤字段 |
| 409 | `HTTP_API_PUSH_DISABLED` | 订阅推送但桌面没开「主动推送」 |
| 413 | `HTTP_API_BODY_TOO_LARGE` | 请求体超过 1 MiB |
| 423 | `HTTP_API_OPERATION_BLOCKED` | 应用已锁定或需要先更新 |
| 502 | `HTTP_API_UPSTREAM_FAILED` / `HTTP_API_UPSTREAM_TRUNCATED` | 内部服务异常或连接中断 |
| 504 | `HTTP_API_REQUEST_TIMEOUT` | 超过上面的等待时间 |

数据接口自身的错误（参数不合法、账号无法确定、超出读取上限等）也以 JSON 返回具体 `code`，不会用空结果代替错误。

## 接口一览

| 分组 | 方法 | 路径 | 用途 |
| --- | --- | --- | --- |
| 账号与状态 | GET | `/health`、`/api/v1/health` | 存活检查，免令牌 |
| | GET | `/api/v1/accounts` | 账号列表与当前选中账号 |
| 会话 | GET/POST | `/api/v1/sessions` | 会话列表，支持关键词、排序、分页 |
| | GET | `/api/v1/sessions/{session_id}/messages` | 按会话拉取 ChatLab 格式完整文档（增量同步） |
| 消息 | GET/POST | `/api/v1/messages` | 读取一个会话的消息，可加关键词、时间范围、媒体链接 |
| | GET/POST | `/api/v1/search` | 跨会话全文搜索 |
| 联系人与群 | GET/POST | `/api/v1/contacts` | 联系人、群、公众号；`usernames` 按精确 ID 批量查（GET 逗号分隔，最多 2000 个） |
| | GET/POST | `/api/v1/group-members` | 群成员，可附带发言数 |
| 导出 | GET | `/api/v1/exports/formats` | 支持的导出格式 |
| | POST | `/api/v1/exports` | 把一个会话导出到本机文件 |
| 媒体下载 | GET | `/api/v1/media/{opaque_id}` | 下载消息/朋友圈接口返回的媒体，语音可转 WAV |
| | GET | `/api/v1/media/by-md5/{md5}` | 按 MD5 读取已缓存图片 |
| | GET | `/api/v1/media/proxy/{token}` | 朋友圈等远端媒体的短期链接 |
| 朋友圈 | GET | `/api/v1/sns/timeline` | 朋友圈时间线 |
| | GET | `/api/v1/sns/usernames` | 发布者与动态数 |
| | GET | `/api/v1/sns/export/stats` | 导出前统计 |
| | POST | `/api/v1/sns/export` | 导出朋友圈（文件正文或写入目录） |
| | GET/POST | `/api/v1/sns/media/proxy` | 解密一条朋友圈图片/视频 CDN 地址 |
| 实时推送 | GET | `/api/v1/push/status` | 推送配置与容量 |
| | GET/POST | `/api/v1/push/messages` | SSE 订阅新消息和撤回 |
| 发送消息 | POST | `/api/v1/messages/send-text` | 以当前微信账号发一条文字消息 |
| | GET/POST | `/api/v1/messages/send-text/status` | 按 `run_id` 查询发送结果 |
| MCP | POST | `/mcp` | AI 客户端入口（只读工具和两个发送工具），见 [MCP 文档](mcp.md) |

各接口的参数和返回字段见下文「使用要点」；字段名以实际返回为准。

## 使用要点

### 会话与消息

- `/api/v1/sessions`：`limit` 1–200。返回 `items`、`count`（本页条数）、`total`（全部匹配数）、`hasMore`、`nextOffset`；`sessions[]` 是 WeFlow 风格的同一批数据（`username/displayName/sessionType/lastTimestamp/unreadCount` 等，`sessionType` 为 `private/group/official/other`）。不传 `order` 保持微信原顺序；`asc`/`desc` 按最后活动时间排序。`unreadCount`、`isMuted` 来自微信本身，取不到时为 `null`。
- `/api/v1/messages`：会话用 `username`/`talker`/`session_id`/`conversation_id` 之一指定（必填）。`limit` 1–10000，用 `nextOffset` 翻页。
  - 传非空 `keyword` 时改为在该会话里做全文搜索（使用微信自带的索引），返回搜索命中字段。
  - `media=true` 时为图片、语音、视频生成 `/api/v1/media/...` 下载链接；和 `keyword` 同时使用时 `limit` 最多 128，返回完整消息。
  - 单次读取有上限（约 5 万行或 120 秒），超出时报错而不是悄悄截断，请缩小时间范围或分页。
- `/api/v1/search`：跨会话搜索，返回 `id/conversation_id/sender_id/sent_at_unix_ms/text/source_type`；`group_by_conversation=true` 时每个会话返回最新一条命中和 `match_count`。用返回的 `next_cursor` 翻页。长整数（时间、ID）以字符串返回，避免精度丢失。

### ChatLab

在 ChatLab 的「设置 → API → 数据源」里添加：地址填 `http://127.0.0.1:5031`（ChatLab 会自己补上 `/api/v1`），Token 填 Alpha 的访问令牌。ChatLab 不需要打开「主动推送」。

- `GET /api/v1/sessions?format=chatlab` 列出会话（原始微信会话 ID），最近活跃的在前；每页 `limit` 条（默认 100，最多 10000），`page.hasMore` 为真时用 `page.nextCursor` 作 `cursor` 取下一页，对应 ChatLab 里的「加载更多」。
- `GET /api/v1/sessions/{session_id}/messages` 返回 ChatLab 0.0.2 文档。`since` 不含该秒、`end` 含该秒，`limit` 最多 5000。翻页只看 `sync`：`hasMore=true` 时把 `nextSince` 作为下一次的 `since`，直到 `hasMore=false`；每页都结束在整秒上，不会漏掉或重复同一秒里的消息。最后一页的 `nextSince` 是最新一条消息的时间，下次增量从它开始。
- 同时发来的多个请求会排队依次处理（ChatLab 选中多个会话时会一起开始拉取）。

### 群成员

成员来自微信联系人数据库，不是"在群里说过话的人"。`includeMessageCounts=true` 会额外统计每人发言数（最多扫描 10 万条，可能较慢）。

### 导出会话

`POST /api/v1/exports`：

```json
{
  "account_id": "<账号 ID>",
  "conversation_id": "<会话 ID>",
  "format": "xlsx",
  "destination_path": "D:\\exports\\chat.xlsx",
  "include_media": false
}
```

- `format`：`json`、`markdown`、`chatlab-json`、`chatlab-jsonl`、`html`、`txt`、`xlsx`、`arkme-json`、`sql`、`weclone`。
- `destination_path` 是本机文件的绝对路径，文件写在你的电脑上，不通过 HTTP 下载。
- 可选 `start_time_unix_ms`、`end_time_unix_ms`（Unix 毫秒，字符串）限定时间范围。
- 返回实际路径、消息数和媒体统计（`media_included`/`media_failed`）。超时后先检查目标文件，再决定是否重试。

### 媒体

- 消息和朋友圈接口返回的媒体链接都在本端口上，下载时同样要带 `Authorization` 头。
- `/api/v1/media/{opaque_id}` 默认返回原始字节，微信语音是 `audio/silk`；加 `?format=wav` 转成 WAV（只解码，不做语音转文字），响应头 `Content-Location` 是转换后文件的链接。
- 朋友圈媒体链接是短期的，过期后重新调用时间线接口获取。
- `/api/v1/sns/media/proxy` 直接解密一条朋友圈 CDN 地址，推荐 POST：`{"url":"https://...","key":"<十进制字符串>"}`，或把 key 放在 `X-Alpha-Sns-Key` 请求头。key 用字符串，避免 64 位整数被 JSON 数字截断。

### 朋友圈

- 时间线：按 `usernames`（逗号分隔，最多 100 个）、`keyword`、`start`/`end` 筛选，`limit` 1–200。为了不全量扫描，`total` 为 `null`，是否还有下一页看 `hasMore`。`media=1` 时为图片/视频/实况照片生成下载链接。
- 导出：`format` 为 `json`、`html`、`markdown`、`arkme-json`。不传 `outputDir` 时直接返回文件正文（不接受 `limit`/`offset`，导出全部筛选结果）；传 `outputDir`（已存在的绝对目录）时在其中新建 `alpha-sns-<UUID>` 子目录，可用 `exportMedia`（或 `exportImages`/`exportLivePhotos`/`exportVideos` 单独选择）一并保存媒体，不会覆盖已有文件。

### 实时推送

先在设置里打开「主动推送」，然后：

```bash
curl -N -H "Authorization: Bearer $ALPHA_HTTP_TOKEN" "http://127.0.0.1:5031/api/v1/push/messages?accountId=<账号 ID>"
```

打开「主动推送」并保存后，设置页的「消息推送」会给出当前账号的推送地址（可一键复制，每个账号不同）和推送会话过滤。复制到的地址和 WeFlow 一样在末尾带 `access_token`，订阅方直接连接即可；界面上令牌默认遮住。上面的 curl 是改用请求头的写法。

- `accountId` 必填，`sessionId` 可选（只订阅一个会话）。
- **推送会话过滤**（设置里按账号保存，与 WeFlow 相同）：「全部会话」「白名单」（只推送名单内的会话）「黑名单」（名单内的会话不推送；和新消息通知一样，默认把微信里设了免打扰的会话也算进黑名单，之后新设免打扰的会话自动加入，可单个移出，也可关掉「免打扰会话不推送」）。过滤由 Alpha 强制执行，订阅方不能在请求里自带或放宽它（带 `filterMode`/`filterSessions` 返回 400 `HTTP_API_PUSH_FILTER_RESERVED`）；`sessionId` 指向被过滤排除的会话时返回 403 `push.session_filtered`。白名单为空时连接保持但不推送任何会话。
- 在设置里修改过滤会立即生效：该账号已连接的订阅收到 `stream.error`（`push.filter_changed`）后断开，重连后按新过滤推送。
- 事件：`stream.ready`、`message.new`、`message.revoke`、`stream.error`。消息字段：`event/eventId/accountId/sessionId/sessionType/rawid/avatarUrl/sourceName/content/timestamp`，群聊另有 `groupName`；`timestamp` 为 Unix 秒。`sessionType` 为 `private/group/official/other`。私聊和公众号的 `sourceName` 是会话名称；群聊的 `sourceName` 是发言成员的微信 ID（群名在 `groupName`），需要昵称时用 `/api/v1/group-members` 对应。
- 只推送连接后到达的消息，不回放历史，不支持 `Last-Event-ID` 续传；断线期间的消息请用 `/api/v1/messages` 补拉。按 `event + rawid` 去重。
- 最多 8 个订阅者；每轮只读取通过过滤、且最近 60 秒内有新活动的会话，最多 128 个（账号的会话总数不受此限，超出时返回 `push.session_limit`，可用白名单缩小范围）。收到 `stream.error` 后连接会关闭，需要重连。

### 发送文字消息

以登录的微信账号给文件传输助手、好友或群发一条文字，和聊天输入框走同一条发送流程：同样记录未确认的发送，发出后同样写入电脑微信的本地聊天记录，聊天页能看到。

**准备**：需要激活「发送文字消息」，并在「高级功能 → 发送文字消息」里连接一次微信（Alpha 运行期间一直有效；重新连接或连接公众号导出后需要再连一次）。没有连接时返回 409 `SEND_CONNECTION_REQUIRED`。

```bash
curl -s http://127.0.0.1:5031/api/v1/messages/send-text \
  -H "Authorization: Bearer $ALPHA_HTTP_TOKEN" -H "Content-Type: application/json" \
  -d '{"username":"filehelper","text":"来自脚本的消息"}'
```

请求体（JSON，不允许多余字段）：

| 字段 | 说明 |
| --- | --- |
| `username` | 接收方的微信内部 ID：`filehelper`、`wxid_` 开头的好友或 `xxx@chatroom` 群，不是昵称、备注或微信号。`talker`、`session_id` 是别名，只传一个。不知道 ID 时先用 `/api/v1/contacts?keyword=名字` 查 |
| `text` | 正文，最多 10000 UTF-8 字节 |
| `account` / `account_id` | 可选，规则同上 |
| `wait_seconds` | 可选，等待结果的秒数 0–120，默认 30 |
| `idempotency_key` | 可选，16–128 个可见 ASCII 字符。同一个 key 重复调用返回第一次的结果，不会再发；换了正文会返回 409 `SEND_IDEMPOTENCY_CONFLICT`。只在 Alpha 运行期间记住 |

返回 `run_id`、`account_id`、`username`、`state`，以及可能有的 `error_code`、`provider_message_id`、`local_record`（`state` 为 `written`/`failed`/`unknown`）和 `message`（中文说明这个结果的含义和下一步，干净送达时没有）。`state` 的含义：

| state | HTTP | 含义 |
| --- | --- | --- |
| `delivered` | 200 | 微信已确认送达 |
| `sending` | 202 | 等待时间到了还没结果，稍后用 `run_id` 查询 |
| `failed` / `canceled` | 200 | 没有发出 |
| `unknown` | 200 | 结果未确认。**先到微信里核对，不要重复发送**；可以用查询接口再读一次 |
| `not-submitted` | 200 | 查询确认这条没有发出 |

查询：`GET /api/v1/messages/send-text/status?run_id=<run_id>`（或 POST 同名字段，可带 `account`）。查询只读取原结果，不会重发。

**拒绝发送时**（没有发出任何消息）返回 `{"error":{"code":"…","message":"…"}}`，`message` 是中文原因和下一步。常见情况：

| HTTP | code | 含义 |
| --- | --- | --- |
| 409 | `SEND_CONNECTION_REQUIRED` | 还没在 Alpha 里连接微信（重启或重新连接后需要再连） |
| 404 | `SEND_TARGET_NOT_FOUND` | 当前账号的联系人里找不到这个 `username`（例如填了错的 wxid、昵称或微信号）。`candidates` 给出按关键词找到的可能联系人（`username`、`display_name`、`alias`） |
| 400 | `SEND_TARGET_UNSUPPORTED` | 联系人的 ID 是自定义的（不以 `wxid_` 开头），暂不支持发送 |
| 409 | `MATERIAL_TEXT_SEND_RECOVERY_REQUIRED` | 上一条还没结果，见下 |
| 403 | 授权相关 code | 「发送文字消息」未激活 |
| 423 | `HTTP_API_OPERATION_BLOCKED` | 应用已锁定、正在更新或退出 |

发送前会先在联系人里核对接收方；联系人数据暂时读不到时不拦截格式正确的 ID，交给微信判断。

**限制**：同一账号同时只能有一条未确认的发送。上一条还没结果（包括聊天页里待确认的那条）时返回 409 `MATERIAL_TEXT_SEND_RECOVERY_REQUIRED`，`pending` 里给出那条的 `run_id`、`username`、`created_at_unix_ms` 和 `sending`（是否仍在发送）。到聊天页或高级功能核对处理后再发。未激活返回 403。

## 安全提示

- 令牌等同于你聊天数据的读取权限和以你的微信身份发消息的权限，不要写进公开仓库、截图或分享给他人。怀疑泄露时在设置里换一个新令牌。
- 本服务只在本机可用；不要用端口转发、内网穿透把它暴露到网络上。
