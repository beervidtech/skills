# vidgate CLI 命令参考

全局 flag（可前可后）：`--json`（恒输出 envelope）/ `--token` / `--base-url` / `--timeout <ms>` / `--help`。
凭据优先级：`--token` > `VIDGATE_API_TOKEN` 环境变量 > `~/.config/vidgate/config.json`。
退出码：0 成功 / 1 本地 / 2 认证 / 3 业务拒绝 / 4 限流 / 5 上游 / 6 内部。

## 凭据

| 命令 | 说明 |
|---|---|
| `vidgate auth login --token <t>` | 保存凭据（config 文件 0600）；`--base-url` 同传可固化网关地址 |
| `vidgate auth status` | 查看生效 token（脱敏）/baseUrl 及来源 |
| `vidgate auth logout` | 清除本地凭据 |
| `vidgate version` | 打印版本 |

## 只读命令（CLI ≥ 0.0.x）

| 命令 | 端点 | 说明 |
|---|---|---|
| `vidgate seats list` | GET /v1/seats | 坐席列表：slotNo/status/绑定账号（TT=businessId，TTS=creatorUserOpenId）/authStatus |
| `vidgate videos list [--cursor <c>] [--limit <n>] [--library tt\|tts]` | GET /v1/videos | 游标分页（nextCursor 为 null = 无下一页）；--library 按库过滤 |
| `vidgate videos get <id>` | GET /v1/videos/{id} | 视频库记录详情 |
| `vidgate products query --account-id <id> [--type shop\|showcase] [--page-token <t>] [--page-size <n>] [--fresh]` | POST /v1/products/query | 可挂车商品；翻页游标见 tts-publish.md §商品 |

## 写命令（CLI ≥ 1.4.0）

| 命令 | 端点 | 必填 | 可选 |
|---|---|---|---|
| `vidgate videos upload <file>` | POST /v1/videos | file（positional） | `--library tt\|tts`（默认 tt）、`--account-id`（**library=tts 时必填**） |
| `vidgate videos delete <id>` | DELETE /v1/videos/{id} | id | —（软删=立即失效） |
| `vidgate publish tt` | POST /v1/publish/tiktok | `--video-url` `--account-id` | `--caption` `--brand-organic` `--branded-content` `--disable-comment` `--disable-duet` `--disable-stitch` `--thumbnail-offset <ms>` `--wait` |
| `vidgate publish tts` | POST /v1/publish/tts | `--file-id` `--account-id` `--product-id` | `--product-title` `--title` `--cover-uri` `--cover-timestamp-ms` `--music-id` `--ai-generated` `--wait` |
| `vidgate publish status --tt --share-id <id>` | GET /v1/publish/tiktok/status | --tt + shareId | 与 --tts 互斥 |
| `vidgate publish status --tts --video-id <id>` | GET /v1/publish/tts/status | --tts + videoId | 与 --tt 互斥 |
| `vidgate publish records [--capability TT\|TTS] [--page <n>] [--page-size <n>]` | GET /v1/publish/records | — | 只读快照，不刷新状态 |
| `vidgate publish stats <itemId>` | GET /v1/publish/stats | itemId（=postId） | TT 视频数据回收 |
| `vidgate precheck submit` | POST /v1/publish/tts/precheck | `--file-id` `--account-id` `--product-id` | `--product-title` `--wait` |
| `vidgate precheck query <taskId>` | GET /v1/publish/tts/precheck/{taskId} | taskId | — |
| `vidgate photos upload <file> --account-id <id>` | POST /v1/photos | file + accountId | 返回 photoUri（发布时作 coverUri） |
| `vidgate music search --account-id <id> --keyword <kw>` | POST /v1/music/search | accountId + keyword | `--page-token` `--search-id`（第 2 页起必带）`--page-size`（≤50）`--region` `--language` |

## --wait 轮询（publish / precheck submit）

- 默认关；`--wait` 开启后轮询到终态：5s 起步 + 抖动，10 分钟封顶（`--timeout` 覆盖）。
- 发布终态：`publish_complete` / `publish_failed` / `beervid_error`；预审终态：`passed` / `failed`。
- 退出码：成功终态 → 0；`publish_failed`/`beervid_error`/预审 failed → 3（业务失败）；等待超时 → 1。
- 轮询中间结果不污染 stdout；最终 envelope 照常输出（`--json` 契约不变）。
