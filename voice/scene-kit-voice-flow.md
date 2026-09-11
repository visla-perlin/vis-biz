# Scene Kit Voice（角色配音 / Character Voice）流程

> 范围：AIGC motion video 中，kit 角色（`on_camera_speaker`）的 voice 同步流程  
> 视角：业务规则 + 数据载体 + 链路 + 状态机  
> 关联：[tts-flow.md](./tts-flow.md)（API 端 TTS 主流程）、[aigc/aigc-prompt-metadata-rule.md](../aigc/aigc-prompt-metadata-rule.md)（metadata 契约）、[video-generation-voice-flows.md](./video-generation-voice-flows.md)（video 生成接口的 voice 流程 / 一段式 vs 两段式）

---

## 0. 一句话定义

当 scene 使用 kit 角色（`delivery_mode = on_camera_speaker`）时，需要用 **kit 绑定的 voice** 生成角色配音（ref_audio），再驱动 motion video 做**口型同步**。本文描述这条链路的判断条件、数据载体、三种执行策略（RefAudioFlow）以及与 motion video 的协作关系。

---

## 1. 何时需要 Character Voice

四个条件**全部满足**才触发：

| # | 条件 | 数据来源 |
| - | ---- | -------- |
| 1 | `motion_video.speech_plan.delivery_mode = on_camera_speaker` | metadata（候选）/ aigc.deliveryMode |
| 2 | `speech_plan.speaker_kit_index >= 0`（有绑定 kit） | metadata / aigc.projectKitsDetailId |
| 3 | 反查到的 `ProjectKitsDetail` 存在且 `status = ACTIVE`、归属当前 project | DB |
| 4 | `kit.voiceId` 非空（有绑定音色） | DB |

任一不满足 → 不做 character voice，退化为 `off_screen_vo`（画外音）或 `silent`（静音）。

> ⚠ **当前实现缺陷**：vcjs 首次触发 motion video（`createMotionVideoRel`）时 `sceneKitVoice = null`，**不会自动绑 kit**，导致上述条件 2/3 不成立。首次生成的 motion video `delivery_mode` 必为 `off_screen_vo`/`silent`。详见第 8 节。

---

## 2. 核心概念

| 概念 | 说明 |
| ---- | ---- |
| **Kit**（`ProjectKitsDetail`） | 角色/素材实体，含 `voiceId`（绑定音色）、`kitIndex`、`visualReference`、`name` |
| **Character Voice** | `on_camera_speaker` 模式下，用 kit 的 voice 生成 ref_audio，驱动 motion video 口型同步 |
| **delivery_mode** | 语音交付模式：`on_camera_speaker`（角色配音）/ `off_screen_vo`（画外音）/ `silent`（静音） |
| **ProjectSceneKitVoice** | scene × kit × voice 的绑定关系记录，承载 TTS 音频与状态（见第 3 节） |
| **RefAudioFlow** | ref_audio 就绪后的后续动作策略，共 3 种（见第 4 节） |

---

## 3. 数据载体

### 3.1 `project_scene_kit_voice`（ProjectSceneKitVoice）

scene 与 kit voice 的绑定关系，是 character voice 链路的核心载体。

| 字段 | 作用 |
| ---- | ---- |
| `project_kits_detail_id` | 绑定的 kit |
| `voice_id` | 使用的音色（来自 `kit.voiceId`） |
| `transcript_text` | 发音纠正后的台词文本 |
| `transcript_md5` | 台词 MD5，用于判断文案是否变化 |
| `audio_url` | TTS 生成的音频 URL |
| `duration` | 音频时长（ms） |
| `audio_status` | 音频状态机（见 3.2） |
| `batch_id` | 批次 ID，同批次一起触发 / 一起终态判定 |
| `latest` | 是否当前最新（同一 scene 仅一条 `latest=true`） |

### 3.2 状态机（`audio_status`）

```
INIT ──prepareRelationAudio──► PROCESSING ──TTS 成功──► SUCCESS
                                  │
                                  └──TTS 失败/超时──► FAILED

SUCCESS ──新 relation 创建──► STALE（markSceneRelationsNotLatest）
```

| 值 | 语义 |
| -- | ---- |
| `INIT(0)` | 新建，尚未生成 TTS |
| `PROCESSING(1)` | TTS 生成中 |
| `SUCCESS(2)` | TTS 完成，`audio_url` 可用 |
| `FAILED(3)` | TTS 失败 |
| `STALE(4)` | 已被新 relation 取代，不再生效 |

---

## 4. 三种 RefAudioFlow

ref_audio（TTS）就绪后，按 `refaudio_flow` 字段决定后续动作：

| Flow | 语义 | 执行动作 | 是否重新生成视频画面 | 典型场景 |
| ---- | ---- | -------- | :--: | -------- |
| `REFAUDIO_THEN_AUDIO_ALIGN` | 只对齐 voiceover | 把 ref_audio 写入 scene 的 `SYNTHETIC_VOICE_OVER` timeline | 否 | 只需要音频，不碰视频 |
| `REFAUDIO_THEN_CHANGE_VOICE` | 换现有 video 的声音 | `triggerSceneKitChangeVoice`（AI batch voice conversion） | 否（声音替换） | 已有 motion video，只换声音 |
| `REFAUDIO_THEN_AIGC_REDO` | 重做 AIGC video | `triggerSceneKitMotionVideo`（AI `createVideos`，带 ref_audio） | **是**（重新生成） | 需要口型同步，重新生成视频 |

### 4.1 如何选择 Flow

| 触发场景 | 选择的 Flow | 判断依据 |
| -------- | ----------- | -------- |
| storyboard 完成后自动同步（`syncFromSpeechPlan`） | `AUDIO_ALIGN` | 只需对齐音频 |
| 用户换 kit voice，文案未变（`changeProjectKitVoice`） | `CHANGE_VOICE` / `AUDIO_ALIGN` | `audioScriptUpdated = false` → CHANGE_VOICE |
| 用户换 kit voice，文案变了 | `AIGC_REDO` | `audioScriptUpdated = true` → AIGC_REDO |
| AIGC 生成需要 character voice（`ensureAigcRefAudioRelation`） | `AIGC_REDO` | motion video 需要带 ref_audio 重做 |

> `audioScriptUpdated` 判断：`transcript_modified_status = CHANGED` 或 `REFAUDIO_THEN_AIGC_REDO` flow 时为 `true`。

---

## 5. 完整执行链路

```
┌─────────────────────────────────────────────────────────────┐
│  创建 relation（visla-api / vcjs）                           │
│  ensureAigcRefAudioRelation / syncFromSpeechPlan /           │
│  changeProjectKitVoice                                       │
│    ├─ 判断是否已有可用 relation（isUsableRelation）          │
│    │    可用 → 复用，不重做 TTS                               │
│    │    不可用 → 新建 relation（latest=true）+ 旧 relation → STALE │
│    └─ enqueue sys_task（type=PROJECT_SCENE_KIT_REFAUDIO_FLOW）│
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  XXL-Job 消费 sys_task（vcjs ProjectSceneKitVoiceTaskService）│
│                                                              │
│  ① prepareRelationAudio（生成 TTS）                          │
│     voiceId = relation.voiceId ?: kit.voiceId                │
│     transcriptText = 发音纠正(scene.transcript, voice.locale) │
│     按 vendor 分流:                                          │
│       Azure   → 同步生成 → audioStatus=SUCCESS               │
│       OpenAI  → 异步 make_speech → AI callback → SUCCESS     │
│       Minimax → 异步 TTS → callback → SUCCESS                │
│     扣费（TTS credit，按字符计）                              │
│                                                              │
│  ② TTS 完成 → continueAfterRefAudioPrepared                  │
│                                                              │
│  ③ 按 refaudio_flow 分流:                                    │
│     AUDIO_ALIGN  → 批量对齐 voiceover → finalizeTerminal      │
│     CHANGE_VOICE → triggerSceneKitChangeVoice                │
│     AIGC_REDO    → triggerSceneKitMotionVideo                │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Voice 一致性判断（是否复用 / 是否重做 TTS）

这是用户描述的核心业务规则：**scene 的 voice 是否是当前 kit 的 voice，如果不是则需要重做 TTS**。

### 6.1 判断逻辑（`isUsableRelation`）

在创建新 relation 前，先查 `project_scene_kit_voice` 中该 scene + kit 的 `latest=true` 记录，判断是否可复用：

```
可复用（不重做 TTS）需全部满足：
  ① relation.projectId == project.id
  ② relation.projectKitsDetailId == kit.id          ← 同一个 kit
  ③ relation.voiceId == kit.voiceId                  ← voice 未变
  ④ relation.audioStatus == SUCCESS
  ⑤ relation.audioUrl 非空
  ⑥ 台词未变化（isRelationTranscriptCurrent）
     └─ 对当前 scene.transcript 做发音纠正后，与 relation.transcript_md5 比对
```

任一不满足 → **不可复用** → 新建 relation（`markSceneRelationsNotLatest` + insert）→ 重做 TTS。

### 6.2 台词变化判断（`isRelationTranscriptCurrent`）

- 对当前 `scene.transcript` 按 `voice.locale` 做发音纠正
- 计算 MD5，与 `relation.transcript_md5` 比对
- 不一致 → 台词已变 → 不可复用

> 这保证了：换 kit / 换 voice / 改台词，任一变化都会触发 TTS 重做。

---

## 7. 与 Motion Video 的关系（`AIGC_REDO`）

`REFAUDIO_THEN_AIGC_REDO` 是唯一会重新生成视频画面的 flow。

### 7.1 执行体：`triggerSceneKitMotionVideo`

```
1. 找 motionRel:
   - 优先找 PREPARING_REFAUDIO 状态的 pending rel
   - 找不到 → 基于 selected aigc video rel 创建新 motionRel
     （createSceneKitMotionVideoRelFromPrevious，复制 refImages）

2. 构建 motionAigc:
   - deliveryMode = ON_CAMERA_SPEAKER
   - enableAudio  = true
   - projectKitsDetailId = kit.id
   - projectSceneKitVoiceId = relation.id

3. 构建 ref_audio（来自 relation）:
   - audioUrl   = relation.audioUrl
   - audioRange = (0, relation.duration)
   - voiceOver  = relation.transcriptText

4. 构建 AiAnimateAssetBo（带 ref_audio + speech_plan）
   → 调用 aiGateWayClient.createVideos
```

### 7.2 发给 AI 的 character voice 参数

```json
{
  "motion_video": {
    "enable_audio": true,
    "speech_plan": {
      "delivery_mode": "on_camera_speaker",
      "speaker_kit_index": 0,
      "speaker_kit_name": "Charles_Vlogger",
      "speaker_kit_visual_reference": "..."
    }
  },
  "ref_audio": {
    "audio_id": "<voiceId>",
    "audio_url": "<relation.audioUrl>",
    "audio_range": { "start_time": 0, "end_time": 4464 }
  },
  "voice_over": "<transcriptText>",
  "audio_script_updated": true
}
```

### 7.3 完成后的闭环

```
triggerSceneKitMotionVideo 成功
  → motionRel.processStatus = UPLOADING
  → AI callback（视频生成完成）
  → visla_media.original_link 回填
  → asset.processStatus = COMPLETED
  → onSceneKitVoiceBatchTerminal（批次终态判定）
  → project 进入 editing / 推送通知
```

---

## 8. 触发场景

### 8.1 OpenAPI pipeline 自动触发

```
PLAN_KITS(40) → CREATE_VIDEO(50, storyboard image) → CREATE_MOTION_VIDEO(70) → voice → EXPORT(80)
```

- storyboard / motion video 完成后，`handleVoiceGenerate` → `syncBatchFromScenes`
- 遍历每个 scene 的 `motionVideo.speechPlan`：
  - `ON_CAMERA_SPEAKER` + 有 kit → `syncFromSpeechPlan`（创建 relation + `AUDIO_ALIGN` task）
  - 否则 → `syncFromProjectDefaultVoice`（项目默认 voice）

> ⚠ 见第 1 节缺陷：vcjs 首次 `createMotionVideoRel` 不绑 kit，所以首次 motion video 的 `speech_plan` 不会是 `ON_CAMERA_SPEAKER`，这条自动路径当前实际不会命中 character voice。

### 8.2 用户编辑态触发（visla-api）

| 入口 | Service 方法 | Flow |
| ---- | ------------ | ---- |
| AIGC 生成时需要 character voice | `ensureAigcRefAudioRelation` | `AIGC_REDO` |
| 换 kit voice | `changeProjectKitVoice` | `CHANGE_VOICE` / `AIGC_REDO`（按 `audioScriptUpdated`） |
| 改台词重做 | `estimateTranscriptRedoCredit` → 触发 | `AIGC_REDO` |

---

## 9. 已知问题与待优化项

### 9.1 首次 motion video 不自动做 character voice

**现象**：vcjs `triggerMotionVideos` → `createMotionVideoRel` 时 `sceneKitVoice = null`，`motionAigc.projectKitsDetailId = null`，`deliveryMode` 必为 `OFF_SCREEN_VO`/`SILENT`。

**根因**：`prepareMotionVideoRels` 调用 `createMotionVideoRel` 时硬编码传 `sceneKitVoice = null`，没有从 metadata 的 `ref_images[].kit_index` 或 `speech_plan.speaker_kit_index` 反查 kit。

**影响**：OpenAPI 首次生成的 motion video 不带 character voice，需要后续二次触发（`REFAUDIO_THEN_AIGC_REDO`）才能补上。

### 9.2 `REFAUDIO_THEN_AIGC_REDO` 依赖已生成的 motion video

**现象**（线上日志证实）：`triggerSceneKitMotionVideo` 用 motionRel 的 `asset.assetLink` 作为输入图，但首次场景下 motion video 尚未生成，`assetLink` 为空 → scene 被 skip → relation FAILED。

```
WARN AigcMotionVideoService - skip motion video scene because visla media original link is blank
```

**根因**：`triggerSceneKitMotionVideo` 依赖「已有的 motion video」作为输入，但首次触发时只有 storyboard image，没有 motion video。

### 9.3 首次触发 refImages 丢失

**现象**：`createMotionVideoRel` 没有 copy storyboard 阶段的 refImages 到 motionAigc，导致 `mergeMotionVideo` 查 refImages 返回空 → mode 退化为 `PROMPT_TO_VIDEO`，keyframes/ingredientsRefImages 丢失。

**根因**：`copyMotionVideoRefImages` 只在 `createSceneKitMotionVideoRelFromPrevious` 调用，`createMotionVideoRel` 没调。

---

## 10. 涉及代码索引

### visla-api

| 文件 | 职责 |
| ---- | ---- |
| `service/ProjectSceneKitVoiceService.java` | relation 创建、`isUsableRelation`、`ensureAigcRefAudioRelation`、`changeProjectKitVoice` |
| `controller/v3/ProjectEditingV3Controller.java` | `/scene/kit-voice/change-voice`、estimate-credit 端点 |

### vcjs

| 文件 | 职责 |
| ---- | ---- |
| `service/ProjectSceneKitVoiceService.java` | `syncFromSpeechPlan`、`syncBatchFromScenes`（OpenAPI 自动同步） |
| `service/xxl/ProjectSceneKitVoiceTaskService.java` | XXL-Job 消费 sys_task、`prepareRelationAudio`（TTS）、flow 分流 |
| `service/AigcMotionVideoService.java` | `triggerSceneKitMotionVideo`（AIGC_REDO）、`triggerSceneKitChangeVoice`（CHANGE_VOICE） |
| `service/pipeline/ProjectPipelineCompletionCoordinator.java` | 批次终态判定、pipeline 协调 |
| `enums/ProjectSceneKitRefAudioFlowEnum.java` | 三种 Flow 枚举 |
| `enums/ProjectSceneKitAudioStatusEnum.java` | audioStatus 状态机 |
