# TikTok 发布深入指南（TT）

## 流程

```
seats list（拿 businessId）→ videos upload（拿 videoUrl，7 天有效）→ publish tt（拿 shareId）→ 轮询终态
```

发布后 TikTok 会自己来下载 videoUrl → 转码 → 审核 → 上架。**提交成功 ≠ 发布成功**，必须轮询到终态（`--wait` 已内置）。

## 视频规格（不满足会在发布阶段被官方拒）

| 项 | 要求 |
|---|---|
| 格式 | mp4 / mov（本平台只收这两种；官方还支持 webm） |
| 大小 | ≤100MB（平台侧；官方上限 1GB） |
| 时长 | 3 – 600 秒 |
| 宽高 | 均 ≥ 360 px |
| 帧率 | 23 – 60 FPS |

## 官方限频（每 TikTok 账号）

≤6 个/分钟、≤15 个/天。超限官方报错（透传 3001）。批量发布场景务必串行限速。

## videoUrl 有效期

上传后 **7 天**（平台计时）。到期发布 → `2005`，需重新上传。发布前不必自己算时间——网关会校验并返回明确错误。

## publish tt 字段语义

| flag | 说明 |
|---|---|
| `--caption` | 视频文案 ≤2200 字符；支持 `#话题` 与 `@提及`（@仅互关好友生效，App 内显示为纯文本，≤30 条）；`\n` 换行仅 App 内生效 |
| `--thumbnail-offset <ms>` | 封面帧毫秒，默认 0（首帧）。**别填超过视频时长的值，官方直接判发布失败** |
| `--disable-comment/duet/stitch` | 互动开关，默认 false=不禁用；受账号级设置约束（账号全局禁评时只能传 true） |
| `--brand-organic` | 推广自己的品牌/业务（带「推广内容」标签） |
| `--branded-content` | 与品牌付费合作（带「品牌内容」标签；优先级高于 --brand-organic，同传时后者被忽略） |

## 状态轮询与 postId

- 提交响应的 `shareId`（形如 `v_pub_url~v2.xxx`）是轮询凭证，**务必保存**。
- 建议 5–10s 一次、最多约 30 次；终态 `publish_complete` / `publish_failed` / `beervid_error`。
- 失败后原因读 `failReason` 或 `beervid.reason`（官方原文，向用户如实解释）。
- `postId`（TikTok item_id）在 publish_complete 后写入，官方数据可能延迟约 3 分钟——刚发布完查不到 postId 属正常，稍后再查。

## 数据回收（stats）

```bash
vidgate publish stats <postId> --json
```

返回 `videoViews/likes/comments/shares/reach` 等（`stats` 可得性取决于授权 scope；无数据时 stats 为 null）。itemId 即 postId；查不到说明视频未发布成功或不属于你（1005）。

## 常见失败模式

- 时长/帧率/分辨率不达标 → publish_failed，reason 会指明
- caption 超长/含违规内容 → 官方审核失败
- videoUrl 过期 → 2005（重传即可）
- 账号授权失效（坐席 authStatus=expired）→ 引导用户去控制台重绑，勿重试
