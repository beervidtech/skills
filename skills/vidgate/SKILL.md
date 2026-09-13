---
name: vidgate
description: 通过 vidgate CLI 发布视频到 TikTok / TikTok Shop（挂车带货）、查询坐席与商品、跟踪发布状态。Use when the user wants to upload or publish videos to TikTok / TikTok Shop, query seats or shoppable products, run TTS precheck, or check publish status via the vidgate API platform.
metadata:
  author: beervid
  version: "0.1"
---

# vidgate 视频发布

vidgate 是 TikTok / TikTok Shop 视频发布 API 平台。本 skill 教你用官方 CLI（`vidgate`）完成发布全流程。

## 0. 前置自举（每次会话先检查）

```bash
which vidgate || npm install -g @vidgate/cli
```

- 没有 Node ≥22 环境 → 停止并提示用户安装。
- 检查凭据：`vidgate auth status`（退出码非 0 = 未配置）→ 提示用户提供 API Token 并执行 `vidgate auth login --token <t>`，或让用户 `export VIDGATE_API_TOKEN=vg_live_...`。
- 写链路命令（`videos upload` / `publish *` / `precheck` / `photos` / `music`）要求 CLI ≥ 1.3.0；`vidgate version` 核对。0.0.x 版本只有只读命令（auth / seats / videos list,get / products query）。

## 1. 凭据纪律（安全红线）

- Token 一律经环境变量或 `vidgate auth login` 注入；**禁止**把 token 写进任何源码、脚本、提交。
- Token 明文只在用户控制台创建时可见一次；弄丢就让用户去控制台重建，不要尝试找回。

## 2. 输出纪律

- **永远带 `--json`**：恒输出服务端 envelope 原文 `{ code, message, data }`；以 `code === 0` 判成败（HTTP 200 也可能是官方失败）。
- 退出码语义：0 成功 / 1 本地参数 / 2 认证(1001-1003) / 3 业务拒绝(1004-1008,2001,2005) / 4 限流(2002) / 5 上游官方(3001,3002) / 6 内部错误。429 限流 CLI 已自动按 Retry-After 退避，无需处理。

## 3. 流程路由

### TT 发布（TikTok 普通视频）

```
vidgate seats list --json                      # 拿已绑定 TT 账号的 businessId（accountId）
vidgate videos upload <file> --json            # → videoUrl（7 天有效）
vidgate publish tt --video-url <url> --account-id <id> --caption "文案 #话题" --wait --json
```

- 视频要求：mp4/mov ≤100MB；官方硬规格 3–600s、宽高 ≥360px、23–60 FPS。
- `--wait` 自动轮询到终态：`publish_complete` 成功 / `publish_failed` 失败（读 `failReason`/`beervid.reason` 向用户解释官方原因）。

### TTS 挂车（TikTok Shop 带货视频）

```
vidgate seats list --json                                  # 拿 TTS 账号 creatorUserOpenId（accountId）
vidgate videos upload <file> --library tts --account-id <id> --json   # → fileId
vidgate products query --account-id <id> --json            # 拿可挂车商品 productId
vidgate precheck submit --file-id <f> --account-id <id> --product-id <p> --product-title <锚点文案> --wait --json   # 可选但建议
vidgate publish tts --file-id <f> --account-id <id> --product-id <p> --product-title <锚点文案> --wait --json
```

## 4. TTS 四条硬规则（官方规则，违反必失败）

1. **fileId 一次性消耗**：发布调用即消耗，**失败也不可复用**；重发必须重新上传拿新 fileId。
2. **fileId 绑定上传账号**：A 账号上传的 fileId 只能由 A 账号发布（upload 与 publish 的 `--account-id` 必须一致）。
3. **锚点文案（productTitle）**：≤30 字符，不含标点和 emoji。
4. **预审可选但建议**：配额 50 次/天/账号；violation FAIL（status=failed）不建议发布——是否发布由用户决定，明确告知后果。

## 5. 错误处置速查

| code | 含义 | 处置 |
|---|---|---|
| 1001 | token 无效/撤销 | 引导用户重新创建并 login |
| 1003 | 套餐过期 | 提示用户续费，写操作被拒 |
| 1005 | 资源不存在/不属于你 | 核对 id 与账号归属（坐席列表查） |
| 2001 | 参数/文件校验失败 | 按 message 修正 |
| 2002 | 限流/预审配额耗尽 | 限流 CLI 已退避；预审配额次日恢复 |
| 2005 | TT 视频 URL 过期（7 天） | 重新上传 |
| 3001 | 官方业务错误 | 读 message 向用户解释官方原因；内容类违规改素材重试 |
| 3002 | 官方超时 | 稍后重试 |

## 6. 无 CLI 降级

用户拒绝安装 CLI 时：按在线 API 参考直连 HTTP（文档站 `/docs/api-reference`，需用户登录控制台查看）；envelope 与错误码语义同上。
