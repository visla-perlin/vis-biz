# Video 生成接口的 Voice 流程

> 范围：visla-api 编辑态三个 video 生成接口（animate-storyboard / generate-video / regenerate-video）如何接入 voice，以及与 vcjs OpenAPI 自动生成的差异  
> 视角：接口流程 + voice 接入判定 + 一段式 vs 两段式  
> 关联：[scene-kit-voice-flow.md](./scene-kit-voice-flow.md)（character voice 内部子流程）、[tts-flow.md](./tts-flow.md)（API 端 TTS 主流程）

---

## 0. 一句话定义

video 生成接口是否生成「角色配音（character voice）」，由入参 `kitUuid` 决定，最终落到一个布尔值 `waitingRefAudio`：为 `true` 时不在本接口内直接生成 video，而是交给 scene kit voice flow（TTS → AIGC_REDO）；为 `false` 时立即调 AI gateway 生成 video。

---

## 1. 判定：是否走 Character Voice

三个接口共用同一套判定逻辑，核心是 `kitUuid`。

### 1.1 deliveryMode 决策（`resolveSpeechPlanBase`）

```
传了 kitUuid（非空非 0）
  → 校验 kit 存在 + 归属 project + kit.voiceId 有效（validateKitVoice）
  → buildSpeechPlanBase(kit)
  → deliveryMode = ON_CAMERA_SPEAKER          ← character voice

没传 kitUuid
  → resolveBlankKitDeliveryMode(scene)
  → scene 有台词 → OFF_SCREEN_VO              ← 画外音（非 character voice）
  → scene 无台词 → SILENT                      ← 静音（非 character voice）
```

### 1.2 character voice 接入（`applySceneKitAigcRelation`）

```
kitUuid 为空 或 speechPlan.deliveryMode != ON_CAMERA_SPEAKER
  → return false                               ← 不走 character voice

否则
  → ensureAigcRefAudioRelation(project, scene, kit)
     ├─ isUsableRelation 判断已有 relation 能否复用
     │   （同 kit + 同 voice + SUCCESS + audioUrl 非空 + 台词 MD5 未变）
     │   可复用 → 返回 Pair(relation, false)    ← 不重做 TTS，但仍绑 kit
     │   不可复用 → 新建 relation（latest=true，旧→STALE）
     │              + enqueue PROJECT_SCENE_KIT_REFAUDIO_FLOW task
     │                （flow=AIGC_REDO, batch_source=AIGC_VIDEO）
     │              → 返回 Pair(relation, true) ← 要（重）做 TTS
  → paramBo 写入 projectKitsDetailId / projectSceneKitVoiceId
  → return waitingRefAudio（= 是否新建了 relation 要等 TTS）
```

### 1.3 `waitingRefAudio` 是分水岭

| waitingRefAudio | 含义 | 后续动作 |
| :--: | --- | --- |
| `false` | 不走 character voice，或 relation 可直接复用 | **立即** `callAigcGatewayAndSyncStatus` → `createVideo` |
| `true` | 走 character voice 且需（重）做 TTS | `markPreparingRefAudioIfNeeded`（asset/rel → `PREPARING_REFAUDIO`），**不调 AI gateway**，交 vcjs scene kit voice flow |

`markPreparingRefAudioIfNeeded`：asset.processStatus = rel.processStatus = `PREPARING_REFAUDIO`，assembleBo.waitingRefAudio = true。  
`callAigcGatewayAndSyncStatus` 开头 `if (waitingRefAudio) return`，所以 `waitingRefAudio=true` 时 video 不会在此生成。

---

## 1.5 ref_audio 注入机制（三层，容易误解）

> ⚠ **关键**：`applySceneKitAigcRelation` 只标记 `waitingRefAudio`（是否要等 TTS），**不负责注入 ref_audio**。ref_audio 的真正注入在 `getSceneVoiceMetadataMap` 里，这是最容易被忽略的一层。

### 问题背景

character voice 的 motion video 生成时，AI gateway 的 `createAigcMotionVideos` 消息需要带 `ref_audio`（口型同步音频）。这个 `ref_audio` 从哪来、什么时候注入、由谁负责，分散在三个方法里，容易误解为「只有 TTS 完成后才有 ref_audio」。

### 三层注入（`assembleAigcBo` 内部）

```
assembleAigcBo（每个 scene）
  ├─ ① applySceneKitAigcRelation（ProjectSceneService:944）
  │   → ensureAigcRefAudioRelation：判断 relation 能否复用
  │     ├─ 可复用（voice 一致 + SUCCESS + 台词未变）→ waitingRefAudio=false
  │     │   只设 paramBo.projectKitsDetailId / projectSceneKitVoiceId
  │     │   ※ 不注入 ref_audio，只标记「不需要等 TTS」
  │     └─ 不可复用 → waitingRefAudio=true，新建 relation + enqueue TTS task
  │
  ├─ ② getSceneVoiceMetadataMap（assembleAigcBo 调用）
  │   → applySceneKitVoiceRefAudio（ProjectSceneService:748）
  │     → 查 project_scene_kit_voice 表（queryLatestBySceneIds）
  │     → 过滤 audioStatus=SUCCESS + audioUrl 非空的 relation
  │     → 【ref_audio 的真正注入点】用 relation.audioUrl 组装 RefAudio
  │     → 写入 Map<sceneId, RefAudio>
  │   ※ 这是「voice 相同直接复用」的执行点
  │
  └─ ③ saveAndAssembleAigcData（用 ② 的 RefAudio）
      → 如果 ② 拿到了 kit voice ref_audio，createVideo 的消息就带 ref_audio
      → AI 收到带 ref_audio 的消息 → 做口型同步
```

### 三种情况对照

| 情况 | waitingRefAudio | ref_audio 来源 | createVideo 消息 | deliveryMode |
| --- | :--: | --- | --- | --- |
| 不走 character voice（无 kit） | false | 项目默认 voice（`getSceneVoiceMetadataMap` 非 kit 路径） | 带 voiceover ref_audio | OFF_SCREEN_VO / SILENT |
| 走 character voice，relation 可复用（voice 一致） | **false** | **`applySceneKitVoiceRefAudio` 从 relation 注入** | **带 kit ref_audio** | ON_CAMERA_SPEAKER |
| 走 character voice，relation 不可复用（voice 变了 / 首次） | true | 无（TTS 还没做） | **不调 createVideo**，等 vcjs TTS | ON_CAMERA_SPEAKER |

### 核心结论

- **「voice 相同可以直接用」**：对应第二种情况。`waitingRefAudio=false`，scene 进入 `readyBoList`，ref_audio 由 `applySceneKitVoiceRefAudio` 从已完成的 relation（audioUrl 已就绪）注入，createVideo 一次生成带口型同步的 motion video，**不需要重做 TTS**。
- **「voice 不同需要 TTS」**：对应第三种。`waitingRefAudio=true`，跳过本次 createVideo，交 vcjs scene kit voice flow（TTS → AIGC_REDO → `triggerSceneKitMotionVideo` 带新 ref_audio 生成）。
- **AI 接口始终是同一个** `createAigcMotionVideos`（`/gateway/ai/aigc/video/generations`），不是两条不同的 AI 路径。区别只在消息里带不带 ref_audio + deliveryMode。

---

## 2. 三个编辑态接口

### 2.1 `POST /animate-storyboard`（批量：storyboard 静帧 → motion video）

- Controller：`ProjectEditingV3Controller.java:403`（无 `@ProjectOperateLockRequired`，与另两个不同）
- 两个重载（由 reqBody 是否带 scenes 决定）：
  - 无 scenes（`ProjectSceneService:1233`）：全量 scene
  - 有 scenes（`ProjectSceneService:1304`）：指定 scene 列表，每个 scene 是 `SaveVideoFromAigcReqBody`（**可带 kitUuid**）

流程：
```
clearFlagBoListPair 组装 List<AigcAssembleBo>
  ├─ 每个 bo：resolveSpeechPlanBase(kitUuid) → applySceneKitAigcRelation → waitingRefAudio
  └─ assembleAigcBo 组装 aigc + markPreparingRefAudioIfNeeded
checkCredit（按总时长）
setProjectProcessingAiStoryboard
afterCommit:
  readyBoList = boList.filter(!waitingRefAudio)      ← 等 TTS 的 scene 先排除
  createVideo(readyBoList)                            ← 只生成「不走 CV / 可复用」的
  readyBoList 状态置 UPLOADING / FAILED
```

**character voice 的 scene**：waitingRefAudio=true，不参与本次 `createVideo`，video 生成完全交给 vcjs（TTS → AIGC_REDO → `triggerSceneKitMotionVideo`）。

### 2.2 `POST /scene/generate-video`（单 scene 首次生成）

- Controller：`ProjectEditingV3Controller.java:648`，带 `@ProjectEditFlagAnnotation` + `@ProjectOperateLockRequired(needEditLock=true)`
- 入参 `SaveVideoFromAigcReqBody`：prompt / genModel / videoLength / aspectRatio / sceneUuid / refImages / **enableAudio** / **kitUuid** / motionVideoMode
- Service：`generateAiVideo`（`ProjectSceneService:3584`）

流程：
```
ACL + 用户校验
findAndAssertScene
enableAudio = !aiStoryboardEnable
checkUnfinishedSceneRecords
checkAndAssembleReqBody
assembleAigcBo(:3520)
  ├─ 新建 vislaMedia + asset（VIDEO）
  ├─ paramBo.setSpeechPlan(resolveSpeechPlanBase(kitUuid))
  ├─ waitingRefAudio = applySceneKitAigcRelation(paramBo, ..., kitUuid)
  ├─ saveAndAssembleAigcData
  └─ markPreparingRefAudioIfNeeded
if (!waitingRefAudio)
  afterCommit → callAigcGatewayAndSyncStatus → createVideo       ← 立即生成
```

### 2.3 `POST /scene/regenerate-video`（单 scene 重生成，复用 asset）

- Controller：`ProjectEditingV3Controller.java:694`，注解同 generate-video，额外校验 `aigcUuid` 非空
- Service：`reGenerateAIVideo`（`ProjectSceneService:3615`）
- 与 generate-video 的**关键区别是复用已存在的 aigc/rel/asset/media**：

```
取 existAigc（按 aigcUuid）+ existRel（getOneActiveRel）
  └─ 校验归属 project + scene，mediaType=VIDEO
取复用的 asset + vislaMedia
paramBo.setSpeechPlan(resolveSpeechPlanBase(kitUuid))
waitingRefAudio = applySceneKitAigcRelation(paramBo, ..., kitUuid)
projectAssetService.retryAigc(vislaMedia, asset, existAigc, existRel, paramBo)   ← 复用 asset 重置
assembleAigcData（不新建 asset，复用 existAigc/existRel）
markPreparingRefAudioIfNeeded
if (!waitingRefAudio)
  afterCommit → callAigcGatewayAndSyncStatus → createVideo
```

---

## 3. 一段式 vs 两段式（核心认知）

这是本文档最重要的结论，区分了两条生成路径的本质差异。

### 3.1 一段式（visla-api 编辑态接口 + kitUuid）✅ 正确实现

接口带 `kitUuid` 时，character voice 的 TTS **前置**，video 只生成一次：

```
[visla-api] assembleAigcBo
  → aigc 首次创建即：
     deliveryMode = ON_CAMERA_SPEAKER
     enableAudio  = true
     projectKitsDetailId     = kit.id
     projectSceneKitVoiceId  = relation.id
     refImages（FIRST_FRAME / LAST_FRAME）正确写入
  → asset/rel → PREPARING_REFAUDIO
  → enqueue AIGC_REDO task
  → 本接口不调 createVideo（waitingRefAudio=true）

[vcjs] scene kit voice flow
  → TTS（用 kit.voiceId）→ SUCCESS
  → AIGC_REDO → triggerSceneKitMotionVideo（带 ref_audio，一次生成带口型同步的 motion video）
```

**特点**：motion video 只生成 1 次，TTS 前置，不存在「先生成无配音 video 再丢弃重做」的浪费。

> 📌 **线上日志已证实**（animate-storyboard 带 4 个 kit scene）：全程无 `aiGateWayClient.createVideos` 调用，4 个 scene 的 motion aigc 首次创建即 `ON_CAMERA_SPEAKER + enable_audio=true + 绑 kit`，relation 用 `kit.voiceId`，asset/rel 状态为 `PREPARING_REFAUDIO`，等待 vcjs TTS 后一次生成。

### 3.2 两段式（vcjs OpenAPI 自动 `triggerMotionVideos`）⚠️ 当前设计

OpenAPI pipeline 的 `CREATE_MOTION_VIDEO` 阶段（`triggerMotionVideos`）走两段式：

```
第一段：triggerMotionVideos（vcjs AigcMotionVideoService:153）
  → createMotionVideoRel(... sceneKitVoice = null)   ← 硬编码 null
  → deliveryMode = OFF_SCREEN_VO / SILENT
  → 不绑 kit，不带 ref_audio
  → createVideos（storyboard 静帧 → 无配音 motion video）

第二段：scene kit voice flow（如果需要 character voice）
  → TTS → AIGC_REDO → triggerSceneKitMotionVideo
  → 把第一段的 motion video 丢弃，带 ref_audio 重新生成
```

**问题**：第一段生成的 motion video 在第二段会被丢弃重做，是当前设计最大的浪费；且第二段的 `triggerSceneKitMotionVideo` 依赖「已存在的 motion video」做输入图，首次场景下若 motion video 尚未回调完成（`visla_media.original_link` 为空），会 skip → FAILED。

### 3.3 两者对比

| 维度 | 一段式（编辑态 + kit / OpenAPI + kit） | 两段式（OpenAPI 自动，已废弃） |
| --- | --- | --- |
| motion aigc.deliveryMode | `ON_CAMERA_SPEAKER` | `OFF_SCREEN_VO` / `SILENT` |
| 是否绑 kit | ✅ | ❌（sceneKitVoice 硬编码 null） |
| 是否 TTS 前置 | ✅ `ensureAigcRefAudioRelation` / `createRelationForMotionVideo` | ❌ 直接发 AI |
| motion video 生成次数 | 1 次（TTS 后直接带 ref_audio） | 2 次（先无配音，再 AIGC_REDO 重做） |
| 输入图 | refImages（FIRST_FRAME / LAST_FRAME） | storyboard 静帧（第一段）/ 已生成 motion video（第二段） |

> ✅ **已实现**：vcjs OpenAPI 自动路径（`triggerMotionVideos`）已改为一段式——`prepareMotionVideoRels` 按 storyboard.metadata 的 kit 设置分流，有 kit 的 scene 走 character voice（建 relation + TTS + 首次带 ref_audio 生成），无 kit 的 scene 走原批量路径。详见 §8.1。
>
> 同时 visla-api OpenApi `generate-motion-video` 接口已支持 `kit_uuid` 入参（详见 §2.2）。

---

## 4. 不走 Character Voice 的完整流程

`kitUuid` 为空（或 relation 可复用）时，三接口共用：

```
[visla-api]
resolveSpeechPlanBase → OFF_SCREEN_VO（有台词）/ SILENT（无台词）
applySceneKitAigcRelation → return false（waitingRefAudio=false）
saveAndAssembleAigcData（aigc.deliveryMode = OFF_SCREEN_VO/SILENT，无 projectKitsDetailId）
callAigcGatewayAndSyncStatus
  → aiGeneratedContentService.createVideo
  → 调 AI gateway（createVideos / createImageFromAigc）
  → asset/rel → UPLOADING

[vcjs] AI 生成完成回调
  → AigcVideoConsumer.handleMessage（ai-aigc-video-cb.fifo）
  → AigcProjectAssetMessageHandler.handleSuccuss
  → processVislaAssetInitActions（回填 visla_media.original_link）
  → asset COMPLETED → ProjectAssetPreprocessCompletedEvent
  → onProjectAssetCompleted → tryCompleteSyntheticPipeline
  → project → EDITING
```

特点：**不碰 `project_scene_kit_voice` 表，不做 TTS，不经过 scene kit voice flow**，voice 来自项目默认配音（`getSceneVoiceMetadataMap`）。

---

## 5. 走 Character Voice 的完整链路（三接口 + vcjs 协作）

`kitUuid` 有效且 relation 需新建时：

```
[visla-api]
applySceneKitAigcRelation
  → ensureAigcRefAudioRelation
     ├─ 新建 project_scene_kit_voice（latest=true, INIT, voiceId=kit.voiceId）
     ├─ enqueue sys_task: type=PROJECT_SCENE_KIT_REFAUDIO_FLOW
     │            content={refaudio_flow:REFAUDIO_THEN_AIGC_REDO,
     │                     batch_source:AIGC_VIDEO, relation_id}
     └─ 旧 relation → STALE
  → asset/rel.processStatus = PREPARING_REFAUDIO
  → 不调 createVideo（waitingRefAudio=true）

[vcjs] XXL-Job: ProjectSceneKitRefAudioFlowJob
  → processRefAudioFlow(task)（ProjectSceneKitVoiceTaskService:167）
  → prepareRelationAudio（TTS，:344）
     ├─ voiceId = relation.voiceId ?: kit.voiceId
     ├─ transcriptText = 发音纠正(scene.transcript, voice.locale)
     ├─ Azure → 同步生成 → SUCCESS
     └─ OpenAI/Minimax → 异步 make_speech → AI callback → SUCCESS
  → continueAfterRefAudioPrepared → continueAfterAudioAligned
  → flow = REFAUDIO_THEN_AIGC_REDO
     → triggerSceneKitMotionVideoAfterCommit
     → triggerSceneKitMotionVideo（AigcMotionVideoService:271）
        ├─ 找 motionRel（PREPARING_REFAUDIO rel，或基于 selected VIDEO rel 新建）
        ├─ deliveryMode = ON_CAMERA_SPEAKER, enableAudio = true
        ├─ ref_audio = relation.audioUrl + audioRange(0,duration) + voiceOver(transcript)
        ├─ copyMotionVideoRefImages（复制 refImages）
        └─ aiGateWayClient.createVideos（带 ref_audio + speech_plan）

[vcjs] AI 视频生成完成回调
  → AigcVideoConsumer → handleVideoCallback（定位 relation，success/fail task）
  → onSceneKitVoiceCompleted → onSceneKitVoiceBatchTerminal
  → finalizeSceneKitBatch（AIGC_REDO 批）
  → completeSyntheticVoiceProject → project → EDITING
```

> character voice 内部的状态机、三种 RefAudioFlow、isUsableRelation 复用判断详见 [scene-kit-voice-flow.md](./scene-kit-voice-flow.md)。

---

## 6. Regenerate Voice（换 kit 音色）

「regenerate voice」若指**换 kit 的 voice 而非重生成 video**，是独立接口 `PUT /kit/{kitUuid}/change-voice`（`ProjectEditingV3Controller:1655`），不在上述三个接口内。

- Service：`changeProjectKitVoice`（`ProjectSceneKitVoiceService:190`）

流程：
```
校验目标 voice 有效（validateProjectVoice，含 custom voice workspace 归属）
更新 kit.voiceId = voiceId（project_kits_detail）
查该 kit 所有 latest=true 的 relation → 涉及哪些 scene
筛出「有 selected aigc video rel」的 relation（affected）
setProjectProcessingCharacterVoice
对每个 affected relation:
  ├─ audioScriptUpdated = isRelationAudioScriptUpdated(scene, relation, targetVoice)
  ├─ refAudioFlow 选择:
  │   ├─ selectedRel.type == SYSTEM        → REFAUDIO_THEN_AUDIO_ALIGN（只对齐音频）
  │   ├─ audioScriptUpdated == false       → REFAUDIO_THEN_CHANGE_VOICE（换声音，不重做画面）
  │   └─ audioScriptUpdated == true        → REFAUDIO_THEN_AIGC_REDO（重做画面 + 口型同步）
  ├─ createRefaudioRelation（新建 relation, voiceId=新voiceId）
  └─ enqueue task（batch_source=CHANGE_PROJECT_KIT_VOICE）
afterCommit → pushProjectSceneListRefresh 通知
return ProjectKitChangeVoiceVo(batchId, affectedCount)
```

三种 flow 的下游动作（vcjs TTS 完成后）：

| Flow | 动作 | 是否重做画面 | vcjs 方法 |
| --- | --- | :--: | --- |
| AUDIO_ALIGN | 把 ref_audio 写入 scene 的 SYNTHETIC_VOICE_OVER | 否 | finalize（批量 audioalign SQS） |
| CHANGE_VOICE | `triggerSceneKitChangeVoice`（AI batch voice conversion，换现有 video 的声音） | 否（声音替换） | `AigcMotionVideoService:592` |
| AIGC_REDO | `triggerSceneKitMotionVideo`（带 ref_audio 重做 video，口型同步） | **是** | `AigcMotionVideoService:271` |

配套预估接口：
- `GET /kit/{kitUuid}/change-voice/estimate-credit`（`:1674`）→ `estimateChangeVoiceCredit`
- `GET /scene/kit-voice/estimate-transcript-redo-credit`（`:1689`）→ `estimateTranscriptRedoCredit`（改台词重做预估）

---

## 7. 接口总览对比

| 维度 | animate-storyboard | generate-video | regenerate-video |
| --- | :--: | :--: | :--: |
| 粒度 | 批量（全量/指定 scenes） | 单 scene | 单 scene |
| asset/aigc | 新建 | 新建 | **复用** existAigc/existRel |
| 必填 | projectUuid | sceneUuid | sceneUuid + **aigcUuid** |
| kitUuid 来源 | reqBody.scenes[].kitUuid | reqBody.kitUuid | reqBody.kitUuid |
| character voice 判定 | resolveSpeechPlanBase + applySceneKitAigcRelation | 同左 | 同左 |
| 不走 CV | filter 后 createVideo | createVideo | createVideo |
| 走 CV（新建 relation） | waitingRefAudio=true，排除出 readyBoList，交 vcjs AIGC_REDO | 不调 createVideo，交 vcjs AIGC_REDO | 同左 |
| 走 CV（复用 relation） | 参与 createVideo（带复用 ref_audio） | createVideo | createVideo |
| 锁注解 | **无**（注：与另两个不一致，待确认是否需补） | needEditLock=true | needEditLock=true |

---

## 8. 已知问题与实现进展

### 8.1 ~~OpenAPI 自动生成走两段式，character voice 场景浪费一次生成~~ ✅ 已修复

- **原现象**：vcjs `triggerMotionVideos` 首次生成无配音 motion video（OFF_SCREEN_VO/SILENT），需要 character voice 时第二段 AIGC_REDO 丢弃重做。
- **修复**：`prepareMotionVideoRels` 按 storyboard.metadata 的 `speechPlan` 分流——有 kit 的 scene 调 `createRelationForMotionVideo` 建 relation + enqueue AIGC_REDO，motionRel 标记 PREPARING_REFAUDIO，不参与本次批量 createVideos；TTS 完成后由 `triggerSceneKitMotionVideo` 首次带 ref_audio 生成。无 kit 的 scene 走原批量路径。
- 代码位置：`AigcMotionVideoService.prepareMotionVideoRels` + `ProjectSceneKitVoiceService.createRelationForMotionVideo`（vcjs）。

### 8.2 AIGC_REDO 在首次场景下找不到 motion video 输入图 ✅ 已规避

- **原现象**：`triggerSceneKitMotionVideo` 用 motionRel 的 assetLink 做 input 图，首次场景下 `visla_media.original_link` 为空 → skip → FAILED。
- **根因澄清**：`initVideoMedia` 新建的 vislaMedia 有 S3 占位 originalLink（非空），`ProjectAssetWraperService` 合并后 assetLink 非空。原 FAILED 发生在 visla_media 异常初始化或被清空的特殊情况。
- **一段式规避**：`prepareMotionVideoRels` 在建 motionRel 后立即 `copyMotionVideoRefImages`（复制 storyboard 的 FIRST_FRAME/LAST_FRAME），确保 `triggerSceneKitMotionVideo` 的 `buildAnimateAsset` 有 refImages 可用；且 motionRel 的 S3 占位 assetLink 保证 :875 检查不 skip。

### 8.3 全 kit 场景的 fetch pipeline 收尾靠轮询（已知限制）

- **现象**：OpenAPI pipeline 中若所有 scene 都走 character voice（全 kit），`triggerMotionVideos` 返回空 list，`handleTriggerSceneMotion` 仍发 `CHECK_MOTION_COMPLETE_AND_SEND_WEBHOOK`。CHECK 轮询会发现 motionRel 处于 PREPARING_REFAUDIO（非终态）→ 持续 requeue 直到 kit scene 的 motion 通过 scene kit voice flow 回调完成（rel → COMPLETED）→ CHECK 发现全部终态 → 收尾。
- **影响**：全 kit 场景的完成通知靠 CHECK 轮询发现状态，不是事件驱动（`onSceneKitVoiceBatchTerminal` 与 fetch pipeline 的协调未打通）。功能正确，但有轮询延迟。
- **优化方向**：让 `onSceneKitVoiceBatchTerminal` 在全 kit 批次完成时主动驱动 fetch pipeline 完成（替代 CHECK 轮询）。


---

## 9. 代码索引

### visla-api

| 文件 | 职责 |
| --- | --- |
| `controller/v3/ProjectEditingV3Controller.java` | animate-storyboard(:403) / generate-video(:648) / regenerate-video(:694) / change-voice(:1655) / estimate-credit 端点 |
| `service/ProjectSceneService.java` | `generateAiVideo`(:3584) / `reGenerateAIVideo`(:3615) / `animateAiStoryboard`(:1233,:1304) / `assembleAigcBo`(:3520) / `resolveSpeechPlanBase`(:820) / `applySceneKitAigcRelation`(:917) / `markPreparingRefAudioIfNeeded`(:3539) / `callAigcGatewayAndSyncStatus`(:3375) |
| `service/ProjectSceneKitVoiceService.java` | `ensureAigcRefAudioRelation`(:132) / `changeProjectKitVoice`(:190) / `isUsableRelation`(:162) / credit estimate(:271,:284) |

### vcjs

| 文件 | 职责 |
| --- | --- |
| `service/AigcMotionVideoService.java` | `triggerMotionVideos`(:153，两段式第一段) / `triggerSceneKitMotionVideo`(:271，AIGC_REDO) / `triggerSceneKitChangeVoice`(:592，CHANGE_VOICE) / `createMotionVideoRel`(:1236) / `createSceneKitMotionVideoRelFromPrevious`(:402) |
| `service/xxl/ProjectSceneKitVoiceTaskService.java` | `processRefAudioFlow`(:167) / `prepareRelationAudio`(:344，TTS) / `continueAfterRefAudioPrepared`(:621) / `handleVideoCallback`(:249) |
| `service/ProjectSceneKitVoiceService.java` | `syncFromSpeechPlan`(:51) / `syncBatchFromScenes`(:94) / `syncFromProjectDefaultVoice`(:125) |
| `service/pipeline/ProjectPipelineCompletionCoordinator.java` | `onProjectAssetCompleted`(:79) / `onSceneKitVoiceBatchTerminal`(:144) / `tryCompleteSyntheticPipeline`(:252) |
| `service/pipeline/ProjectPipelineCompletionFinalizer.java` | `completeSyntheticVoiceProject`(:102) / `completeCreateVideoRefAudioProject`(:140) |
| `sqs/consumer/service/OpenApiProjectTaskFetchService.java` | `handleTriggerSceneMotion`(:1961) → `triggerMotionVideos`（OpenAPI pipeline CREATE_MOTION_VIDEO 阶段） |
| `sqs/consumer/service/AigcVideoConsumer.java` | AI 视频生成完成回调入口(:66) |
