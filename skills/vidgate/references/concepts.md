# 平台概念与限制全集

## 核心概念

| 概念 | 说明 |
|---|---|
| 坐席（Seat） | 1 个 TikTok 账号 = 1 坐席（计费单位，按坐席×月）。一个坐席可绑同一账号的两种能力：TT 发视频 + TTS 挂车 |
| API Token | `vg_live_` 前缀，控制台「API Keys」页创建，明文只显示一次；可随时撤销（吊销即生效） |
| 视频库 | TT 库：上传返回 `videoUrl`（公网，7 天有效，发布用）；TTS 库：上传返回 `fileId`（一次性消耗，发布/预审用） |
| 商品库 | TTS 账号下可挂车商品的实时拉取（店铺 + 橱窗），拿 `productId` 用于挂车 |
| 发布记录 | 每次发布的只读快照（`publish records`）；最新状态用 `publish status` 实时刷新 |

## 账号 ID 口径（高频踩坑）

| 能力 | ID 类型 | 获取 |
|---|---|---|
| TT 发布 | `businessId` | `vidgate seats list --json` 坐席 TT 绑定字段 |
| TTS 挂车/预审/商品/素材 | `creatorUserOpenId` | `vidgate seats list --json` 坐席 TTS 绑定字段 |

同一个 TikTok 账号两侧 ID 不同，**不可混用**；用错会得到 1005。

坐席 `authStatus=expired` = 授权失效：引导用户去控制台「坐席」页重新绑定，不要反复重试发布。

## 硬限制汇总

| 项 | 限制 |
|---|---|
| 视频大小 | ≤100MB（mp4/mov） |
| TT 视频规格 | 3–600s；宽高 ≥360px；23–60 FPS |
| TT videoUrl 有效期 | 上传后 7 天（到期发布 → 2005） |
| TTS fileId | 一次性消耗（失败也不可用）；绑定上传账号 |
| TT 官方限频 | 每账号 ≤6 个/分钟、≤15 个/天 |
| TTS 预审配额 | 50 次/天/账号（同 fileId 重复提交多次消耗） |
| API 限流 | 60 次/分钟/token（超限 429 + code 2002 + Retry-After；CLI 已自动退避） |
| TTS 锚点文案 | ≤30 字符，无标点无 emoji |
| TTS 封面图 | JPG/JPEG/PNG/WEBP/HEIC/BMP，≤10MB，宽高比 9:16~16:9 |
| 套餐过期 | 写操作（上传/发布）被拒（1003），只读不受影响 |

## envelope 约定

所有端点返回 `{ code, message, data }`；`code=0` 成功。即使官方失败 HTTP 也可能是 200——**永远以 code 为准**（CLI `--json` 原样输出 envelope）。
