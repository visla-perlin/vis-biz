# Clip 预处理 Action 体系

> 范围：clip 素材（以及共用料表的其他实体）的媒体预处理动作体系 —— 谁发起、谁执行、谁回调、契约长什么样、状态如何流转。
> 视角：跨仓库链路（visla-api → transcoder → vcjs）+ 数据契约 + 状态机。
> 背景：clip 的所有媒体级操作（转码、缩略图、快速编辑等）都不在 API 进程内完成，而是通过 **action 协议** 交给外部 transcoder 服务异步执行，再由 vcjs 消费结果回调落库。理解这套协议是新增任何媒体 action（如 transform）的前提。

---

## 1. 三方链路全景

```
┌────────────┐  SQS(transcoder queue)  ┌────────────┐  SQS(job-process-rsp FIFO)  ┌────────────┐
│ visla-api  │ ───────────────────────> │ transcoder │ ─────────────────────────> │    vcjs    │
│ (发起方)    │   PreprocessEntityReqBody│ (执行方)    │   PreprocessEntityBo(结果)  │ (回调处理方) │
└────────────┘                          └────────────┘                            └────────────┘
```

1. **visla-api（发起方）**
   - Controller 接收用户操作（如 clip 快速编辑），完成 ACL 权限、操作锁、参数校验。
   - Service 层按需复制媒体数据（如 reformat 前 `deepCopyVislaMediaForReformat` 深拷贝 visla_media），并把 clip 置为 `PROCESSING`。
   - `PreprocessService` / `VislaMediaProcessService` 组装 `PreprocessEntityReqBody`（actions 列表 + 每个动作的参数 + entity 信息），经 `AwsSqsService.sendSqsForPreprocess` 发送到 transcoder 队列。
   - 发送前会落一条 `service_request_record`（幂等 / 追踪用），消息体带上该记录 id。
2. **transcoder（执行方，独立服务）**
   - 消费 transcoder 队列消息，按 action 执行实际媒体处理（ffmpeg 等），产出新媒体文件（S3 link）。
   - 处理完成后把结果（每个 action 的 code + 产物字段）发送到 job-process-rsp FIFO 队列。
   - **注意：transcoder 是独立仓库 / 独立团队，新增 action 需要同步排期实现，是跨团队外部依赖。**
3. **vcjs（回调处理方，"job 那边"）**
   - `ProcessReqJobService.handleJobProcessRspFifoQueue` 消费结果队列。
   - `TranscodingProcessHandler.processHandler` 按 `EntityType` 分发：
     - CLIP → `handleClipResult`：
       - composite video clip → `CompositeVideoService`
       - new media clip → `VislaMediaProcessService.handleVislaClipResult`
       - 旧 clip → `ClipService.handleOldClipResult`
     - INTRO_OUTRO / PROJECT_ASSET / AGENT_PROJECT_MEDIA / KIT 等各有分支。
   - new media 路径再按媒体类型路由到策略 handler：`ClipVideoProcessHandler` / `ClipImageProcessHandler` / `ClipAudioProcessHandler` / `ClipCompositeVideoProcessHandler`（`ProcessHandlerContext` 按 mediaType + newMedia 选型）。
   - handler 内按 action 更新 `media_process_result` 各列、更新 `visla_media` / `media_thumbnail` / `media_resolution` / clip 本体，必要时**触发后继 action**（见 §4）。
   - 全部动作完成后：更新 clip 状态 COMPLETED、发通知（`notificationService.addNotificationForClipPreprocess`）、同步 ES。
   - 最后 `requestRecordService.updateRequestRecordForCallBack` 闭环 `service_request_record`。

另有一条简化请求队列 `job-process-req`（`ProcessReqBo`：entityId / entityType / action），用于 API 通知 vcjs 推进某实体的预处理状态机（INIT / TODO / DOING 三种推进），不携带媒体参数。

## 2. 请求契约（visla-api → transcoder）

`PreprocessEntityReqBody`（fastjson 序列化，字段蛇形命名）：

```jsonc
{
  "id": 123,                    // service_request_record id（发送前回填）
  "type": "editing",            // entityType
  "actions": ["reformat"],      // PreprocessActionEnum.name 列表
  "message_group_id": "...",    // SQS FIFO 分组
  "data": [
    {
      "entity_id": 456,         // clip id
      "entity_link": "s3://...",// 处理源（standard/playback link）
      "entity_type": "editing",
      "media_type": "video",    // video/audio/image
      "tracking_id": "...",
      "reformat": {             // 与 action 同名的参数块
        "type": 1, "clip_link": "s3://(产物目标地址)",
        "x": 0.5, "y": 0.5, "width": 0.8, "height": 0.8
      }
    }
  ]
}
```

要点：

- `actions` 是动作名列表，`data[]` 内以**同名参数块**携带每个动作的参数；产物目标地址（如 `clip_link`）由 API 预生成并传入。
- 一次消息可携带多个 action（如新上传 clip 的 THUMBNAIL + METADATA + SEPARATE 批量预处理）。

## 3. 结果契约（transcoder → vcjs）

`PreprocessEntityBo`（jackson 反序列化，`@JsonProperty` 蛇形）：

```jsonc
{
  "id": 123,                    // 对应 service_request_record id
  "data": [
    {
      "tracking_id": "...",
      "entity_id": 456,
      "media_type": "video",
      "reformat": { "code": 0 } // 每个动作一个结果块，code=0 成功
    }
  ]
}
```

- 每个动作结果块必带 `code`（0 = 成功，非 0 = 失败码）。
- 产出型动作还带产物字段（如 `separate.audio_link`、`thumbnail.links`、`metadata.width/height/duration` 等），vcjs 落到 visla_media / media_thumbnail 等表。

## 4. Action 清单与后继动作

`PreprocessActionEnum`（visla-api 与 vcjs 双侧同 id 同名，**必须同步维护**）：

| action | id | 用途 | 结果后的典型后继动作 |
| ------ | -- | ---- | -------------------- |
| standardization | 1 | 格式统一转码 | — |
| metadata | 3 | 宽高 / 时长 / 编码信息分析 | — |
| thumbnail | 5 | 缩略图 | — |
| scale | 7 | 多分辨率 | — |
| playback | 9 | m3u8 | — |
| separate | 15 | 音视频分离 | asr 流程 |
| asr_extract | 17 | ASR 转写 | transcript 分句 |
| cut / extract / trim | 25/28/29 | 快速剪辑 | METADATA + THUMBNAIL + PLAYBACK + SEPARATE |
| voice_over | 27 | 替换音轨 | METADATA + SEPARATE + PLAYBACK |
| merge | 35 | 多 clip 合并 | SEPARATE + METADATA + THUMBNAIL + PLAYBACK + VIDEOSHOT + SCALE |
| reformat | 37 | 变换裁剪（crop/fit/hold） | METADATA + THUMBNAIL + PLAYBACK |
| noise_reduction | 45 | 音频降噪 | — |
| **transform** | **49** | **旋转+翻转复合变换（API-1281 新增），结果 = flip(rotate(source))** | METADATA + THUMBNAIL +（视频）PLAYBACK |
| crop | 48 | intro/outro 打包裁剪 | — |

后继动作的意义：**编辑类动作改变了媒体内容，必须重新生成派生物**（元数据 / 缩略图 / m3u8 等）。实现方式是在 handler 的结果 case 里把后继动作置 `DOING` 并加入 `nextActionEnums`，由 `VislaMediaProcessCoreService` 再次发 SQS。旋转 90°/270° 会导致宽高互换，依赖后继 METADATA 重取宽高回写 clip / visla_media。

## 5. media_process_result 状态机

每个实体一行，每列一个动作，取值同 `ProcessActionStatusEnum`：

| 值 | 枚举 | 含义 |
| -- | ---- | ---- |
| 0 | INIT | 未做过该动作（上传常态） |
| 3 | TODO | 待发起 |
| 5 | DOING | 已发出、等回调 |
| 1 | SUCCESS | 成功 |
| 127 | FAILED | 失败 |

整体 `ProcessStatusEnum`（clip 状态）计算（`getVislaProcessStatusEnum`）：

- 常规转码维度（metadata/thumbnail/standardization/separate/playback）全 SUCCESS 且编辑维度（cut/voiceOver/reformat/**transform**）无 DOING/TODO → COMPLETED
- 编辑维度 DOING/TODO 仍在途 → PROCESSING（**INIT 不阻塞**，否则误伤正常上传）
- 常规维度全 FAILED → FAILED

## 6. 快速编辑入口（visla-api）

`TeamspaceClipController` 下 quick edit 组，注解组合固定：`@SecureUrl(tokenValidate)` + `@WorkspaceStorageStatsRequired` + `@DomainOperateLockRequired(CLIP_EDIT_GROUP)` + `@EntityParentRecordAnnotation`：

| 接口 | 动作 | 支持媒体 |
| ---- | ---- | -------- |
| `POST /{clipUuid}/cut` | cut | video |
| `POST /{clipUuid}/extract` | extract | video |
| `POST /{clipUuid}/trim` | trim | video |
| `POST /{clipUuid}/merge` | merge | video |
| `POST /{clipUuid}/reformat` | reformat（crop/fit/hold） | video |
| `POST /{clipUuid}/generate-noise-reduction` | noise_reduction | audio/video |
| `POST /{clipUuid}/transform`（API-1281 新增） | transform（rotate+flip） | **video + image** |

使用统计：`RptQuickEdit`（rpt_quick_edit 表）按动作记次，`QuickEditEnum` 维护动作 → 字段映射，新动作需加列。

## 7. 新增 action 的跨仓库检查清单

以 transform 为例（通用模板）：

1. **visla-api**：`PreprocessActionEnum` 加值（id 用下一个空闲值）→ `PreprocessEntityReqBody` 加参数块 → Controller 端点（注解组合）→ Service（校验 / ACL / 锁 / 媒体深拷贝 / clip 置 PROCESSING）→ 组装参数块发 SQS → `media_process_result` 加列（Flyway + Entity + Mapper XML 全链路）→ `QuickEditEnum` + rpt_quick_edit 加列（如需统计）。
2. **vcjs**：`PreprocessActionEnum` 同步加值 → `PreprocessEntityBo` 加结果块 → `MediaProcessResult` Entity + Mapper XML 加列 → 编辑类 action 还需在 `TranscodingProcessHandler` 路由判断加值 → 对应媒体类型 ProcessHandler 实现 `handlerEditingTranscodingResult` case（成功置 SUCCESS + 触发后继动作）→ `getEditProcessStatus` / `getVislaProcessStatusEnum` 把新列纳入编辑维度。
3. **transcoder（外部依赖）**：实现动作执行与产物上传，按结果契约回发。**必须与媒体团队对齐排期，否则链路空转（DOING 卡死）。**
4. DB 列变更：Flyway 由 visla-api 负责，vcjs 同步 Entity + Mapper（共享表）。
