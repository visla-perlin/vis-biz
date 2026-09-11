# TTS（合成配音）业务流程

> 范围：API 端用户主动发起的合成配音（`POST /v3/{projectUuid}/voice/synthetic`）  
> 视角：数据载体 + 链路 + 状态机  
> 背景：project generation（all-in-one / agent）内部也会自动触发 TTS，逻辑复用本文链路，但本文不展开 generation 主体

---

## 1. TTS 在 project generation 中的位置（背景）

```
project generation（all-in-one / agent）
  idea/script → 分镜（scenes + transcript）→ recommend footage
       ↓
    ★ TTS 合成配音 ★   ← 本文焦点
       ↓
    AIGC footage 生成 / 替换
       ↓
    export 导出
```

TTS 是 generation 流水线的一环：拿到 scene 的 transcript 文案，按选定 voice 生成旁白音频，回填到 scene 的 voiceOver timeline + transcript 时间戳 + scene 时长联动。

> 编辑态用户也可独立调用 `POST /voice/synthetic` 重新生成配音（regenerate），链路与 generation 内部触发一致。

---

## 2. 涉及的数据载体（NORMAL / DB）

| 表 / 字段 | 作用 | TTS 链路职责 |
| ----------- | ------ | ------------- |
| `project.ai_generate_voice_status` | 配音状态机（NORMAL/PROCESSING） | 发起=PROCESSING，完成=NORMAL |
| `project.synthetic_all` | 是否整项目配音 | forAll 时置 true |
| `project_scene.transcript` | 场景文案 + 时间戳 | TTS 后写回 items/sentences 时间戳 |
| `project_scene.duration` / `pause_time` | 场景时长 | TTS 后按音频时长联动 |
| `project_timeline`（category=SYNTHETIC_VOICE_OVER） | 旁白音频 timeline | TTS 后新建/替换 |
| `project_asset`（SCENES_SYNTHETIC_VOICE_OVER） | 音频文件实体 | TTS 生成后新建 |
| `visla_media` + `media_synthetic_voice` | media + voiceId 绑定 | 随 asset 写入 |
| `agent_project_attributes.voice_id` | 项目级 global voice | forAll 时更新 |
| `synthetic_voice_metadata` | voice 字典（locale/vendor/style） | 读取，不写 |

---

## 3. API 发起流程

### 3.1 入口

```
POST /v3/{projectUuid}/voice/synthetic
  → SyntheticVoiceV3Controller.generateSyntheticVoice
    （ACL PROJECT_CAN_EDIT + 操作锁 PROJECT_OP_GROUP）
  → SyntheticVoiceMetadataService.generateSyntheticVoice(project, workspace, reqBody)
```

### 3.2 准备阶段（generateSyntheticVoice 主逻辑）

```
1. 锁校验 + 状态校验（ai_generate_voice_status != PROCESSING）
2. 解析 sceneIds（指定场景）或 forAll（全项目 → queryProjectSceneListSorted）
3. 选 voice：
     voiceUuid 数字 → voiceSelectionService.selectVoice(voiceId)
     否则            → selectVoice(project)（按语言等自动选）
4. skip record voice（replaceRecordVoice=false 时）：
     扫 project_timeline（RECORD_VOICE_OVER / SELF_RECORDING footage）→ 移除这些 scene
5. kit voice 分流（handleSyntheticVoiceForKitScenes）→ 单独任务，不在本文主线
6. 按 voiceVendor 分流：
     OpenAI / Minimax → createAudioByAiTts（异步 AI 网关）
     Azure           → 组 SyntheticVoiceInputBo → afterCommit 发 SQS
7. 更新 project：
     ai_generate_voice_status = PROCESSING
     forAll → synthetic_all = true
8. forAll → 写 agent_project_attributes.voice_id（global voice）
9. 返回 projectDetail
```

### 3.3 分流 A：Azure（本地 TTS）

```
SyntheticVoiceInputBo{
  voiceId, sceneIds, replaceRecordVoice, projectId, userId,
  volume, trackingId, voiceDecorators, pipelineNewPath
}
→ preprocessService.sendSyntheticVoiceAfterTransactionCommit
   → afterCommit → sqsService.sendSqsToSyntheticVoice
      → SQS queue: synthetic-voice-req
```

> 注意：消息体**不带 scene 文案**，只带 sceneIds。Job 侧从 DB 读 transcript。

### 3.4 分流 B：OpenAI / Minimax（AI TTS）

```
createAudioByAiTts：
  1. 从 DB scene 读 transcript 组装 AITtsRequestBo
     （含 pronounce 纠正、segment 分组、video properties）
  2. 存请求 JSON 到 S3
  3. 调 AI 网关 aiAllInOneMakeVoice（异步）
     成功返回 → 等 AI 回调
     失败     → 抛 PROJECT_VOICE_CREATE_FAILED
```

---

## 4. Job 执行流程（vcjs）

### 4.1 Azure 路径

```
SQS synthetic-voice-req
  → SyntheticVoiceConsumer.handleMessage
  → AzureVoiceGenerateServiceImpl.regenerateVoiceByAzure
       1. 校验 credit / sceneIds 非空
       2. 从 DB 读 scene transcript（getByIdsInBatch）
       3. tryFixLanguage（voice locale 对齐 project language）
       4. HD voice？
            是 → createHDVoiceOverAsync（异步 aligner 队列，见 §5）
            否 → createNewSyntheticVoiceResourcesSync（本地同步）
       5. 释放配音锁
```

`createNewSyntheticVoiceResourcesSync`（核心）：

```
1. doTTS(sceneScripts) → 调 Azure 生成音频 + 拿 word-level 时间戳
2. insertNewProjectAssetOfAudios
     → project_asset（SCENES_SYNTHETIC_VOICE_OVER）+ visla_media + media_synthetic_voice
3. insertNewTimelineAndAudioVolumeStyles
     → project_timeline（SYNTHETIC_VOICE_OVER）新建/替换
       （先 deleteBySceneIdAndCategory 清旧 SYNTHETIC/RECORD，再 insert）
4. updateProjectTranscriptWithWordsTimeStamp
     → project_scene.transcript 写回 word/sentence 时间戳
     → resetSceneTime：按音频时长联动 scene.duration / pause_time
       （forceAlignWithVoice=true → 音频时长+pause；false → max(原时长, 音频时长)）
5. resetBgmTime + syncAiKenburnsCropOffset
6. updateProjectStatus + sendMessage（通知用户）
```

### 4.2 AI TTS 回调路径

```
AI 网关回调 → SQS ai-tts-cb
  → AiNlpTtsConsumer.handleMessage
  → AITtsVoiceService.processTtsCb
       1. 从 S3 读回调结果（voiceOvers + sceneIds + ssml markup）
       2. 从 DB 读 scene（getByIdsOrderByOffset）
       3. 组装 project_asset（音频）+ media
       4. （后续与 Azure 路径类似：写 timeline + transcript + scene 时长）
       5. updateProjectStatus + sendMessage
```

---

## 5. Align（对齐）流程

### 5.1 何时触发

| 场景 | 是否走 aligner | 时间戳来源 |
| ------ | :---: | ------ |
| Azure 非 HD（同步） | 否（newPipeline 内部算） | Azure word boundary（同步返回） |
| Azure HD（异步） | ✅ force-aligner 队列 | aligner 回调 |
| AI TTS 回调 | 视回调数据 | 回调结果或 aligner |

### 5.2 HD voice 异步 align 链路

```
createHDVoiceOverAsync
  → 调 AI 生成音频 + 提交 force-aligner
  → SQS audio-aligner-rsp（回调）
  → VoiceoverAlignerService.handleMessage
  → VoiceOverAlignerHandlerService.processHandler
       → insertSceneVoice / parseSceneData：
            ✅ 写 project_timeline（SYNTHETIC_VOICE_OVER / RECORD_VOICE_OVER）
            ✅ 改 project_scene.transcript（items/sentences 时间戳）
            ✅ 改 project_scene.duration（按 audioInfo endTime）
            ✅ pronounce 纠正还原（仅 old pipeline 的 fillSentenceBo 必需；new pipeline 冗余，见 5.3/5.4）
            ✅ highlight / avatar duration 联动
```

### 5.3 newPipeline 对齐（NewPipelineAlignService）

> 代码：`vcjs` `us.visla.cron.transcript.pipeline.align.NewPipelineAlignService` → `AlignService`

#### 5.3.1 解决的问题

TTS 合成后有两份数据，必须合二为一：

| 数据 | 有什么 | 缺什么 |
|------|--------|--------|
| 原文（`scene.transcript.originalVoiceover`） | 正确的字幕文字 | 时间戳 |
| 音频对齐结果（Azure word boundary / aligner 回调） | 精确的词级时间戳 | 可能是发音拼写、有零时长、覆盖率不齐 |

**aligner 返回的 item 不能直接用作字幕**（缺陷：① content 是发音拼写；② 会吐零时长坏数据；③ 音频识别漏词/多词导致 token 数对不上；④ 句子边界来自音频而非原文）。newPipeline 的做法是**以原文为基准从零重建 item**，而不是修补 aligner item。

#### 5.3.2 核心契约：content 与时间戳来源分离

```
输入① transcriptJson  → codec.read → originalVoiceover（原文）
输入② sceneBo/words    → mapAlignerWords → audioWords（音频词，含发音拼写，含时间戳）
   │
   ├─ IcuSegmenter.segment(原文) → sentenceSpans（分句）+ normalizedVoiceover
   ├─ LcsAligner.align(原文tokens, 音频子词) → List<Match>（保序匹配对）
   └─ buildItemsGlobal + fillGaps → List<TranscriptItemVo>

输出 item：content 恒取原文 token，时间戳取音频词（匹配失败的 token 由 fillGaps 插值兜底）
```

关键设计：**原文决定「显示什么字」，音频决定「每个字何时出现」**。发音拼写只参与 LCS 时间戳锚定，永远不进输出 content。这正是 newPipeline 下 `doPronounceCorrect` 还原变得冗余的根本原因（见 5.4）。

#### 5.3.3 两个入口

| 入口 | 触发路径 | 入参 |
| ------ | --------- | ------ |
| `processAlignerBos` | Azure HD 异步 / OpenAI / Minimax 回调（aligner 队列返回后） | transcriptJson, localeTag, sceneBo |
| `processSyncTtsResult` | Azure 非 HD 同步（合成完直接 word boundary） | sceneId, transcriptJson, localeTag, words |

#### 5.3.4 纯计算约束（2026-08 重构）

该服务**零 DB 依赖**：不查 `project_scene`、不查 `project_ai_property`。

- transcript 由调用方传入（调用方调用前已持有 scene）
- locale 由调用方传入（三条上游链 `AudioWordTimestampService` / `VoiceOverAlignerHandlerService.processHandler` / Azure metadata 都已解析完成）
- PJSON 改造时本类完全不用动，只要调用方传入正确的 transcriptJson

### 5.4 发音纠正文本的生命周期（跨项目契约）

> **核心结论：发音纠正文本（原始词→发音拼写）在任何路径下都不落库。`scene.transcript` 始终是原始人类可读文案。**

#### 5.4.1 全路径流向

| 路径 | 发音替换发生地 | 替换后用途 | 是否落库 |
| ------ | -------------- | ---------- | :---: |
| OpenAI / Minimax（visla-api 发起） | `TtsScriptAssembler.buildCorrectedScriptMap` | 填入 `AITtsRequestBo` 请求体发出 | ✗ |
| Azure HD 异步（vcjs） | `replaceSentencesInPlace` 改内存 sceneList | 送 Azure HD 合成 | ✗ |
| Azure 非 HD 同步（vcjs） | `replaceSentencesInPlace` 改内存 sceneList | 送 Azure 合成 | ✗ |
| Aligner 回调（vcjs） | `doPronounceCorrect` 还原 aligner 返回的 sceneBo | 还原对齐结果 | ✗ |

`replaceSentencesInPlace`（vcjs）虽原地改内存对象的 transcript 字段，但**无任何 update dao 调用**；visla-api 的 `buildCorrectedScriptMap` 更是返回新 Map 不碰 scene。SQS 消息（`SyntheticVoiceInputBo`）只传 sceneIds，**不传发音文本**。

#### 5.4.2 `doPronounceCorrect` 作用域

| Pipeline | 是否需要 `doPronounceCorrect` | 原因 |
| ---------- | :---: | ------ |
| old pipeline | ✅ 必需 | `fillSentenceBo` 直接吃 aligner item 的 content，不还原会泄漏发音拼写进字幕 |
| new pipeline | ✗ 冗余 | content 取自 originalVoiceover，发音拼写不参与输出；还原后的 sceneBo.items.content 无消费者 |

> 注：new pipeline 下 `doPronounceCorrect` 仍会执行（代码未按 pipeline 分支跳过），属冗余计算而非功能 bug。若 old pipeline 下线，可连同 `fillSentenceBo` 一起删除。

---

## 6. 状态机

```
project.ai_generate_voice_status:
  NORMAL →（发起）→ PROCESSING →（job/回调完成）→ NORMAL
                                 →（失败）→ NORMAL + 通知失败
```

锁：`PROJECT_OP_GROUP`（API 入口）+ `synthetic_voice lock`（130s，generateSyntheticVoice 内部 checkSyntheticLock）。

---

## 7. PJSON 下的影响（断裂点）

PJSON 项目的 scene / timeline / transcript 在 **PES**，但当前 TTS 全链路读写 DB 的这些表：

| 链路环节 | 代码位置 | NORMAL 读写 | PJSON 后果 |
| --------- | --------- | ------------ | ----------- |
| API skip record | `getSkipSceneIds` | 扫 DB timeline（RECORD_VOICE_OVER） | PJSON 无 DB voice timeline → 跳过逻辑失效 |
| API forAll | `queryProjectSceneListSorted` | DB scene 列表 | PJSON 无 DB scene → 空 |
| Azure Job 读文案 | `getByIdsInBatch` | DB scene transcript | PJSON 无 transcript → 生成空音频 |
| Azure Job 写结果 | `insertNewTimelineAndAudioVolumeStyles` | DB timeline | 写错存储，PES 无 voiceOvers |
| Azure Job 写 transcript | `updateProjectTranscriptWithWordsTimeStamp` | DB scene.transcript + duration | PES 无更新 |
| AI TTS 回调 | `AITtsVoiceService` | 同上 | 同上 |
| Align 写回 | `parseSceneData` | DB timeline + scene | 同上 |
| 辅助 voice 读取 | `getSceneVoiceMetadataMap`（AIGC 用） | DB SYNTHETIC_VOICE_OVER timeline | PJSON 只剩 global voice |

**结论**：TTS 链路当前**完全无 PJSON 感知**（无 `@PjsonProjectOp` / PES 分支）。PJSON 项目下用户主动 regenerate 配音会直接断裂。

> 注：generation 阶段项目仍是 NORMAL（storageType≠PJSON），首轮 TTS 不受影响。本节仅指**已迁移 PJSON 的编辑态**下的 regenerate。

---

## 8. 关键代码索引

### visla-api

| 角色 | 位置 |
| ------ | ------ |
| API 入口 | `SyntheticVoiceV3Controller.generateSyntheticVoice` |
| 主逻辑 | `SyntheticVoiceMetadataService.generateSyntheticVoice` |
| AI TTS 组装 | `SyntheticVoiceMetadataService.createAudioByAiTts` |
| Azure 发 SQS | `PreprocessService.sendSyntheticVoiceAfterTransactionCommit` → `AwsSqsService.sendSqsToSyntheticVoice` |

### vcjs

| 角色 | 位置 |
| ------ | ------ |
| Azure 消费 | `SyntheticVoiceConsumer` → `AzureVoiceGenerateServiceImpl.regenerateVoiceByAzure` |
| Azure 本地生成 | `AzureVoiceGenerateServiceImpl.createNewSyntheticVoiceResourcesSync` |
| Azure HD 异步 | `AzureVoiceGenerateServiceImpl.createHDVoiceOverAsync` |
| AI TTS 回调 | `AiNlpTtsConsumer` → `AITtsVoiceService.processTtsCb` |
| Align 回调 | `VoiceoverAlignerService` → `VoiceOverAlignerHandlerService.parseSceneData` |
| newPipeline 对齐 | `NewPipelineAlignService.processSyncTtsResult` |
