# Avatar 技术细节文档

> 本文是 [README.md](./README.md)（业务规则）的实现配套，含类、方法、行号、落库字段、数据结构。
> 业务流程描述见 README.md 对应章节（本文以 `📖 见 README §x` 互引）。
> 基于 visla-api / vcjs 源码梳理，行号为撰写时快照，改动后可能漂移。

---

## 1. 代码结构索引

### 1.1 visla-api（同步、面向用户）

| 职责 | 关键类 |
| ------ | -------- |
| 对外接口 | `controller/v3/WorkspaceAvatarController.java`（CRUD/生成/变声）<br>`controller/v3/ProjectEditingV3Controller.java`（regenerate/delete unrendered）<br>`controller/v3/ProjectLayerV3Controller.java`（scene 级 look 操作）<br>`controller/v3/SyntheticVoiceV3Controller.java`（TTS） |
| Avatar 主数据 | `service/AvatarService.java`（saveAvatar/importAvatar/saveAvatarDescribe/generateAvatarPreviewLook） |
| Look 服务 | `service/AvatarLookService.java`（initAvatarLook/generateLookSampleVideo/processLookVideoMatting/changeVoiceId） |
| Look 资产/分组/推荐图/收藏 | `service/AvatarLookAssetService.java` / `AvatarLookGroupService.java` / `AvatarRecommendImageService.java` / `WorkspaceFavoriteAvatarService.java` |
| 项目 avatar | `service/ProjectAvatarService.java`（编辑态 + 预览生成同步入口） |
| AI 请求构造 | `internal/ai/service/AiAvatarService.java` |
| 导出 | `service/ProjectExportVideoService.java` |
| 图层处理 | `service/layer/AvatarLayerHandler.java` |
| Domain | `domain/Avatar.java` / `AvatarLook.java` / `AvatarLookAsset.java` / `AvatarLookGroup.java` / `ProjectAvatar.java` / `WorkspaceFavoriteAvatar.java` / `AvatarRecommendImage.java` |
| 枚举 | `enums/Avatar*Enum.java` / `ShotTypeEnum.java` / `ProjectAvatarRenderStatusEnum.java` |

### 1.2 vcjs（异步、回调与批处理）

| 职责 | 关键类 |
| ------ | -------- |
| SQS 回调消费 | `sqs/consumer/service/AiAvatarCreatorCbConsumer.java`（look 生成回调）<br>`AiAvatarLookProcessCbConsumer.java`（matting/sampling/advance 回调）<br>`AiAdvancedAvatarChangeVoiceCbConsumer.java`（变声回调）<br>`ProjectAvatarPreviewCbConsumer.java`（预览渲染回调）<br>`SyntheticVoiceConsumer`（Azure TTS）<br>`AudioWordTimestampConsumer`（OpenAI/MiniMax TTS 对齐） |
| Look 后处理 | `service/AvatarProcessService.java`（matting/sampling/advance 落库） |
| 项目 avatar | `service/ProjectAvatarService.java`（预览生成异步主链路 + 计费） |
| TTS | `service/tts/AzureVoiceGenerateServiceImpl.java`<br>`service/AudioWordTimestampService.java`<br>`service/VoiceOverAlignerHandlerService.java` |
| 高级 avatar 导入 | `service/AdvancedAvatarImportService.java` |
| 维护 Job | `xxl/jobs/one/AvatarInitJob.java` / `AdvancedAvatarInitJob.java` / `AdvancedAvatarVoiceBackfillJob.java`<br>`xxl/jobs/DropDataForDeletedAvatarJob.java`<br>`xxl/jobs/one/WorkspaceFavoriteAvatarJob.java` |
| Entity | `store/entity/Avatar*.java` / `ProjectAvatar.java` |

---

## 2. 数据模型全字段表

📖 业务视角见 README §2/§3

### 2.1 Avatar（角色）

| 字段 | 类型 | 说明 |
| ------ | ------ | ------ |
| id, workspaceId, name, description | — | 基础 |
| thumbnailLink, mediaLink | String | 角色图（mediaLink=character image） |
| category | AvatarCategoryEnum | PUBLIC(0)/PRIVATE(1) |
| sourceType | AvatarSourceTypeEnum | GENERATE(0)/UPLOAD(1)/IMPORT(2)/CAPTURE(3) |
| ageGroup, gender, ethnicity | Enum | 人群属性 |
| mediaType | MediaTypeEnum | 均为 IMAGE |
| processStatus | AssetProcessStatusEnum | INIT/PROCESSING/COMPLETED/FAILED |
| status, createdBy, modifiedBy, createdAt, modifiedAt | — | 通用 |

### 2.2 AvatarLook（造型 + 动作视频载体）★ 核心

| 字段 | 类型 | 说明 |
| ------ | ------ | ------ |
| id, avatarId, groupId, name, description | — | groupId 同组 basic+advance 共享 |
| lookType | AvatarLookTypeEnum | BASIC(1)/ADVANCE(2) |
| voiceId | Long | 绑定音色（驱动 lip-sync） |
| thumbnailLink, imageLink, transparentImageLink | String | 静态图（透明图用于叠加） |
| videoLink | String | AI 生成的原始 look 视频 |
| foregroundLink, mattingLink | String | 抠像后前景/抠像视频（matting 产出） |
| sampleVideoLink, sampleMattingLink | String | 口播采样视频（sampling 产出，含声音） |
| width, height | Integer | 视频分辨率 |
| processStatus | AssetProcessStatusEnum | 见 §3.2 |
| status, createdBy, modifiedBy, createdAt, modifiedAt | — | 通用 |

### 2.3 AvatarLookAsset（多景别资产，仅 advance）

| 字段 | 说明 |
| ------ | ------ |
| lookId | 所属 look |
| assetType | 资产类型（如 advanced_base） |
| shotType | ShotTypeEnum 值：head_shot/base_full_body/medium_wide/medium_shot/medium_close/subtle_side |
| assetUrl | 原始图 |
| transparentImageUrl | 透明背景图 |
| title, description, tags | 元数据 |
| width, height, status | — |

### 2.4 ProjectAvatar（场景使用实例）★ 核心

| 字段 | 类型 | 说明 |
| ------ | ------ | ------ |
| id, projectId, sceneId, lookId, assetId | — | assetId=0 表示未渲染(image占位) |
| mediaType | MediaTypeEnum | IMAGE(占位)/VIDEO(已渲染) |
| shotComposition | AvatarCompositionModeEnum | 构图（HOST_ONLY/HOST_PIP/...） |
| shape, zIndex, coordOffset, scale, rotation, bgColor | — | 位置/变换 |
| visible | Boolean | 是否显示 |
| startTime, endTime, duration | Integer | 相对 asset 视频时间轴的切片区间（毫秒） |
| timeOffsetInScene | Integer | scene 内偏移 |
| reused | Boolean | TRUE=引用跨场景共享视频 |
| renderStatus | ProjectAvatarRenderStatusEnum | INIT(0)/PROCESSING(3)/COMPLETED(9)/FAILED(12) |
| status, createdAt, modifiedAt | — | 通用 |

> ⚠️ `startTime/endTime/duration` 是**相对 avatar 视频资产自身时间轴**的切片值，非相对 scene。跨场景合并时各 scene 各自记录自己在长视频中的区间。

### 2.5 其他表

- **AvatarLookGroup**：id, avatarId, createdAt, modifiedAt（advance look 分组）
- **AvatarRecommendImage**：id, workspaceId, avatarId, name, imageLink, thumbnailLink, selected, status
- **WorkspaceFavoriteAvatar**：id, workspaceId, userId, avatarId, lookId

---

## 3. 枚举与状态机

📖 业务视角见 README §5

### 3.1 关键枚举

| 枚举 | 取值 |
| ------ | ------ |
| AvatarLookTypeEnum | BASIC(1, policy="basic") / ADVANCE(2, policy="advanced") |
| AvatarSourceTypeEnum | GENERATE(0)/UPLOAD(1)/IMPORT(2)/CAPTURE(3) |
| AvatarCategoryEnum | PUBLIC(0)/PRIVATE(1) |
| AvatarCompositionModeEnum | HOST_ONLY(0)/VO_ON_VISUAL(1)/VISUAL_ONLY(2)/HOST_PIP(3)/VISUAL_PIP(4)/SMART_COMPOSITION(5)/SOT(6) |
| ShotTypeEnum | head_shot/base_full_body/medium_wide/medium_shot/medium_close/subtle_side |
| AvatarProcessEnum | MATTING(1)/SAMPLING(3)/ADVANCED_AVATAR(5, 仅vcjs) |
| ProjectAvatarRenderStatusEnum | INIT(0)/PROCESSING(3)/COMPLETED(9)/FAILED(12) |

### 3.2 AvatarLook.processStatus 流转

```
Basic:  INIT →[look video 回调]→ PROCESSING →[matting 回调]→ PROCESSING →[sampling 回调]→ COMPLETED
Advance: INIT →[look video 回调]→ PROCESSING →[advanced_avatar 回调]→ COMPLETED
任一步失败 → FAILED
```

### 3.3 ProjectAvatar.renderStatus 流转

```
INIT(0) →[generatePreviewAvatarVideos 起]→ PROCESSING(3) →[回调成功]→ COMPLETED(9)
                                                        →[回调失败]→ FAILED(12)
```

---

## 4. 关键链路技术实现

### 4.1 Avatar 创建（4 来源）

📖 业务见 README §4.1

**saveAvatar** (`AvatarService.java:230-281`)，按 `sourceType` 分支：

| 来源 | mediaLink | thumbnailLink | 额外 |
| ------ | ----------- | --------------- | ------ |
| GENERATE | reqBody.avatarUrl | recommendImage.thumbnailLink → copyThumbnailToAvatar | selectAvatarRecommendImage 标记选中 |
| IMPORT | reqBody.avatarUrl | null | 无 |
| UPLOAD | reqBody.avatarUrl | null | afterCommit → preprocessAvatarThumbnail 异步 |
| CAPTURE | reqBody.avatarUrl | null | 同 UPLOAD |

新建 avatar：`sourceType=reqBody.sourceType, mediaType=IMAGE, category=PRIVATE, processStatus=COMPLETED, status=ACTIVE`。
非 GENERATE 来源要求 `FeePlanEnum.allowBusinessAndEnterprise` + `checkAvatarCountLimitation`。

**importAvatar** (`AvatarService.java:283-312`)：从 Clip 取图（newMedia→vislaMedia.standardLink / 旧→clip.clipLink）+ thumbnailLink，`copyMediaToAvatar` + `copyThumbnailToAvatar`，`sourceType=IMPORT`。

**GENERATE 完整子链**：

- `saveAvatarDescribe` (`:323-341`)：建 avatar(processStatus=INIT)
- `AiAvatarService.createAvatarCharacterImage` (`:149-165`)：发 `AVATAR_CHARACTER_IMAGE` 请求
- vcjs 回调 → `handleAiAvatarCreatorSuccessResult`：删除旧推荐图 + 批量建 AvatarRecommendImage + preprocess 缩略图，avatar.processStatus=UPLOADING
- 用户选图 → `generateByRecommendImage`（`AvatarRecommendImageService:176-214`）：复制图 → 建 group → init basic+advance look → createAvatarLookVideo

### 4.2 Look 生成 AI 请求结构

📖 业务见 README §4.2

**双 look 初始化** (`AvatarService.generateAvatarPreviewLook:344-357`)：

```
groupId = createGroup(avatar.id)
basicLook   = initAvatarLook(avatar, reqBody, groupId, BASIC)
advanceLook = initAvatarLook(avatar, reqBody, groupId, ADVANCE)
createAvatarLookVideo(avatar, basicLook, advanceLook, workspace, described)
```

**AI 请求** `AvatarCreateLookVideoBo`（`AiAvatarService.constructCreateLookVideoBo:78-135`）：

| JSON key | 说明 |
| ---------- | ------ |
| `character_image` | avatar.mediaLink（角色图 S3 URL） |
| `avatar_look_description` | 造型描述 |
| `image2image` | AvatarImageToImageBo（appearance_posture_gesture + avatar_character 属性），空则 null |
| `basic_avatar` | BasicAvatarLookBo{look_id, video_url_prefix, image_url_prefix} |
| `advanced_avatar` | AdvanceAvatarLookBo{look_id, video_url, video_matting_url, video_origin_url, voice} |

`msgType = AVATAR_LOOK_VIDEO(3001)`，BO 序列化为 JSON 存 S3，`aiGateWayClient.createAvatarLookVideo(baseReqBo)`。

**AvatarModelPolicyBo**（`bo/avatar/AvatarModelPolicyBo.java`）：字段 `host_only` / `host_pip`。**不参与 look 生成**，而在 Agent 视频合成时（`AiAgentService.assembleCreateVideosBo`）按构图模式告知 AI 用 basic/advanced 策略。

### 4.3 Look 后处理落库（vcjs）

📖 业务见 README §4.3

**Basic 两步串行**：

1. **MATTING 成功**（`AvatarProcessService.handlePrecessLookVideoMattingSuccess:138-163`）：
   - 落库 AvatarLook：`foregroundLink`, `mattingLink`, `transparentImageLink`, `thumbnailLink`, `width`, `height`
   - **不改 processStatus**（仍 PROCESSING）
   - Consumer 自动触发 `generateLookSampleVideo`（`AiAvatarLookProcessCbConsumer:84-86`）
2. **SAMPLING 成功**（`handlePrecessSampleVideoMattingSuccess:215-229`）：
   - 落库 AvatarLook：`sampleVideoLink`, `sampleMattingLink`, **`processStatus=COMPLETED`**
   - `releaseLockByAvatarIfGroupCompleted` + pushAvatarCreateLookCbResult

**Advance 一步**（`handlePrecessAdvanceLookSuccess:231-309`）：

- IMAGE 资产 → 每条插一条 `AvatarLookAsset`（shotType/assetUrl/transparentImageUrl/title/description/tags/width/height）
- VIDEO 资产 → 更新 AvatarLook：`videoLink`, `sampleVideoLink`, `sampleMattingLink`, `width/height`, `imageLink`(medium_shot), `transparentImageLink`, **`processStatus=COMPLETED`**
- `isGroupAllCompleted`（同组 look 全 COMPLETED/FAILED）→ releaseLock

**失败**：`handleProcessLookVideoMattingFailed` → processStatus=FAILED + releaseLock + 通知 + Zoom 告警。

### 4.4 变声

📖 业务见 README §4.4

`AvatarLookService.changeVoiceId:345-367`：

- 置 `processStatus=PROCESSING`
- BASIC → `generateLookSampleVideo`（重新 sampling）
- ADVANCE → `generateAdvancedLookSampleVideo`：发 `ChangeAdvancedAvatarVoiceBo`{videoOriginUrl/videoMattingUrl/videoUrl/sampleAudioUrl} → AI 替换音轨
- 回调（`AiAdvancedAvatarChangeVoiceCbConsumer`）→ `handleChangeVoiceResult` 更新 videoUrl

### 4.5 ProjectAvatar 绑定与位置计算

📖 业务见 README §4.5

- `initProjectAvatarData`（visla-api `:299-360`）/ vcjs `buildProjectAvatar`：初始 `mediaType=IMAGE, assetId=0, renderStatus=INIT, startTime=0, endTime=scene.duration`
- 位置计算：按 `shotComposition` + `videoFormat` 调 `resetTo16_9/9_16/1_1`（basic）或 `resetTo*Advance`（advance）算 scale/coordOffset
- 构图自动判定（`addProjectAvatarLook:280-296`）：非 HOST_ONLY/HOST_PIP 时，scene 有 FOOTAGE timeline → HOST_PIP，否则 HOST_ONLY

### 4.6 预览生成跨场景合并（vcjs）

📖 业务见 README §4.6

**generatePreviewAvatarVideos**（vcjs `:474-511`）：先全量置 `renderStatus=PROCESSING`，按 lookType 分流。

**Basic 跨场景合并成功**（`handlePreviewAvatarSuccess:711-810`）：

1. 读 `segmentAudiosUrl` → `List<AvatarSceneRefBo>{sceneId, duration}`，求总 duration
2. 建一个 `VislaMedia`（assetId, assetUrl, VIDEO, 总duration, w/h）+ matting 标记
3. 建一个 `ProjectAsset`（AVATAR_PREVIEW, UNCATEGORIZED, VIDEO, COMPLETED）
4. 逐 scene 更新 ProjectAvatar：`assetId`(共享), `reused=TRUE`, `renderStatus=COMPLETED`, `mediaType=VIDEO`, `duration=ref.duration`, `startTime=累加偏移`, `endTime=偏移+duration`
5. `reportAvatarCreditCost(project, 总偏移)`

**Advance 逐场景成功**（`handleAdvancedPreviewAvatarSuccess:828-936`）：

- 每 scene 一个 VislaMedia + ProjectAsset + speaking 音频资产 + MediaSyntheticVoice
- `realDuration = min(max(lastWordEndTimeMs, durationMs), sceneDuration)`
- 更新 ProjectAvatar：`assetId, reused=TRUE, renderStatus=COMPLETED, duration=realDuration, startTime=0, endTime=realDuration`
- 调 `processAdvanceAvatarAudio` 字幕重对齐
- 按 videoFormat 调 `resetTo*Advance` / `fillDefaultPositionIn*` 重算位置
- **不计费**

**失败**（`handlePreviewAvatarFailed:811-826`）：全项目 avatar `renderStatus=FAILED` + errorLog + projectNotify，不抛异常（avatar 失败不影响主流程）。

### 4.7 TTS → Avatar 完整时序

📖 业务见 README §4.7

**入口**：`POST /v3/{projectUuid}/voice/synthetic` → `SyntheticVoiceMetadataService.generateSyntheticVoice:684`

**finally 块**（`:810-828`）：

- `forAllProject=true` → `releasePreviewVideo(project, null)`（全项目 avatar→image）
- 指定 sceneIds → `releasePreviewVideo(project, sceneIds)` + `resetContinuousAvatarPreviewVideo`

**releasePreviewVideo**（visla-api `ProjectAvatarService:428-453`）落库：
`assetId=0, startTime=0, endTime=0, mediaType=IMAGE, renderStatus=INIT, reused=FALSE`（保留 lookId/位置参数）

**resetContinuousAvatarPreviewVideo**（`:1304-1317`）：

- **ADVANCE look 直接 return**
- 否则 `getContinuousSceneIdsByLookId`（向前向后找连续同 look scene，断开即停）→ `reused=FALSE, renderStatus=INIT`

**两条 TTS 路径**：

| | Azure | OpenAI/MiniMax |
| --- | --- | --- |
| visla-api 出口 | `sendSyntheticVoiceAfterTransactionCommit` → SQS `synthetic-voice-req` | `createAudioByAiTts:871` → `aiGateWayClient.makeSpeech` |
| vcjs 消费 | `SyntheticVoiceConsumer` → `regenerateVoiceByAzure:178` → `createNewSyntheticVoiceResourcesSync:393` | `AudioWordTimestampConsumer` → `processAlignerBos:85` |
| scene 时长更新 | `updateProjectTranscriptWithWordsTimeStamp:568`（resetSceneTime + triggerSceneDurationChange） | `VoiceOverAlignerHandlerService.insertSceneVoice`（parseSceneData） |
| avatar duration 同步 | `syncAvatarDurationsFromScenes:486` ✅ | `syncAvatarDurationsFromScenes:117` ✅ |
| 触发 avatar 视频重生成 | ⚠️ **regenerate 不触发**（仅 `createVoice:325`→`checkOrTriggerVatarGeneration`） | ✅ finally `generatePreviewAvatarVideos:153` |

**syncAvatarDurationsFromScenes**（vcjs `:454-472`）：只改 `duration/endTime/modifiedAt`（= scene.duration），不改 startTime/assetId/mediaType/renderStatus/reused。

**generatePreviewAvatarVideos 触发条件**（OpenAI/MiniMax finally `:130-163`）：

1. agentProjectId > 0
2. `canUseAvatar()`（= isGenerateAvatar && isPreviewAvatar）
3. `queryProjectAvatarListOnShow` 非空 且 `progressStatus != EDITING(20)`

### 4.8 导出 avatar 组装

📖 业务见 README §4.8

**预检**（`ProjectSceneCheckService.checkAvatarRenderStatus:223-233`）：所有 avatar `renderStatus=COMPLETED`，否则抛 `PROJECT_HAS_NOT_YET_FINISHED_PROCESSING`。在 exportScenes2Video / exportScenes2VideoForAdobe / exportScene2Pdf 均调用。

**组装**（`ProjectExportVideoService.constructExportSceneAvatar:1135-1228`）：

- 遍历 `getAssembleByProjectId`，匹配 sceneId，`visible=FALSE` 跳过，**第一个匹配后 break**（一 scene 一 avatar）
- 建 `ExportSceneAvatarBo{zindex, metadata(ProjectAvatarLayerMetadataVo), avatarParts}`
- `assetId>0` → `avatarParts=[AvatarPartTimeBo{assetId, assetTime=[MediaTimeBo{start,end,duration}]}]`
- `assetId=0` → avatarParts 空 → `exportVideo.setNeedDoAvatar(true)`
- `scene.setAvatar(avatarBo)`

> advance/basic 导出无差异（lookType/shotComposition/environmentImage 字段已注释）。`AvatarLayerHandler.doAddLayerToExportScene` 为空实现（AVATAR 不在 SCENE_EXPORT 能力集，走专用 `scene.avatar` 路径）。

### 4.9 image → video 转化方法清单

📖 业务见 README §6

**A 类（显式转 IMAGE）**：`initProjectAvatarData:306` / vcjs `buildProjectAvatar` / `replaceAvatarLook:208` / `replaceAvatarLookByGroupId:253` / `replaceSceneAvatarLook:460` / `addOrReplaceSceneAvatarLook:479`(替换) / `releasePreviewVideo:463` / `resetAvatarPreviewVideoInEditScene:1287` / `adjustAvatarInMergeScene:908`(fallback) / `adjustAvatarInSplitScene:968,982`(fallback)

**B 类（reused=FALSE + INIT）**：`resetContinuousAvatarPreviewVideo:1314` / `resetAvatarPreviewVideoInMergeScene:1274` / `changeDeletedProjectAvatar:1448`

**重生成判定**（`regenerateUnrenderedAvatars:599-640`）：`assetId==null || assetId==0 || reused==FALSE`

### 4.10 计费链路

📖 业务见 README §7

`reportAvatarCreditCost`（vcjs `:1063-1080`）：

```
seconds = abs(totalDuration / 1000)   // 毫秒÷1000
vislaEventBus.post(ResUsageRptEvent.avatarPreviewEvent(agentProjectId/null, null/projectId, seconds))
```

- operationType=`Avatar lip sync video render`，resource type=`lip_sync`，unit=`second`，amount=seconds
- agent project → entityType=AGENT_PROJECT；普通 → PROJECT
- 消费：`ResUsageRptEventConsumer` → `CreditConsumeHandleService.handleMessage` → 按 feePlan 单价 × 秒扣 workspace credit
- ⚠️ 仅 basic `handlePreviewAvatarSuccess` 计费，advance 不计费

---

## 5. AI 消息类型（AiMessageTypeEnum）

| key | visla-api name | vcjs name | 用途 |
| ----- | --------------- | ----------- | ------ |
| 3000 | avatar_character_image | avatar_character_image | 生成角色推荐图 |
| 3001 | avatar_look_video | avatar_look_video | 生成 look 视频 |
| 3002 | avatar_background_image | ⚠️ avatar_wall_paper | 壁纸（name 不一致） |
| 3003 | avatar_create_scenes | avatar_create_scenes | 场景级 advance avatar 视频 |
| 3004 | —（缺失） | avatar_change_voice | 变声（vcjs 独有） |

---

## 6. 回调数据结构

| 结构 | 用途 | 关键字段 |
| ------ | ------ | --------- |
| `AvatarProcessCommonCbBo` | matting 回调 | entityId, foregroundFileKey, mattingFileKey, thumbnailFileKey, transparentImageFileKey, width, height |
| `AvatarProcessSampleCbBo` | sampling 回调 | entityId, foregroundLink, mattingLink |
| `AdvanceLookCbBo` | advance look 回调 | lookId, assets[]{assetId, assetUrl, assetMattingUrl, originAssetUrl, assetType, mediaType, shotType, metadata{title,desc,tags}, resolution{w,h}} |
| `ChangeAdvancedAvatarVoiceCbBo` | 变声回调 | entityId, videoUrl, audioUrl, durationMs |
| `ProjectAvatarPreviewCbBo.AvatarAssetMetadataBo` | basic 预览回调 | assetId, assetUrl, assetMattingUrl, **segmentAudiosUrl**(分段) |
| `ProjectAvatarPreviewCbBo.AdvancedAvatarBo` | advance 预览回调 | scene{sceneId, footage{assetId,assetOriginUrl,assetUrl,assetMattingUrl,resolution}, ttsInfo{audioUrl,tsUrl,durationMs,lastWordEndTimeMs}} |
| `AvatarSceneRefBo` | 跨场景分段 | sceneId, duration |

---

## 7. Controller 接口清单

### 7.1 WorkspaceAvatarController（`/v3/{workspaceUuid}/...`）

| 方法 | 路径 | 说明 |
| ------ | ------ | ------ |
| getAvatarDetail | GET `/{avatarUuid}` | avatar 详情 |
| getAvatarList | GET `/` | 列表（category 分 public/private） |
| saveAvatar | POST `/` | 创建（4 来源） |
| renameAvatar | PUT `/{avatarUuid}` | 重命名 |
| deleteAvatar | DELETE `/{avatarUuid}` | 删除 |
| saveDescribe | POST `/save-describe` | GENERATE 描述 → 推荐图 |
| generatePreviewLook | POST `/{avatarUuid}/looks` | 生成 look（basic+advance） |
| regeneratePreviewLook | POST `/{avatarUuid}/looks/regenerate` | 重试 look |
| getAvatarLookList / Group | GET | look / group 列表 |
| renameAvatarLook / Group | PUT | 重命名 |
| deleteAvatarLook / Group | DELETE | 删除 |
| changeLookVoice | PUT `/{avatarUuid}/looks/voice` | 变声 |
| importAvatarFromClip | POST `/import` | 从 clip 导入 |
| getRecommendImages / generateByRecommendImage / editRecommendImage / deleteRecommendImage | — | 推荐图管理 |

### 7.2 ProjectEditingV3Controller

| 方法 | 路径 | 说明 |
| ------ | ------ | ------ |
| regenerateUnrenderedAvatars | PUT `/regenerate-unrendered-avatars` | 重生成（assetId=0 或 reused=FALSE） |
| deleteUnrenderedAvatars | DELETE `/cleanup-unrendered-avatars` | 删除（assetId=0） |
| replaceAvatarLook | — | 全项目换 look |
| replaceAvatarLookByGroupId | — | 按 group 换 look |

### 7.3 ProjectLayerV3Controller（scene 级）

addOrReplaceSceneAvatarLook / replaceSceneAvatarLook / applyAvatarToAllScenes

### 7.4 SyntheticVoiceV3Controller

| 方法 | 路径 | 说明 |
|------|------|------|
| generateSyntheticVoice | POST `/v3/{projectUuid}/voice/synthetic` | TTS 生成（Azure/OpenAI/MiniMax） |
| textToSpeech | POST `/v3/textToSpeech` | 单次文本转语音 |

---

## 8. 已知风险与不对称点

📖 业务见 README §10

1. **Azure regenerate voice 不触发 avatar 重生成**（`regenerateVoiceByAzure` 不调 `checkOrTriggerVatarGeneration`，仅 `createVoice` 调）。改 Azure 链路注意。
2. **ADVANCE look 连续链重置被跳过**（`resetContinuousAvatarPreviewVideo` / `resetAvatarPreviewVideoInMergeScene` 开头 `if ADVANCE return`）。
3. **枚举不同步**：`AvatarProcessEnum`（vcjs 多 ADVANCED_AVATAR）；`AiMessageTypeEnum`（3002 name 不同，3004 vcjs 独有）。违反 AGENTS.md §4.1 双向同步要求。
4. **advance 预览不计费**（仅 basic 计费）。
5. **`processAdvanceAvatarAudio` 中 `syncAvatarDurationsFromScenes` 被注释**（`AudioWordTimestampService:211`），advance 回调用 `realDuration` 单独更新。
6. **visla-api / vcjs 都有 `generateBasic/AdvancePreviewAvatarVideos`**：visla-api 版是同步用户入口（返回 Boolean，有批量查询/壁纸 metadata 校验优化），vcjs 版是异步主链路（void，消息驱动）。
7. **look video 回调后 processStatus 仍 PROCESSING**：需 matting+sampling 完成才 COMPLETED。
8. **场景裁剪 `adjustTimeInCutScene` 不触发重生成**：只改 offset/duration，duration<=0 删除。
9. **`applyAvatarToAllScenes` 的 look 同步被注释**（2025-08-07）：现在只同步位置，不同步 look、不转 image。

---

*本文档基于源码梳理，业务规则见 [README.md](./README.md)。行号会随代码演进漂移，以实际代码为准。*
