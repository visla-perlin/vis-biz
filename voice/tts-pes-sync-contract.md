# TTS ↔ PES 同步对接说明

> 受众：PES / API / Job 三方对齐需求  
> 状态：**待确认（未实现）**  
> 关联：`tts-pjson-design.md`、`tts-flow.md`、foundation 分支 `refactor/tts-scene-provider`（已 merge 入 `v2-pjson`）  
> 前提：PJSON 项目；`storageType=PJSON`；scene 结构在 PES，**音频二进制与 asset 元数据在 Visla DB**
>
> **文档归属**：本文档仅描述 normal 视角下 TTS 与 PES 同步的契约。
> PJSON 路径下的 TTS 协议实现见 `vme-pjson/pjson-tts-design.md`。
> avatar 联动规则（不在本文档范围）见 `vis-biz/avatar/README.md`。

---

## 1. 结论先行（请先确认）

| 节点 | 是否同步到 PES | 说明 |
| ------ | :-------------: | ------ |
| **仅音频文件生成成功**（S3 有 mp3，DB 有 `project_asset`） | **否（默认）** | asset 只在 DB；PES 不存文件内容 |
| **可上屏的 voice 结果就绪**（含起止时间 + transcript 时间戳） | **是** | 统一走 `generate-voice:apply-voice-over` |
| Azure 本地 TTS 完成 | **是（一次）** | 生成 + 时间戳同一链路，结束时一次 notify |
| OpenAI/Minimax：AI 回调建 asset 后、align 前 | **否** | 尚无可靠 word 级时间戳，不写 voiceOvers |
| OpenAI/Minimax：align 回调成功 | **是（一次）** | 与 Azure 共用同一 action 与 payload 形状 |
| TTS 失败 / 部分 scene 失败 | **见 §5** | 状态在 DB；结构是否清理需确认 |

**一句话：不是「有 audio 就同步 PES」，而是「有可应用的 voiceOver + transcript 时间轴时再同步 PES」。**

---

## 2. 数据归属

| 数据 | 权威存储 | PES 是否持有 |
| ------ | ---------- | :------------: |
| 音频文件（S3 link、md5、duration） | DB `project_asset` / media | 否（仅引用 assetUuid） |
| voice 与 asset 绑定 | DB `media_synthetic_voice` | 否 |
| `project.ai_generate_voice_status` | DB | 否（前端可走 project 状态） |
| scene.voiceOvers（category / range / volume） | **PES** | 是 |
| scene.transcript（items/sentences/时间戳） | **PES** | 是 |
| scene.duration / pauseTime / timeOffset | **PES** | 是 |
| layer / blur / text / BGM / avatar / kenburns 联动 | **PES** | 是 |
| 发音纠正字典 | DB workspace_pron_corr | 否（Job 算完再写 transcript） |

读路径：PES 结构里的 `assetUuid` → API 回填 DB asset 详情（与 AIGC footage 一致）。

---

## 3. 两条链路的同步时机

### 3.1 Azure（本地 HD/非 HD 同步）

```
API 发起 SQS
  → Job doTTS（本地生成音频）
  → 写 DB project_asset（+ media_synthetic_voice）
  → 【NORMAL】写 timeline + scene.transcript/duration + BGM/kenburns…
  → 【PJSON】不写 timeline/scene 结构
  → 组装 VoiceOverResult（最终态字段）
  → ★ 同步 PES：generate-voice:apply-voice-over（一次，可多 scene）
  → DB：ai_generate_voice_status = NORMAL + 通知用户
```

**生成 audio 后是否单独再调 PES？**  
**不需要。** 与「可上屏结果」合并为**一次** `apply-voice-over`。

### 3.2 OpenAI / Minimax（AI 异步 + align）

```
API 调 AI 网关
  → AI 回调 processTtsCb
       → 写 DB project_asset（音频已在 S3）
       → 发音纠正 + 发 aligner SQS
       → ★ 此阶段不同步 PES 结构
  → Align 回调 processAlignerBos / insertSceneVoice
       → 【NORMAL】parseSceneData 写 timeline/scene/highlight + 联动
       → 【PJSON】不写上述结构；用 align 结果 + 最终字段组装 VoiceOverResult
       → ★ 同步 PES：generate-voice:apply-voice-over（一次）
       → DB：status + 通知
```

**AI 回调「只有 audio」时是否同步 PES？**  
**默认否。** 理由：

1. 尚无 align 后的 `audioRange` / word 级 `transcript.items`  
2. PES 若先挂「无时间轴 voiceOver」会闪错误态，align 后再改一次  
3. 前端应以 `ai_generate_voice_status=PROCESSING` 等待，直到 align 完成  

**若产品要求「先显示处理中占位」**：可二期增加弱同步（见 §6 可选），一期不做。

---

## 4. 同步协议

### 4.1 通道

```text
Job / API → PES notify_action_ready
  action_id: generate-voice:apply-voice-over
```

与 AIGC `generate-video:apply-footage` 同通道、不同 action。

### 4.2 时机与调用方

| 时机 | 调用方 | 备注 |
|------|--------|------|
| Azure 同步链路成功结束 | vcjs `TtsSceneWriter`（经 captureFromAzureReload） | 事务提交后 afterCommit 更佳 |
| Align 全量处理成功结束 | vcjs `TtsSceneWriter`（经 captureFromAlign） | 多 audio 文件可多次 insertSceneVoice，**建议聚合成一次 notify** 或按文档约定「每个 alignerBo 一次」 |

**推荐：每个「用户可感知完成」事件只 notify 一次**，payload 含本次涉及的全部 `scenes[]`。

### 4.3 action_content（对接字段）

```json
{
  "scenes": [
    {
      "sceneUuid": "<sceneId 字符串>",
      "asset": {
        "assetUuid": "<project_asset.id>",
        "mediaType": "AUDIO",
        "assetType": "SCENES_SYNTHETIC_VOICE_OVER",
        "duration": 5747
      },
      "voiceCategory": "SYNTHETIC_VOICE_OVER",
      "audioRange": { "startTime": 0, "endTime": 5747 },
      "voiceVolume": 1.0,
      "transcript": {
        "items": [],
        "sentences": [],
        "originalVoiceover": "",
        "originalKeywords": []
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

| 字段 | 谁算 | PES 用途 |
| ------ | ------ | ---------- |
| asset.* | Job（DB 已落 asset） | voiceOvers 挂载 + assets 索引 |
| audioRange | Job（Azure boundary / aligner audioInfo） | voiceOver 起止 |
| transcript.* | Job（含发音纠正还原后最终态） | 写 scene.transcript |
| sceneDuration / pauseTimeHint / forceAlignWithVoice | Job（与现网算术一致） | 写 duration/pause；PES 可再校验 |
| voiceVolume / creationTemplate / syntheticAll / slowdownToFit | API/Job 设置 | 音量、联动策略、BGM 范围 |
| highlightData | Job 可选 | 有则写高亮；null 则不动或清空（**需 PES 确认**） |

### 4.4 PES 收到后应做

1. 对每个 scene：**替换** `voiceOvers`（SYNTHETIC_VOICE_OVER，挂 assetUuid + range + volume）  
2. 写 `transcript` / `duration` / `pauseTime`  
3. **timeOffset 级联**（后续 scene）  
4. layer / blur / text / BGM / avatar / kenburns 等附属联动（PES 自有数据）  
5. 保证 `assetUuid` 出现在可读的 assets 索引中  

**PES 不算**：发音纠正、aligner 算法、S3 上传。

---

## 5. 失败与部分成功

| 场景 | DB | PES（建议） |
| ------ | ----- | ------------- |
| TTS/align 全失败 | status 失败 + 通知 | **不**发 apply-voice-over；或显式 clear（二期） |
| 多 scene 部分 match 失败 | 与 NORMAL 一致：成功 scene 写结果 | payload **只含成功 scene**；失败 scene 保持旧 voice 或空（**需产品确认**） |
| notify PES 失败 | asset 已在 DB；status 是否回滚 | Job 打日志 + 可重试；**需确认是否阻断「成功通知用户」** |

---

## 6. 可选二期（一期不做）

| 能力 | 说明 |
| ------ | ------ |
| audio-ready 占位 | AI 回调后 notify「仅 assetUuid、无 range」——一般不建议 |
| clear-voice-over | 失败或用户取消时清 PES voiceOvers |
| highlightData 必填 | foundation 未回填 highlight；二期从 DB 读出再传 |
| 按 scene 增量 notify | 长片多 scene 时拆包 |

---

## 7. 待 PES / 产品确认清单（动手前）

请逐条回复 **是 / 否 / 改**：

1. **确认**：仅 audio 落库、无时间轴时，**不同步 PES**。  
2. **确认**：Azure 完成与 Align 完成都用 **同一** `generate-voice:apply-voice-over`。  
3. **确认**：PES 全权做 timeOffset / layer / BGM / avatar / kenburns。  
4. **确认**：`sceneUuid` = Visla sceneId 的十进制字符串（与 AIGC 一致）。  
5. **确认**：`highlightData == null` 时 PES 行为 = **保留旧高亮** 还是 **清空**。  
6. **确认**：部分 scene 失败时，payload 是否 **只带成功 scene**。  
7. **确认**：notify 失败时，用户侧是否仍可显示「配音成功」（仅 DB asset 成功）。  
8. **确认**：多 segment / 多 audio 文件时，一次 notify 多 scene 还是多次 notify。  

---

## 8. 与实现的映射（确认后动手）

| 步骤 | 工程 |
| ------ | ------ |
| Provider 读 PES scene | vcjs `TtsSceneProvider` |
| 短路 NORMAL 结构写（PJSON） | Azure `createNewSyntheticVoiceResourcesSync`、Align `insertSceneVoice` 入口 |
| 组 Result + notify | 已有 `capture*` + `TtsSceneWriter.applyVoiceOver`（PJSON 已接 `notifyApplyVoiceOverIfPjson` 骨架） |
| action 常量 / content BO | vcjs `PjsonNotifyActionConstants` / `VmeNotifyActionReadyContentBo.ApplyVoiceOver` |
| API 发起 | visla-api `@PjsonProjectOp generateSyntheticVoice` |
| PES handler | PES `generate-voice:apply-voice-over` |

---

## 9. 流程图（对齐用）

### Azure

```text
[audio 生成] → [DB asset] → [算 transcript/duration]
       → ★ apply-voice-over → [PES 结构]
       → [DB status NORMAL]
```

### OpenAI / Minimax

```text
[AI audio] → [DB asset] → (PES 无结构同步)
       → [aligner]
       → [算 transcript/duration]
       → ★ apply-voice-over → [PES 结构]
       → [DB status NORMAL]
```

---

**请 PES / 产品确认 §7 后，再在 `v2-pjson` 上实现短路写库 + 联调。**  
在此之前 **不改 PES 行为相关业务代码**（merge foundation 除外，已完成）。
