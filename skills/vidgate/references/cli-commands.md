# HAiNA CLI 命令参考

全局 flag（可前可后）：`--json`（恒输出 envelope）/ `--token` / `--base-url` / `--timeout <ms>`。`--help` 仅命令词之前生效（`haina --help`）。
参数值以 `-` 开头时（如实测 TT businessId `-000eHWs…`）空格/等号写法均可（CLI 自动规范化为 `--opt=value`）。
凭据优先级：`--token` > `VIDGATE_API_TOKEN` 环境变量 > `~/.config/vidgate/config.json`。
退出码：0 成功 / 1 本地 / 2 认证 / 3 业务拒绝 / 4 限流 / 5 上游 / 6 内部。
命令面：稳定版 ≥ 1.4.0 全量可用（beta 通道自 0.0.2-beta 起同样全量）；以 `haina --help` 输出为准。

## 凭据

| 命令 | 说明 |
|---|---|
| `haina auth login --token <t>` | 保存凭据（config 文件 0600）；`--base-url` 同传可固化网关地址 |
| `haina auth status` | 查看生效 token（脱敏）/baseUrl 及来源 |
| `haina auth logout` | 清除本地凭据 |
| `haina version` | 打印版本 |

## 本地命令（不访问网关、不消耗配额）

| 命令 | 说明 |
|---|---|
| `haina skill install [--global] [--agent <name>...] [--yes]` | 把本 skill 装进本机 AI agent 的技能目录。默认项目级（按已存在的 agent 目录探测：.claude/.agents/…，全无则兜底 `.agents/skills/vidgate`）；`--global` 装用户级。`--agent` 可选值：claude-code / opencode / codex / cursor / universal（可多个）。目标已存在时：TTY 询问（缺省不覆盖），非 TTY 或 `--yes` 直接覆盖。注意：只写上述探测表内的目录，使用自有 skills 目录的 agent（如 Hermes `~/.hermes/skills`）需安装后手动同步/软链 |

## 只读命令

| 命令 | 端点 | 说明 |
|---|---|---|
| `haina seats list` | GET /v1/seats | 坐席列表：slotNo/status/绑定账号（TT=businessId，TTS=creatorUserOpenId）/authStatus |
| `haina videos list [--cursor <c>] [--limit <n>] [--library tt\|tts]` | GET /v1/videos | 游标分页（nextCursor 为 null = 无下一页）；--library 按库过滤。**已软删的记录仍返回**（status=expired，字段保留），别以「列表消失」判断删除成功 |
| `haina videos get <id>` | GET /v1/videos/{id} | 视频库记录详情（已删记录同样返回，status=expired） |
| `haina products query --account-id <id> [--type shop\|showcase] [--page-token <t>] [--page-size <n>] [--fresh]` | POST /v1/products/query | 可挂车商品；翻页只认 nextPageToken（勿按 totalCount 推算页数），两组游标独立，见 tts-publish.md §商品 |

## 写命令

| 命令 | 端点 | 必填 | 可选 |
|---|---|---|---|
| `haina videos upload <file>` | POST /v1/videos | file（positional） | `--library tt\|tts`（默认 tt）、`--account-id`（**library=tts 时必填**）。TTS 响应 `videoUrl` 恒为 null，以 code=0 判成功 |
| `haina videos delete <id>` | DELETE /v1/videos/{id} | id | 软删=立即失效：发布/预审立即拒绝（2005）；记录仍留列表/详情（status=expired，字段保留），属预期 |
| `haina publish tt` | POST /v1/publish/tiktok | `--video-url` `--account-id` | `--caption` `--brand-organic` `--branded-content` `--disable-comment` `--disable-duet` `--disable-stitch` `--thumbnail-offset <ms>` `--wait` |
| `haina publish tts` | POST /v1/publish/tts | `--file-id` `--account-id` `--product-id` `--title`（**官方必填**，缺省官方报 3001） | `--product-title` `--cover-uri` `--cover-timestamp-ms` `--music-id` `--ai-generated` `--wait` |
| `haina publish status --tt --share-id <id>` | GET /v1/publish/tiktok/status | --tt + shareId | 与 --tts 互斥 |
| `haina publish status --tts --video-id <id>` | GET /v1/publish/tts/status | --tts + videoId（**官方 video_id**：publish tts 响应的 videoId，或 records 里的 shareId 字段；勿用 records 的 videoId——那是平台库 ID，必 1005） | 与 --tt 互斥 |
| `haina publish records [--capability TT\|TTS] [--page <n>] [--page-size <n>]` | GET /v1/publish/records | — | 只读快照，不刷新状态。字段口径：`videoId`=平台视频库 ID（对 videos）；`shareId`=官方句柄（TT=share_id，TTS=video_id，查 status 用它） |
| `haina publish stats <itemId>` | GET /v1/publish/stats | itemId（=postId） | TT 视频数据回收。postId 官方生成延迟约 3 分钟：records 里 postId=null 期间查询必 1005，属预期，等 status 拿到 postId 再查 |
| `haina precheck submit` | POST /v1/publish/tts/precheck | `--file-id` `--account-id` `--product-id` | `--product-title` `--wait` |
| `haina precheck query <taskId>` | GET /v1/publish/tts/precheck/{taskId} | taskId | — |
| `haina photos upload <file> --account-id <id>` | POST /v1/photos | file + accountId | 返回 photoUri（发布时作 coverUri） |
| `haina music search --account-id <id> --keyword <kw>` | POST /v1/music/search | accountId + keyword | `--page-token` `--search-id`（第 2 页起必带）`--page-size`（≤50）`--region` `--language`。官方分页怪癖见 tts-publish.md §音乐：token=数值 offset、searchId 逐次变化、小 page-size 首调可能空页（带返回 token 再查即有） |

## --wait 轮询（publish / precheck submit）

- 默认关；`--wait` 开启后轮询到终态：5s 起步 + 抖动，10 分钟封顶（`--timeout` 覆盖）。
- 发布终态：`publish_complete` / `publish_failed`；`beervid_error` = 平台侧调用失败，可重试。预审终态：`passed` / `failed`。
- 退出码：成功终态 → 0；`publish_failed`/预审 failed → 3（业务失败）；`beervid_error` → 3（平台侧调用失败，可重试）；等待超时 → 1。
- 轮询中间结果不污染 stdout；最终 envelope 照常输出（`--json` 契约不变）。
