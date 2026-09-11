# Kit Voice（角色语音）业务流程

> kit = 视频里的数字人角色（character）：一个形象绑定一种音色，角色在画面里开口说话。
> 本文档讲业务规则：什么操作、走什么处理、scene 怎么变、什么时候触发 AIGC、出错怎么办。
> PJSON 实现适配见 [pjson-kit-voice.md](../../vme-pjson/pjson-kit-voice.md)。

## 1. 业务是什么

普通视频的配音和画面是**互相独立**的：换配音随便换，画面不动。

角色视频不是——**语音和口型必须匹配**（lip-sync）。用户改了角色的音色或台词，画面里那张嘴的口型就对不上了。所以角色的语音有一套专属链路：**改语音的同时必须同步处理画面**。

核心约束由此而来：

| 变更内容 | 画面是否要动 | 原因 |
| ---- | ---- | ---- |
| 只换音色，台词没变 | 要动（但可复用画面重打口型） | 不同人的嘴型不同 |
| 台词变了 | 必须重生成视频 | 口型序列 + 每句时长全变 |

## 2. 用户操作与系统动作

| 用户操作 | 系统判断 | 走哪条路 | 最终效果 |
| ---- | ---- | ---- | ---- |
| 给整个角色换音色 | 台词没变 | 换声（画面复用） | 同一画面，新音色新口型 |
| 给整个角色换音色 | 台词变了 | 重生成视频 | 新画面新语音 |
| 编辑台词后点"重新合成配音" | 覆盖到角色场景时同上分流 | 同上 | 同上 |
| 对单个场景点"重新生成角色视频" | 不判断，固定 | 重生成视频 | 新画面，语音用该角色当前音色重出 |

**判断"台词是否变更"的方式**：入队时把台词快照（md5）存进 relation，分流时和当前台词比对。

## 3. 处理流程与关键节点

所有操作都是**异步**的：入口只做"入队"，真正干活在 vcjs 调度侧。用户操作后界面立即返回，后台走完下面链路再通知刷新。

```
① 入队
   每个 scene 建一行 relation（角色+音色+台词快照+批次号），投一个 SysTask
   一次操作影响 N 个 scene → 同一批次
        ↓  XXL-Job 轮询
② TTS 生成参考音频（所有路径共同的前置）
   为什么先出音频：AIGC 视频以"参考音频"为输入对口型，没音频就没法生成/换声
   文案做发音纠正（workspace 级生僻词表）后调 TTS
        ↓  TTS 异步回调
③ 分流（核心决策点）
   ├─ 台词没变 → 换声：拿现有视频 + 新音频重打口型（画面复用，最快最省）
   ├─ 台词变了 → AIGC 重新生成视频（以新音频为 ref_audio）
   └─ 无 AIGC 视频（SYSTEM 类型）→ 只做音频对齐落位
        ↓
④-A 音频对齐路径
   等整批 TTS 完成 → 外部 aligner 把音频和台词逐句对齐 → 配音写入 scene（voiceOvers）
        ↓
④-B 换声 / ④-C 重生成路径
   调 AIGC → 异步生成 → 回调
        ↓
⑤ AIGC 结果落位（角色场景特有）
   生成的视频作为 broll 应用到 scene：替换旧的、算时长、必要时场景时长联动
        ↓
⑥ 批次收口
   批次内全部 relation 终态（成功或失败）→ 更新项目状态 → 通知用户
```

## 4. scene 什么时候变、变什么

| 路径 | scene 变更内容 | 变更时机 |
| ---- | ---- | ---- |
| 音频对齐 | voiceOvers（配音音轨 + 对齐后的台词时间轴）；联动 avatar 时长 | aligner 回调时 |
| 换声 | broll 换成新视频（时长基本不变） | AIGC 回调时 |
| 重生成 | broll 换成新视频 + 场景时长可能联动（新台词时长不同） | AIGC 回调时 |

**场景时长是否跟着视频走**（重生成路径）：

| 条件 | 行为 |
| ---- | ---- |
| 场景里已有配音音轨 | 场景时长不变，视频按 `min(视频时长, 场景时长)` 截断；有配音时轨时 broll 时长 = 配音时长（**音频驱动**，因为语音不可变速） |
| 场景无配音音轨（首个/唯一 footage） | 场景时长 = 视频时长（视频拉着场景走） |

## 5. 什么时候触发 AIGC 生成

两个入口、一个判断：

1. **用户换音色/重合成，且台词变了** → 重生成整段视频
2. **用户对场景点重新生成角色视频** → 无条件重生成（`BATCH_SOURCE_AIGC_VIDEO`）
3. 台词没变只换声 → **不**走完整 AIGC 生成，只做换声（lip-sync 重打，成本远低于重新生成）

## 6. 端到端时序（含回调队列与代码位置）

以换音色、台词变更（AIGC_REDO flow，最全链路）为例。ⓐⓑⓒ 为三个外部回调。类名前缀省略：A=visla-api `ProjectSceneKitVoiceService`，K=vcjs `service/xxl/ProjectSceneKitVoiceTaskService`，M=vcjs `AigcMotionVideoService`。

| # | 步骤 | 谁→谁 | 载体/队列 | 代码位置 |
| ---- | ---- | ---- | ---- | ---- |
| 1 | 用户请求（换音色/重合成/重生成视频） | 前端→visla-api | HTTP | A.`changeProjectKitVoice` / `generateSyntheticVoice`（kit 分流 `handleSyntheticVoiceForKitScenes`）/ `generateAiVideo` |
| 2 | 建 relation + 投 SysTask | visla-api | DB | A.`createRefaudioRelation` + `enqueueTtsTask`（afterCommit 后任务可见） |
| 3 | 轮询 SysTask | XXL-Job | DB | `xxl/jobs/ProjectSceneKitVoiceJob`（`ProjectSceneKitRefAudioFlowJob`，count=20） |
| 4 | 组 TTS 请求（发音纠正） | vcjs→AI | `aiAllInOneMakeVoice` 异步 | K.`processRefAudioFlow` → `prepareRelationAudio`（extra 带 source/relation_id/batch_id）；task 置 PROCESSING |
| 5 | ⓐ TTS 回调：音频 URL 写 relation | AI→vcjs | SQS `ai-nlp-tts-cb` | `voice/AiNlpTtsConsumer` → `AITtsVoiceService`（按 extra.relation_id 路由）→ K.`handleTtsCallback` → `continueAfterRefAudioPrepared` |
| 6 | 分流决策 | vcjs | — | K.`continueAfterRefAudioPrepared` / `continueAfterAudioAligned`（按 flow） |
| 7a | 【AUDIO_ALIGN】等整批 TTS 终态→发外部 aligner | vcjs→aligner | SQS 请求队列（`AwsSqsService.sendSqsToAudioTimeStamp`） | K.`tryTriggerBatchAudioAlignAfterCommit` → `tryTriggerBatchAudioAlign`（source=project_scene_kit_voice） |
| 7b | 【CHANGE_VOICE】换声请求 | vcjs→AI | 异步 | M.`triggerSceneKitChangeVoice`（画面复用） |
| 7c | 【AIGC_REDO】视频重生成请求（ref_audio=①音频） | vcjs→AI | `createVideos` 异步 | M.`triggerSceneKitMotionVideo`（afterCommit） |
| 8 | ⓑ 对齐回调（仅 AUDIO_ALIGN）：配音写入 scene | aligner→vcjs | SQS `audio-time-stamp-rsp` | `voice/AudioWordTimestampConsumer` → `AudioWordTimestampService.processHandler` → `processAlignerBos(source=project_scene_kit_voice)` → finally：K.`handleSceneKitAudioAlignCompleted`（批收口） |
| 9 | ⓒ AIGC 回调（7b/7c 的结果） | AI→vcjs | SQS `ai-aigc-video-cb`（change-voice 为 `ai-voice-change-cb`） | `AigcVideoConsumer` → `AigcVideoCompletionHandler` → `addOrReplaceFootageAfterAIGC`（footage 落位）；kit 路由：K.`handleVideoCallback`（extra.project_scene_kit_voice_id） / `AiAdvancedAvatarChangeVoiceCbConsumer` → K.`handleChangeVoiceCallback` |
| 10 | 批次收口：状态归位 + 通知 | vcjs | WS 推送 | `ProjectPipelineCompletionCoordinator.onSceneKitVoiceBatchTerminal` → `notifyBatchResult` |

> 队列总表：TTS 回调 `ai-nlp-tts-cb`；视频 AIGC 回调 `ai-aigc-video-cb.fifo`；换声回调 `ai-voice-change-cb`；对齐回调 `audio-time-stamp-rsp`（录音上传的对齐走另一个 `audio-aligner-rsp`，与 kit 无关）。
> 排查卡点时按消息里的 extra 字段定位：`relation_id`（TTS）、`project_scene_kit_voice_id`（AIGC）、`batch_id`（全链路贯穿，同时是 traceId）。

## 7. 异常处理

| 故障点 | 系统行为 | 用户感知 |
| ---- | ---- | ---- |
| TTS 失败 | relation 置 FAILED；task 自动重试（最多 3 次），仍失败则批次失败收口 | 收到失败通知，角色维持原语音 |
| 对齐匹配度低 | 该场景标记未全匹配（日志），不中断批次 | 配音可能整体偏移；极端时部分场景无配音 |
| AIGC 生成失败 | rel 置 FAILED，批次失败收口 | 收到失败通知，画面维持原样 |
| 用户中途重复操作 | 新操作建新 relation，旧 relation 标记过期；调度/回调两侧都跳过旧行（stale 保护） | 只有最新一次操作生效，不会交错 |
| 批次部分失败 | 已成功的场景保留结果，批次按失败收口并通知 | 成功的场景已生效，失败的可单独重试 |

**失败不回滚已成功部分**：语音/画面一旦生成本身是正确产物，失败的是增量，重试只补失败项。

## 8. 调度载体（为什么是 SysTask）

- 每个场景的 TTS→后续动作 是一个长异步链（TTS 秒级、AIGC 分钟级），不能挂在 HTTP 请求里
- relation 表承担"业务状态机"（音频状态、批次、最新行），SysTask 只承担"待办触发"；两者分离，任务失败重试不动业务状态
- XXL-Job 批量轮询，天然削峰

## 9. 技术索引（详细实现）

| 关注点 | 位置 |
| ---- | ---- |
| 入口/入队/credit 估算 | visla-api `ProjectSceneKitVoiceService` |
| TTS 执行/回调路由/批次协调 | vcjs `service/xxl/ProjectSceneKitVoiceTaskService` |
| AIGC 触发（重生成/换声） | vcjs `AigcMotionVideoService` |
| 音频对齐落位 | vcjs `AudioWordTimestampService`（source=project_scene_kit_voice 分支） |
| AIGC 结果落位 scene | `addOrReplaceFootageAfterAIGC`（详见 [footage/](../footage/) 与 wiki「add-aigc-footage」） |
| PJSON 适配 | [pjson-kit-voice.md](../../vme-pjson/pjson-kit-voice.md) |
