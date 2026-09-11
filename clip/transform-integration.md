# Clip Transform（旋转 / 翻转）对接文档 — API-1281

> 读者：前端、VME（transcoder）server、后端联调。
> 范围：前端传参与状态感知、clip 状态变更、API ↔ VME 的 SQS 消息契约、vcjs 回调后的处理链路。
> 业务规则与设计取舍见 [`transform.md`](./transform.md)，预处理体系全貌见 [`preprocess-actions.md`](./preprocess-actions.md)。

---

## 1. 功能概述

Clip 菜单新增 4 个快速编辑操作：**Flip Horizontal / Flip Vertical / Rotate Left / Rotate Right**，支持 **video + image** clip。用户在菜单里预览调整，点 **confirm** 后前端一次性提交最终变换状态，后端深拷贝媒体并交 VME 转码，完成后 clip 恢复可用。

```
前端(confirm) ──POST /clip/{uuid}/transform──> visla-api ──SQS──> VME 转码
                                │                              │ 产物上传 S3
                                │ clipStatus=processing        │ 回发结果
                                ▼                              ▼
前端(轮询/通知) <──── clipStatus 更新 ──── vcjs 消费回调 + 后继动作(元数据/缩略图/m3u8)
```

## 2. 前端对接

### 2.1 交互模型（confirm 模式）

4 个按钮只改**本地预览状态**，不立即请求；点 confirm 才提交一次全量参数：

| 按钮 | 本地状态变化 |
| ---- | ---- |
| Rotate Right | `angle = (angle + 90) % 360` |
| Rotate Left | `angle = (angle + 270) % 360` |
| Flip Horizontal | `horizontal = !horizontal` |
| Flip Vertical | `vertical = !vertical` |

预览渲染顺序固定：**先应用 rotation，再应用 flip**（与后端转码语义一致，见 §4.3）。

confirm 时提交 `{rotate: angle, flip: {horizontal, vertical}}`。

- 状态归零（`angle=0` 且 flip 全 false）→ 前端直接不发请求（发了也会被后端 400 拒绝）。
- clip 处于 `processing/publishing/failed` 状态、或 composite video clip → 按钮置灰（发了会被后端 403 拒绝）。

### 2.2 接口定义

```
POST /v3/clip/{clipUuid}/transform
Content-Type: application/json
```

请求体：

```json
{
  "rotate": 90,
  "flip": { "horizontal": false, "vertical": false },
  "duplicate": true
}
```

| 字段 | 类型 | 必填 | 说明 |
| ---- | ---- | ---- | ---- |
| rotate | Integer | 是 | 顺时针角度，只允许 **0/90/180/270**，默认 0 |
| flip | Object | 否 | `{horizontal: boolean, vertical: boolean}`，缺省全 false |
| duplicate | boolean | 否 | **Keep a Copy of Original 复选框**，与 cut 的 `duplicate` 字段同语义同默认（false，需显式传）：勾选→传 true，变换前浅拷贝一份原 clip（标题加 " copy"，引用旧 media 保留原始内容），变换仍作用于原 clip；取消勾选→传 false 或不传，原内容被变换结果覆盖 |

checkbox 勾选（默认）→ `duplicate: true`；取消勾选 → `duplicate: false`。响应里返回的仍是原 clip 的 `clipUuid`，副本出现在同 folder 列表中（标题 "... copy"）。

响应（`ClipEditVo`）：

```json
{
  "code": 0,
  "data": {
    "clipUuid": "1234567890",
    "clipStatus": "processing",
    "clipLink": "https://s3.../standard.mp4",
    "capabilities": { ... }
  }
}
```

- `clipLink` 是新媒体目标地址（转码完成前内容尚未生成），前端播放请继续用旧数据 + 本地预览。
- 接口返回即表示任务已受理，后续以 §2.3 方式感知完成。

错误码：

| 场景 | 返回 |
| ---- | ---- |
| rotate 非 0/90/180/270 | `ILLEGAL_PARAMS` ("rotate must be 0/90/180/270") |
| rotate=0 且 flip 全 false（无变换） | `ILLEGAL_PARAMS` ("no transform to apply") |
| clip 类型非 video/image（如 audio） | `ACTION_DENIED` |
| **composite video clip** | `ACTION_DENIED` |
| clip 正在 processing/publishing/failed | `ACTION_DENIED` |
| 无快速编辑权限（ACL） | `ACTION_DENIED` |
| 触发操作锁（同 clip 并发编辑，10s 自动释放） | 锁冲突错误 |

### 2.3 状态感知（如何知道完成）

两种方式，建议**通知为主、轮询兜底**：

1. **通知**：处理结束（成功或失败）推送 `PREPROCESS_FINISHED` 通知（targetType=CLIP），payload 关键字段：`success`（bool）、`clipStatus`、`clipUuid`、`processResult`。
2. **轮询**：`GET /v3/clip/{clipUuid}/info`，关注 `clipStatus`（及 `processResult` 明细）。

`clipStatus` 枚举：`init / downloading / uploading / processing / publishing / completed / failed`。

## 3. 状态变更明细

一次 transform 的状态时序（T0 用户 confirm）：

| 时刻 | 事件 | clip.clipStatus | media_process_result.transform | 同行其他列 |
| ---- | ---- | ---- | ---- | ---- |
| T0 | API 受理（事务提交后发 SQS） | `processing` | `DOING(5)` | duplicate=true 时已生成原 clip 副本（标题 " copy"）；media 深拷贝完成，clip 派生 link 清空 |
| T1 | VME 回报 code=0 | 仍 `processing`（后继动作在途） | `SUCCESS(1)` | metadata/thumbnail(+video: playback) → `DOING`，并再发后继 SQS |
| T2 | 后继动作全部完成 | `completed` | `SUCCESS(1)` | 宽高已按 90°/270° 互换回写，缩略图/m3u8 重新生成，发成功通知、同步 ES |
| T1' | VME 回报 code≠0 | `failed` | `FAILED(code)` | 发失败通知；用户可**重新发起** transform（走新一轮深拷贝，原素材不受影响） |

要点：

- `transform` 列取值语义与其他 action 一致：`0=INIT`（从未做过，存量 clip 默认）、`3=TODO`、`5=DOING`、`1=SUCCESS`、`≥127=FAILED`。
- INIT **不阻塞** clip 的 COMPLETED 判定（保护存量数据）；DOING/TODO 会阻塞。
- transform 与 cut/voiceOver/reformat 同属"编辑维度"，任一在途则整体不能 COMPLETED。
- 失败只影响本 clip 本轮编辑，**不产生扣费/退款类副作用**（仅记录 quota 操作日志）。

## 4. 后端 ↔ VME server 对接

### 4.1 请求消息（visla-api → transcoder 队列）

队列：transcoder 队列（API 配置 `properties.transcoder.queue-name`），消息体 fastjson 序列化：

```json
{
  "id": 987654321,
  "type": "editing",
  "actions": ["transform"],
  "message_group_id": "clip-1234567890",
  "data": [{
    "entity_id": 1234567890,
    "entity_link": "https://s3.../old-standard.mp4",
    "entity_type": "editing",
    "media_type": "video",
    "tracking_id": "xxxx",
    "transform": {
      "clip_link": "https://s3.../new-standard.mp4",
      "rotate": 90,
      "flip": { "horizontal": true, "vertical": false }
    }
  }]
}
```

| 字段 | 说明 |
| ---- | ---- |
| id | `service_request_record` 主键，回发结果时必须原样带回（闭环追踪） |
| actions | 固定 `["transform"]` |
| entity_link | 处理源：new media clip 为旧 media standard link；旧 clip 为 clipLink |
| media_type | `video` 或 `image` |
| data[].transform.clip_link | **产物目标地址**（API 预生成），VME 直接把结果上传到这里 |
| rotate | 0/90/180/270；flip-only 时为 0 照传 |
| flip | 全 false 时仅 rotate 生效 |

### 4.2 响应消息（VME → job-process-rsp FIFO 队列）

```json
{
  "id": 987654321,
  "entityType": "editing",
  "service": 1,
  "actions": ["transform"],
  "data": [{
    "tracking_id": "xxxx",
    "entity_id": 1234567890,
    "media_type": "video",
    "transform": { "code": 0 }
  }]
}
```

- `code = 0` 成功；非 0 为失败码（原样落库到 `media_process_result.transform`，并进入 clip 失败链路）。
- `entity_id` / `tracking_id` 原样回传，用于路由与 trace。

### 4.3 VME 实现约束（联调对齐点）

1. **变换顺序写死：结果 = flip(rotate(source))，先旋转后翻转**。两者不可交换，顺序不同结果不同。
2. **rotate 顺时针为正**（0/90/180/270）。前端 Rotate Left = 270 由 API 侧传入，VME 无需理解方向语义。
3. **视频保留音轨**（音频不做变换）；图片同理单图处理。
4. 产物**上传到请求给出的 `clip_link`**（不要自建 key）；上传成功后才回发 code=0。
5. 失败时尽量回发具体失败码（利于排障），不要静默不回（会导致 clip 卡 DOING）。
6. 幂等：同 id 消息重复投递不应产生重复上传/回发（以 SQS 至少一次投递为前提）。

## 5. vcjs 后续处理逻辑（回调消费链路）

```
job-process-rsp(FIFO)
  └─ ProcessReqJobService.handleJobProcessRspFifoQueue
      └─ TranscodingProcessHandler.processHandler (entityType=CLIP, service=TRANSCODER)
          └─ actions 含 TRANSFORM → handleClipEditingResult（编辑类路由）
              └─ new media → VislaMediaProcessService.handleVislaClipEditingResult
                  ├─ handler 按 mediaType 选型: ClipVideo / ClipImageProcessHandler
                  │    └─ handlerEditingTranscodingResult: case TRANSFORM
                  │         code=0 → transform=SUCCESS + nextActions{METADATA,THUMBNAIL,(PLAYBACK)}
                  │         code≠0 → transform=FAILED
                  ├─ 更新 media_process_result / clip 尺寸(size)
                  ├─ hasMoreTranscoding → clipStatus 强制 PROCESSING，
                  │    processCoreService 再发后继 SQS（走常规转码链路）
                  └─ 全部完成 → COMPLETED + PREPROCESS_FINISHED 通知 + ES 同步
最后：requestRecordService.updateRequestRecordForCallBack 闭环 service_request_record
```

后继动作的作用：transform 改变了媒体本体，派生物必须重做——

| 后继 action | 目的 | video | image |
| ---- | ---- | ---- | ---- |
| METADATA | 重取宽高/时长（90°/270° 宽高互换靠它回写 clip 与 visla_media） | ✅ | ✅ |
| THUMBNAIL | 重新生成缩略图 | ✅ | ✅ |
| PLAYBACK | 重新生成 m3u8 | ✅ | ❌（图片无） |

完成判定：常规转码维度全 SUCCESS 且编辑维度（cut/voiceOver/reformat/transform）无 DOING/TODO → COMPLETED。

## 6. 错误场景与排查

| 现象 | 原因 | 排查/恢复 |
| ---- | ---- | ---- |
| clip 长期 processing，transform 卡 DOING | VME 未回发（未实现/异常静默） | 查 `service_request_record` 该消息状态；与 VME 排查；用户可待锁释放后重试（新一轮深拷贝，不叠加画质损失） |
| transform=FAILED，clip=failed | VME 回 code≠0 | code 值即 VME 失败码；用户重新发起即可 |
| 后继 metadata/thumbnail 失败 | S3/转码异常 | clip 停在 processing，`processResult` 明细可看具体 action；对应链路已有失败通知 |
| 接口 403 | 类型不支持 / composite / 状态不允许 / ACL | 见 §2.2 错误码表 |
| 接口 400 | 参数非法 / 无变换 | 前端 confirm 前自行拦截 |

## 7. 联调 Checklist

- [ ] 前端：4 按钮累积状态 + confirm 只发一次；预览顺序先 rotate 后 flip
- [ ] 前端：Keep a Copy of Original 复选框（默认勾选）→ duplicate 字段；完成后列表刷新能看到 copy 副本
- [ ] 前端：clipStatus=processing 期间按钮置灰；完成后刷新列表/详情
- [ ] VME：transform action 实现（顺序、顺时针、保音轨、上传 clip_link、回发 code）
- [ ] 后端：video / image × 成功 / 失败 四条链路回归（image 是首个编辑类动作路径）
- [ ] 后端：90°/270° 后宽高互换、缩略图/m3u8 更新、通知与 ES 同步
- [ ] 数据库：`media_process_result.transform`、`rpt_quick_edit.transform_times` 迁移执行（v0.3.6.5）
