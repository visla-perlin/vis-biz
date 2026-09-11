# TTS PJSON 接入设计

> 状态：设计（未实现）  
> 前置 foundation（已完成，NORMAL-only）：`vcjs` 分支 `refactor/tts-scene-provider`  
> 业务流程：`vis-biz/voice/tts-flow.md`  
> 原则：**不拆 `parseSceneData`**；写库仍在原方法；PJSON 只在统一收口点分叉。
>
> **文档归属**：本文档记录 **normal 视角下 TTS 业务规则**（不是 PJSON 协议）。
> PJSON 路径下的 TTS 协议实现见 `vme-pjson/pjson-tts-design.md`。

---

## 1. Foundation 复用（不重做）

| 模块 | 位置 | PJSON 时怎么用 |
| ------ | ------ | ---------------- |
| `TtsSceneProvider` | `service/tts/` | 方法内加 `storageType`：NORMAL=DB，PJSON=PES `get_scene` |
| `TtsScriptAssembler` | `service/tts/` | **不改**（纯文案，storage 无关） |
| `VoiceOverResult` | `service/tts/` | 作为 PES notify 的原料 |
| `TtsSceneWriter` | `service/tts/` | `applyVoiceOver`：NORMAL 继续 no-op/日志；PJSON → notify |
| `captureFromAzureReload` / `captureFromAlign` | `TtsSceneWriter` | **写完 DB 之后**已组装 Result；PJSON 改为「算完/写完后 notify，并跳过结构写库」 |

**刻意不做**：拆 `parseSceneData` 的 calc/persist。  
原因：for-scene 串行写会改后续 `timeOffset`，批量先算后写会不等价；旁路采集 + Writer 分叉已够 PJSON 接入。

---

## 2. 两条 TTS 链路在 PJSON 下怎么走

### 2.1 总览

```
API POST /voice/synthetic
  ├─ 锁 / PROCESSING / credit（DB，不变）
  ├─ scene 列表：TtsSceneProvider（PJSON → PES）
  ├─ skip record：PES voiceOvers（无 DB timeline）
  ├─ OpenAI/Minimax → createAudioByAiTts（文案 Assembler，scene 来自 Provider）
  └─ Azure → SQS synthetic-voice-req（消息体不变，仍只带 sceneIds）

Job Azure 同步
  ├─ TtsSceneProvider 读 transcript（PJSON → PES）
  ├─ doTTS + 建 project_asset（DB，不变）
  ├─ NORMAL：写 timeline/scene/bgm…（现逻辑）
  └─ PJSON：跳过结构写库；captureFromAzureReload → applyVoiceOver → PES

Job OpenAI/Minimax
  ├─ processTtsCb：Provider 读 scene；建 asset（DB）；发音纠正 Assembler
  ├─ 发 aligner SQS（不变）
  └─ Align 回调 insertSceneVoice/parseSceneData
       ├─ NORMAL：现逻辑写 DB + 联动
       └─ PJSON：跳过结构写库与 triggerSceneDurationChange；
            captureFromAlign → applyVoiceOver → PES
```

### 2.2 谁写什么（边界）

| 数据 | NORMAL | PJSON |
| ------ | -------- | ------- |
| `project_asset` / `media_synthetic_voice` | DB | **DB**（不变） |
| `project.ai_generate_voice_status` / 锁 / credit | DB | **DB** |
| voiceOver timeline | DB | **PES** |
| scene.duration / pause / transcript | DB | **PES** |
| highlight / timeOffset 级联 / layer/blur/text / BGM / avatar / kenburns | DB | **PES 全权** |
| 发音纠正字典 | DB `workspace_pron_corr` | **DB** |

---

## 3. PES 契约：`generate-voice:apply-voice-over`

### 3.1 action

```text
action_id: generate-voice:apply-voice-over
```

（与 AIGC 的 `generate-video:apply-footage` 对称。）

### 3.2 action_content（草案）

```json
{
  "scenes": [
    {
      "sceneUuid": "1504079686018281995",
      "asset": {
        "assetUuid": "1529886831565451264",
        "mediaType": "AUDIO",
        "assetType": "SCENES_SYNTHETIC_VOICE_OVER",
        "duration": 5747
      },
      "voiceCategory": "SYNTHETIC_VOICE_OVER",
      "audioRange": { "startTime": 0, "endTime": 5747 },
      "voiceVolume": 1.0,
      "transcript": {
        "items": [/* word-level，Java 算好 */],
        "sentences": [/* Java 算好 */],
        "originalVoiceover": "...",
        "originalKeywords": ["..."]
      },
      "sceneDuration": 6047,
      "pauseTimeHint": 300,
      "forceAlignWithVoice": true,
      "highlightData": null
    }
  ],
  "workspaceSettings": { "slowdownToFit": false },
  "creationTemplate": "VIDEO_WITH_VOICE",
  "syntheticAll": false
}
```

### 3.3 字段分工

**Java 负责（算完再传）**

- assetUuid / audioRange / transcript items&sentences  
- sceneDuration / pauseTimeHint / forceAlignWithVoice（与现网算术一致）  
- voiceVolume、creationTemplate、syntheticAll、slowdownToFit  
- highlightData：有则传；没有可不传（Align 现网写 DB 后未进 Result 的可二期补）

**PES 负责（自有结构）**

- 替换 scene.voiceOvers  
- 写 scene.transcript / duration / pause  
- timeOffset 级联  
- layer / blur / text / BGM / avatar / kenburns 联动  
- assets 索引挂 assetUuid  

### 3.4 与 capture 的映射

| VoiceOverResult | content 字段 |
| ----------------- | -------------- |
| sceneId | sceneUuid（字符串化） |
| assetId / assetDuration | asset.assetUuid / asset.duration |
| audioStartTime / audioEndTime | audioRange |
| transcriptItems/Sentences/original* | transcript.* |
| sceneDuration / pauseTime / forceAlignWithVoice | 同名 |
| voiceVolume / creationTemplate / syntheticAll | 同名或顶层 |
| highlightData | highlightData（可空） |

---

## 4. 实现切入点（不拆 parseSceneData）

### 4.1 读：`TtsSceneProvider`（v2-pjson）

```text
getByIds / getByIdsOrderByOffset / listAllByProjectId:
  if PJSON → load pseudo ProjectScene from PES (id, transcript, duration, pauseTime,
              timeOffset, sceneType, segmentId, projectId)
  else → 现 DB
```

伪 scene 只需撑住 TTS 计算；不要求 DB 全字段。

### 4.2 写：`TtsSceneWriter.applyVoiceOver`（v2-pjson）

```text
if NORMAL:
  现逻辑：capture 后仅日志（DB 已由上游写完）
if PJSON:
  上游必须「只算 + 建 asset」，不写 timeline/scene 结构
  applyVoiceOver → PjsonProjectEditService.notifyApplyVoiceOver → afterCommit notify
```

### 4.3 上游如何「PJSON 跳过结构写」

**不拆 parseSceneData 的前提下**，推荐：

| 路径 | 做法 |
|------|------|
| Azure `createNewSyntheticVoiceResourcesSync` | 方法开头或写 timeline 前：`if (PJSON) { 建 asset + 组 Result + applyVoiceOver; return; }` 跳过 insertTimeline/updateTranscript/resetBgm/syncKenburns 等结构写 |
| Align `insertSceneVoice` / `parseSceneData` | 在 `insertSceneVoice` 循环层：`if (PJSON) { 对每个 scene 只做计算所需读取 + 组装 Result; 不调会写库的段落 }` **或** `parseSceneData` 顶部 early-return 前用 flag：PJSON 只跑计算段并收集 Result，不执行 delete/insert/update/trigger |

更稳妥、改动面更小的是 **Azure 方法级 early PJSON 分支** + **Align 在 `insertSceneVoice` 入口 PJSON 分支（内部复制最小计算路径或抽 private calc 方法但不改循环语义）**。

若坚持 **零 early-return**：只能 PJSON 也写 DB（错误）。故 PJSON 必须在结构写之前短路。

**推荐落地顺序**：

1. `TtsSceneProvider` PJSON 读  
2. `TtsSceneWriter` PJSON notify + content BO / 常量  
3. Azure 同步路径 PJSON 短路 + capture → notify  
4. Align 路径 PJSON 短路 + capture → notify  
5. API `generateSyntheticVoice` `@PjsonProjectOp` handler  

### 4.4 API（visla-api）

- `@PjsonProjectOp("generateSyntheticVoice")` SHORT_CIRCUIT  
- handler：锁、PROCESSING、Provider 读 scene、Assembler 文案、发 SQS/AI、不写 DB scene  
- 与 job 消息契约不变（仍 sceneIds；job 自己 Provider 读 PES）

---

## 5. 与 foundation 的衔接

```text
refactor/tts-scene-provider（已完成）
  Provider + Assembler + Result + Writer.capture* + Azure/Align 旁路采集
        │ merge 到 v2-pjson
        ▼
v2-pjson 实现本设计
  Provider 读 PES
  Writer.applyVoiceOver → notify
  Azure/Align 结构写短路
  API handler
```

**merge 后不必重做 capture 组装**；只需：

- PJSON 时 **不要先写结构 DB**，改为用同一套 Result 字段直接 notify；或  
- 仍先在内存算齐字段再 notify（Align 需从 parseSceneData 取出计算量，或 PJSON 分支内联最小计算）。

因 **不拆 parseSceneData**，Align 的 PJSON 实现会略「脏」（入口 early 分支），但可接受、且风险低于整方法拆分。

---

## 6. 风险与决策

| 风险 | 缓解 |
| ------ | ------ |
| Align PJSON 要跳过 trigger 级联 | PES 做 timeOffset/layer/BGM |
| capture 在写后 → PJSON 不能「写后再 capture」 | PJSON 短路在写前，用计算中间量或专用 calc 私有方法 |
| highlight 未进 Result | 一期可 null；二期从 highlight 服务回填 |
| Azure 多 scene 共享 asset 的 audioRange | 契约约定：相对 0 或绝对 range 与 NORMAL timeline 一致 |
| job 需识别 storageType | `projectService.getById` 已有 storageType |

---

## 7. 实施清单（实现时用）

- [ ] vcjs：`TtsSceneProvider` PJSON 读 + 伪 ProjectScene  
- [ ] vcjs：`GENERATE_VOICE_APPLY_VOICE_OVER` + `ApplyVoiceOver` content BO  
- [ ] vcjs：`PjsonProjectEditService.notifyApplyVoiceOver`  
- [ ] vcjs：`TtsSceneWriter.applyVoiceOver` PJSON 分支  
- [ ] vcjs：Azure `createNewSyntheticVoiceResourcesSync` PJSON 短路  
- [ ] vcjs：Align `insertSceneVoice`（或 parseSceneData 入口）PJSON 短路  
- [ ] visla-api：`generateSyntheticVoice` PJSON handler  
- [ ] PES：实现 `generate-voice:apply-voice-over`  
- [ ] 联调：Azure 单 scene / 多 scene；OpenAI align；失败回滚 status  

---

## 8. 结论

1. **`parseSceneData` 不拆**——同意；用 **入口级 PJSON 短路 + 已有 capture 字段** 接入。  
2. **foundation 可通用**：Provider / Assembler / Result / Writer 都是 PJSON 的挂钩点。  
3. **PJSON 设计核心**：DB 管 asset + 状态；PES 管 voice 结构与全部联动；Java 传「算好的 transcript/range/duration + 设置类配置」。  
