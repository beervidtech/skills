# TikTok Shop 挂车深入指南（TTS）

## 流程

```
seats list（拿 creatorUserOpenId）→ videos upload --library tts（拿一次性 fileId）
→ products query（拿 productId）→（可选）precheck 预审 → publish tts → 轮询终态
```

全程只用官方返回的 id（fileId / creatorUserOpenId / productId / taskId），平台无自造 id。

## fileId 两条铁律（再强调）

1. **一次性消耗**：`publish tts` 调用即消耗，**失败也不可复用**。重发 = 重新 upload 拿新 fileId。
2. **绑定上传账号**：upload 与 publish 的 `--account-id` 必须一致（A 账号传的只能 A 发）。

## 商品拉取（products query）分页语义

- `--type shop`（店铺商品）/ `--type showcase`（橱窗）；**不传 = 两组都返回**，但两组游标各自独立——**翻页必须在单类型下进行**（带 `--type` + 该组的 `nextPageToken`）。
- 数据有 60s 短缓存；`--fresh` 穿透缓存直取官方。
- 选品硬要求：`reviewStatus=APPROVED` 且有货；商品需与视频内容相关（官方会校验相关性）。

## 预审（precheck，可选但建议）

- 配额 **50 次/天/账号**；同一 fileId 重复提交会生成多个任务并**多次消耗**配额；耗尽 → 2002，次日恢复。
- 提交返回 `taskId`（官方预审任务 ID），用它查结果；`--wait` 自动轮询。
- **发布接口不要求预审凭证**（与官方一致）；violation 结果由你自行判断。

### 结果怎么读（官方双 check 结构）

```json
{
  "status": "failed",
  "violationCheckResult": { "status": "FAIL", "issues": [{ "risk": "...", "suggestions": "..." }] },
  "goodQualityCheckResult": { "status": "FAIL", "issues": [{ "code": "...", "suggestions": "..." }] },
  "qualitySuggestions": []
}
```

- **`violationCheckResult` 是红线**：FAIL = 内容违规，`issues[].risk` 是违规类型、`suggestions` 是官方整改建议。`status=failed` 不建议发布——是否发布由用户决策并承担后果。
- **`goodQualityCheckResult` 是加分项**：FAIL 仅表示「不够优质」（如内容促销感不足），**不影响发布资格**；改进建议聚在 `qualitySuggestions`。
- `status=pending` → 稍后重查。

## 封面与音乐（可选素材）

**封面**：

```bash
vidgate photos upload cover.jpg --account-id <id> --json   # → photoUri
```

- 图片要求：JPG/JPEG/PNG/WEBP/HEIC/BMP，≤10MB，宽高比 9:16~16:9。
- 发布时 `--cover-uri <photoUri>`；也可只传 `--cover-timestamp-ms` 截视频帧；两者同传 coverUri 优先；都不传 = 视频首帧。

**音乐**：

```bash
vidgate music search --account-id <id> --keyword love --json
# 翻页：第 2 页起必须同时带 --search-id（首页响应返回）和 --page-token
```

- ⚠️ **传 `--music-id` 会完全覆盖视频原声（含口播人声），不是混音**。口播讲解类视频不要配 BGM。

## publish tts 字段与官方对照（v202607）

| CLI flag | 官方字段 | 说明 |
|---|---|---|
| `--file-id` | video_info.file_id | 一次性消耗 |
| `--title` | video_info.title | 视频文案（支持 #话题 @提及） |
| `--ai-generated` | video_info.is_ai_generated | AI 生成内容必须传 true（带 AI 标识） |
| `--cover-uri` | video_info.cover_uri | photos upload 返回 |
| `--cover-timestamp-ms` | video_info.cover_timestamp_ms | 封面帧毫秒 |
| `--music-id` | video_info.music_id | 覆盖原声，见上 |
| `--product-id` | product_link_info.product_id | products query 返回 |
| `--product-title` | product_link_info.title | 锚点文案：≤30 字符、无标点无 emoji |
| `--account-id` | creatorUserOpenId | 坐席 TTS 绑定账号 |

发布响应 `quota` 字段：创作者发布配额提示，仅平台侧有配额限制时返回，缺省不是错误。

## 状态轮询

- `publish status --tts --video-id <发布响应的 videoId>`（官方 video_id；无需 accountId，平台反查）。
- 提交后立即可查；5–10s 一次；终态 `publish_complete` / `publish_failed`（失败原因读 `failReason`）。
- TTS 无 webhook，轮询是唯一终态路径。
