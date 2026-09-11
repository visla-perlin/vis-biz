# Avatar 业务域 — 数字人（AI 主播）

> 跨 visla-api / vcjs 共用的 **业务规则 / 契约** 文档。
> 实现细节（类、方法、行号、落库字段）见同目录 [technical.md](./technical.md)，本文每节末尾以 `⚙️ 见 technical.md §x` 互引。

---

## 1. 业务定位

Avatar 是 Visla 的「**数字人 / 虚拟主播**」能力：

- 用户在 **Workspace** 里创建一个数字人角色（Avatar），系统为它生成多种造型（Look）与多个景别（动作视角）的素材。
- 在 **Project** 场景里使用时，系统按场景构图，把数字人 Look + 台词音频（TTS / 录音）合成为「**会说话的动作视频**（lip-sync 视频）」。
- 最终导出时，数字人视频作为独立的 `scene.avatar` 层参与渲染。

**两种 Look 引擎**（贯穿全部流程，务必区分）：

| 引擎 | 枚举 | 模型策略 | 素材产出 | 适用 |
| ------ | ------ | --------- | --------- | ------ |
| **Basic** | `AvatarLookTypeEnum.BASIC(1)` | `basic`（头部 only） | 单条前景/抠像视频 + 采样口播视频 | 头像类 |
| **Advance** | `AvatarLookTypeEnum.ADVANCE(2)` | `advanced`（半身） | 多景别图 + 视频（每景别一条 AvatarLookAsset） | 半身动作类 |

---

## 2. 核心概念

| 概念 | 含义 |
| ------ | ------ |
| **Avatar** | 数字人角色（Workspace 级素材） |
| **AvatarLook** | 角色的一种造型 + 动作视频载体。一个 Avatar 可有多个 Look（basic / advance），同组 Look 共享 `groupId` |
| **AvatarLookAsset** | 一个 Look 的**多景别资产**（每个 shot_type 一条：头部特写/全身/中景/侧面…），仅 advance 有 |
| **AvatarLookGroup** | Look 分组（advance look 按 group 组织，同组 basic+advance 共享 groupId） |
| **AvatarRecommendImage** | AI 生成的角色参考候选图（供用户挑选后生成 Avatar） |
| **WorkspaceFavoriteAvatar** | Workspace 收藏（avatar+look） |
| **ProjectAvatar** | 某场景里使用的一个 avatar 实例（Project 级），引用一个 `lookId` |
| **景别 shot_type** | `ShotTypeEnum`：head_shot / base_full_body / medium_wide / medium_shot / medium_close / subtle_side |
| **构图 shotComposition** | `AvatarCompositionModeEnum`：数字人如何出现在画面里（见 §3.3） |
| **动作视频** | 场景级 lip-sync 视频：Look 素材 + 台词音频 → 会说话的视频 |

---

## 3. 数据模型（业务视角）

### 3.1 两层数据

```
Workspace 层（素材库）                    Project 层（使用实例）
┌──────────────┐  1:N   ┌────────────┐    ┌──────────────┐
│   Avatar     │───────▶│ AvatarLook │◀───│ ProjectAvatar│ (sceneId, lookId)
│ (数字人角色) │        │  (造型)    │    │ (场景实例)   │
└──────────────┘        └─────┬──────┘    └──────────────┘
       │                       │ 1:N                ▲
       │ N:1                   ▼                    │ 引用 lookId
┌──────┴───────┐        ┌──────────────┐           │
│AvatarLookGroup│       │AvatarLookAsset│           │
│  (分组)      │        │ (多景别资产)  │           │
└──────────────┘        └──────────────┘           │
                                                  │
┌──────────────────┐    ┌──────────────────────┐  │
│AvatarRecommendImage│  │WorkspaceFavoriteAvatar│  │
│  (推荐候选图)     │    │  (收藏)              │  │
└──────────────────┘    └──────────────────────┘  │
```

> ⚙️ 全字段清单见 technical.md §2

### 3.2 Avatar 来源

`AvatarSourceTypeEnum`：GENERATE（AI 生成）/ UPLOAD（上传）/ IMPORT（从 clip 导入）/ CAPTURE（摄像头捕获）。
非 GENERATE 来源受 FeePlan 限制（仅 Business/Enterprise），且有数量上限。

### 3.3 场景构图模式（shotComposition）

| 模式 | 含义 |
| ------ | ------ |
| `HOST_ONLY` | 数字人占满画面（主持人模式） |
| `HOST_PIP` | 数字人画中画（右下角小窗） |
| `VO_ON_VISUAL` / `VISUAL_ONLY` / `VISUAL_PIP` | 无数字人（纯视觉） |
| `SMART_COMPOSITION` | 智能构图（同时给 host_only + host_pip 策略） |
| `SOT` | 单镜头 |

---

## 4. 完整业务流程

### 4.1 Avatar 创建（4 种来源）

- **GENERATE**：`saveDescribe`（描述）→ AI 生成角色图（`AVATAR_CHARACTER_IMAGE`）→ 产出 `AvatarRecommendImage` 候选图列表 → 用户选图 → `generateByRecommendImage` 生成正式 Avatar。
- **UPLOAD / CAPTURE**：直接上传/捕获图片 → 异步生成缩略图。
- **IMPORT**：从 Project 的 Clip 素材导入（拷贝 mediaLink + thumbnailLink）。

创建后 `processStatus = COMPLETED`（GENERATE 在选图前为 INIT→PROCESSING）。

> ⚙️ 实现见 technical.md §4.1

### 4.2 Look 视频生成（basic + advance 同时建）

用户对 Avatar 执行 `generatePreviewLook` 时，**同一个请求同时初始化 BASIC 和 ADVANCE 两个 Look**（共享 groupId），发 `AVATAR_LOOK_VIDEO` 给 AI 网关，AI 同时产出两种 look 视频。

> ⚙️ AI 请求结构见 technical.md §4.2

### 4.3 Look 视频后处理（vcjs）

Look 视频生成后还需异步后处理，两条引擎路径不同：

**Basic Look**（串行两步）：

```
look video 回调成功 → MATTING(抠像) → 前景/抠像/透明图/缩略图
                  → 自动触发 SAMPLING(采样) → sampleVideo + 声音口播视频 → COMPLETED
```

**Advance Look**（一步多景别）：

```
look video 回调成功 → ADVANCED_AVATAR → 每个 shot_type 返回 image+video → 落 AvatarLookAsset(每景别一条) → COMPLETED
```

任一步失败 → `processStatus = FAILED` + 释放 workspace 锁 + 通知用户。

> ⚙️ 落库字段见 technical.md §4.3

### 4.4 变声（换声音）

`changeVoiceId(lookId, voiceId)`：

- BASIC → 重新 `SAMPLING`（用新音色合成 sampleVideo）。
- ADVANCE → `changeAdvancedAvatarVoice`（AI 替换视频音轨）→ 回调更新。
- 两路径失败均置 `processStatus = FAILED`。

> ⚙️ 见 technical.md §4.4

### 4.5 项目使用（ProjectAvatar 绑定场景）

- 给项目/场景加 avatar 时，创建 `ProjectAvatar`，**初始即 image 占位**（`mediaType=IMAGE`, `assetId=0`, `renderStatus=INIT`）。
- 根据构图模式 + 视频宽高比，自动计算 avatar 的位置（scale / coordOffset）。
- avatar 时间跨度默认 = 整个 scene duration（`startTime=0`, `endTime=scene.duration`）。

> ⚙️ 见 technical.md §4.5

### 4.6 预览动作视频生成（lip-sync，跨场景合并）

触发预览渲染时，按 lookType 分流（`generatePreviewAvatarVideos`）：

**Basic**（跨场景合并 — 关键机制）：

- 同一 project 的连续 scene 的 avatar **合并成一个长视频**生成。
- 回调返回分段信息（每个 scene 的 duration），建**一个** VislaMedia + 一个 ProjectAsset。
- 每个 scene 的 `ProjectAvatar` 共享同一 `assetId`，通过 `startTime/endTime` 偏移引用长视频的不同区间，`reused=TRUE`。

**Advance**（逐场景回调）：

- 一次回调只处理一个 scene。
- duration = `min(max(lastWordEndTime, ttsDurationMs), sceneDuration)`。
- 每场景独立建 VislaMedia + ProjectAsset + speaking 音频资产。

成功 → `renderStatus=COMPLETED`；失败 → `renderStatus=FAILED`（**不影响项目主流程**，仅通知用户）。

> ⚙️ 跨场景合并落库见 technical.md §4.6

### 4.6.1 Advance 预览（force-align，半身动作视频）

> Advance 与 Basic 的**关键差异**：basic 是「从 voice 重新生成 lip-sync video」（voice-driven）；
> advance 是「**已有 look video + 用户 voice → 重新 force-align 生成新 lip-sync**」（video-driven）。
> 也就是说 advance 的 look 视频是预生成的、动作固定的半身视频，回调里需要**把 voice 重新贴到 video 上并 force-align**。

**Advance 预览生成入口**：`ProjectAvatarService.generateAdvancePreviewAvatarVideos(teamspace, project, advanceAvatars)`。

**advanceAvatars 来源**（`regenerateUnrenderedAvatars`）：

```text
full project avatar list
  ├─ filter(assetId==0 || reused==FALSE) → unrendered
  └─ group by lookType
       ├─ basicAvatars   → generateBasicPreviewAvatarVideos（跨场景合并，参见 §4.6 上半部）
       └─ advanceAvatars → generateAdvancePreviewAvatarVideos（本节）
```

**ai 请求组装**（`CreateSceneAdvanceAvatarVideoBo`）：

- 每个 scene 独立一条 `AvatarScene`（**不跨场景合并**）。
- 每条带 `assetId`（提前生成，预发请求给 AI 网关）+ `assetUrl` / `assetMattingUrl` / `assetOriginUrl`。
- 可选 `environmentImage`（`agentProjectAttributes.autoWallpaper=true` 且项目选中 wallpaper 时附上）。
- 携带 `voiceOver` / `voice`（`projectSceneService.getSceneVoiceMetadataMap`），供 AI 把 voice 贴到 look video 上。

**回调成功**（`vcjs/.../ProjectAvatarService.handleAdvancedPreviewAvatarSuccess`）：

1. 落 VislaMedia + ProjectAsset（**额外多落一个 speaking audio asset** + `media_synthetic_voice`）：

   ```text
   ProjectAsset videoAsset   = AVATAR_PREVIEW / VIDEO / COMPLETED
   ProjectAsset ttsAsset      = speaking audio（generateAvatarSpeakingAsset）
   media_synthetic_voice.voiceId  = avatarLook.voiceId
   media_synthetic_voice.entityId = ttsAsset.id
   media_synthetic_voice.entityType = PROJECT_ASSET
   ```

2. 调 `audioWordTimestampService.processAdvanceAvatarAudio(...)` 走 **force-align**（与 basic 不同：basic 不重新触发 TTS，advance 必须 force-align 重新生成 timeline）。
3. 计算 `realDuration = min(max(lastWordEndTime, ttsDurationMs), sceneDuration)`，落 `ProjectAvatar`：

   ```text
   reused = TRUE
   renderStatus = COMPLETED
   mediaType = VIDEO
   startTime = 0
   endTime = realDuration
   duration = realDuration
   ```

4. 根据 `videoFormat` 调 `resetTo{16_9,9_16,1_1}Advance` 重算 `scale` + `coordOffset`（advance 专用位置计算，参见 §8）。
5. `projectPostProcessorService.projectNotify(project, SUCCESS)`。

**回调失败**（`handleAdvancedPreviewAvatarFailed`）：所有 advance avatar `renderStatus=FAILED` + `entityErrorLog` 记错 + 通知用户。**不影响项目主流程**。

**计费**：advance 预览**不计费**（参见 §7）。

**§3.4 regenerateUnrenderedAvatars 支持「单个 scene」**：

> 该入口当前**仍是全项目**（参数只有 `teamspace, project`，无 `sceneId`）。
> 但 advanceAvatars 已**按 scene 独立生成**（单 scene 单 video），`audioWordTimestampService.processAdvanceAvatarAudio` 也按 scene 调用。
> 因此**业务上每个 scene 各自产生一条 lip-sync 视频**，UI 可从 advance 资产列表中按 scene 看到生成进度（不是 API 入口限定单 scene）。
> 是否有单 scene 入口 API 需在 visla-api 侧进一步查证（参见 `ProjectEditingV3Controller.regenerateUnrenderedAvatars` 是否后续加 sceneId 参数）。

> ⚙️ 落库字段与回调全流程见 technical.md §4.6.1

### 4.7 TTS 与 Avatar（关键链路）

**核心关系**：Avatar 视频由场景的 voice over 音频（TTS/录音）+ 字级时间戳驱动，生成 lip-sync 视频。计费即按「lip-sync 秒数」计。

**voice 来源优先级**（`getSceneVoiceMetadataMap`）：

1. 项目级 `AgentProjectAttributes.voiceId`
2. Look 级 `AvatarLook.voiceId`
3. Scene 级 TTS 音频（`ProjectTimeline(SYNTHETIC_VOICE_OVER)`）+ 其绑定 voiceId

**TTS 生成的两条路径**（`POST /v3/{projectUuid}/voice/synthetic`）：

| 维度 | Azure 普通语音 | OpenAI / MiniMax（AI TTS） |
| ------ | --------------- | --------------------------- |
| vcjs 落点 | `SyntheticVoiceConsumer` → `regenerateVoiceByAzure` | `AudioWordTimestampConsumer` → `processAlignerBos` |
| scene 时长更新 | `updateProjectTranscriptWithWordsTimeStamp` | `VoiceOverAlignerHandlerService.insertSceneVoice` |
| avatar duration 同步 | ✅ `syncAvatarDurationsFromScenes` | ✅ `syncAvatarDurationsFromScenes` |
| 重新生成 avatar 视频 | ⚠️ **regenerate 不触发**（仅 createVoice 触发） | ✅ finally 主动触发 `generatePreviewAvatarVideos` |

**visla-api 侧 finally 块**（TTS 请求时立即释放 avatar 预览）：

- `forAllProject=true` → `releasePreviewVideo(project, null)`：全项目 avatar 转 image 占位。
- 指定 `sceneIds` → `releasePreviewVideo(project, sceneIds)` + `resetContinuousAvatarPreviewVideo`：指定 scene 转 image + 连续同 look 链标记重生成（**advance look 跳过**）。

**完整时序**（见 technical.md §4.7 图）：

```
HTTP → generateSyntheticVoice
  ├─ [finally] releasePreviewVideo / resetContinuousAvatarPreviewVideo（avatar→image 占位）
  └─ 异步发 TTS 请求
        ↓ (vcjs 消费)
  TTS 生成 → 更新 scene duration → syncAvatarDurationsFromScenes（avatar duration 跟随）
        ↓ (OpenAI/MiniMax 路径 或 Azure createVoice)
  generatePreviewAvatarVideos → AI 生成 lip-sync 视频 → 回调写 ProjectAvatar
```

> ⚙️ 完整时序 + 落库字段见 technical.md §4.7

### 4.8 导出

- **导出预检**：`checkAvatarRenderStatus` 要求所有 avatar `renderStatus=COMPLETED`，否则拒绝导出。
- **导出组装**：每个 scene 的 avatar 组装为 `ExportSceneAvatarBo`（zindex + metadata + avatarParts），通过 `scene.avatar` 传递（**不走通用 layer 链**）。
- `assetId>0` → 构造 `avatarParts[AvatarPartTimeBo{assetId, assetTime[start,end,duration]}]`；`assetId=0` → avatarParts 空 + `needDoAvatar=true`（告知渲染管线）。
- **导出本身不触发生成**，生成靠导出前的 `regenerateUnrenderedAvatars` 接口或异步回调完成。
- PDF 导出强制 `needDoAvatar=false`。

> ⚙️ 见 technical.md §4.8

---

## 5. 状态机

### 5.1 AvatarLook.processStatus（素材处理状态）

```
INIT ──▶ PROCESSING ──▶ COMPLETED
              │
              └────────▶ FAILED
```

- Basic：INIT →(look video)→ PROCESSING →(matting)→ PROCESSING →(sampling)→ COMPLETED
- Advance：INIT →(look video)→ PROCESSING →(advanced_avatar)→ COMPLETED

### 5.2 ProjectAvatar.renderStatus（场景实例渲染状态）

```
INIT(0) ──▶ PROCESSING(3) ──▶ COMPLETED(9)
                │
                └────────────▶ FAILED(12)
```

前端据此显示「生成中 / 视频 / 重试」。失败被隔离，不影响项目主流程。

### 5.3 Project.aiGenerateVoiceStatus

TTS 请求时置 `PROCESSING`，vcjs 处理完成后回 `NORMAL`。

---

## 6. image → video 转化场景（完整清单）

> 导出前 `regenerateUnrenderedAvatars` 的重生成判定条件：`assetId == 0 || reused == FALSE`。
> 以下所有场景都会让 avatar 满足该条件，从而在导出/重生成时触发 AI 视频生成 + lip-sync。

### 6.1 A 类：显式转 IMAGE 占位（必触发重生成）

| # | 场景 | 方法 | 入口 |
| --- | ------ | ------ | ------ |
| 1 | 首次给项目/场景加 avatar | `initProjectAvatarData` | addProjectAvatarLook / addOrReplaceSceneAvatarLook |
| 2 | agent project 自动创建 | vcjs `buildProjectAvatar` | vcjs 生成链路 |
| 3 | 全项目替换 look | `replaceAvatarLook` | ProjectEditingV3Controller |
| 4 | 按 group 替换 look | `replaceAvatarLookByGroupId` | ProjectEditingV3Controller |
| 5 | 单 scene 替换 look | `replaceSceneAvatarLook` | ProjectLayerV3Controller |
| 6 | 单 scene 新增/替换 | `addOrReplaceSceneAvatarLook`(替换分支) | ProjectLayerV3Controller |
| 7 | **声音元数据变化 / TTS 重配音** | `releasePreviewVideo` | SyntheticVoiceMetadataService |
| 8 | scene duration 变化 / 重新生成 voice over | `resetAvatarPreviewVideoInEditScene` | AvatarLayerHandler |
| 9 | 场景合并（不满足连续同look复用） | `adjustAvatarInMergeScene`(fallback) | AvatarLayerHandler |
| 10 | 场景拆分（不满足时间对齐） | `adjustAvatarInSplitScene`(fallback) | AvatarLayerHandler |

落库：`assetId=0, startTime=0, endTime=0, mediaType=IMAGE, renderStatus=INIT, reused=FALSE`（保留 lookId / 位置参数）。

### 6.2 B 类：只标记重生成（reused=FALSE + INIT，保留 mediaType）

| # | 场景 | 方法 | 备注 |
| --- | ------ | ------ | ------ |
| 11 | 场景编辑后连续同 look 链 | `resetContinuousAvatarPreviewVideo` | ⚠️ **ADVANCE 直接 return 跳过** |
| 12 | 合并后连续同 look 链 | `resetAvatarPreviewVideoInMergeScene` | ⚠️ **ADVANCE 直接 return 跳过** |
| 13 | look 被删 → 换默认 look | `changeDeletedProjectAvatar` | assetId=0 |

### 6.3 C 类：只调时间（不触发重生成）

| # | 场景 | 方法 | 行为 |
|---|------|------|------|
| 14 | 场景裁剪 cut | `adjustTimeInCutScene` | 仅改 offset/duration |

> ⚙️ 各方法实现见 technical.md §4.9

---

## 7. 计费

- **计量**：lip-sync 视频总时长（毫秒）÷1000 = 秒。
- **事件**：`ResUsageRptEvent.avatarPreviewEvent`（operationType=`Avatar lip sync video render`, resource type=`lip_sync`, unit=`second`）。
- **消费**：`ResUsageRptEventConsumer` → `CreditConsumeHandleService` → 按 feePlan 单价 × 秒数扣 workspace credit。
- **归属**：agent project 记 `AGENT_PROJECT`，普通 project 记 `PROJECT`，金额计算逻辑相同。
- ⚠️ **仅 basic 预览成功（`handlePreviewAvatarSuccess`）计费**；advance 预览成功（`handleAdvancedPreviewAvatarSuccess`）**不在此计费**。

> ⚙️ 见 technical.md §4.10

---

## 8. BASIC vs ADVANCE 完整差异

| 维度 | BASIC | ADVANCE |
| ------ | ------- | --------- |
| 模型策略 | basic（头部） | advanced（半身） |
| **驱动方向** | **voice-driven**（从 voice 重新生成 lip-sync video） | **video-driven**（已有 look video + 用户 voice → force-align） |
| 素材 | 单条前景/抠像视频 + sample 口播视频 | 多景别 AvatarLookAsset（每 shot_type 一条 image+video） |
| 后处理 | matting → sampling 两步串行 | advanced_avatar 一步多景别 |
| 变声 | 重新 sampling | changeAdvancedAvatarVoice 替换音轨 |
| 预览生成 | 跨场景合并（一个长视频，按偏移切片） | 逐场景回调（每场景独立 video） |
| 预览计费 | ✅ reportAvatarCreditCost | ❌ 不计费 |
| 连续链重置(§6.2 #11/12) | ✅ 标记 reused=FALSE | ❌ 直接 return 跳过 |
| duration 计算预览 | segment duration 累加 | min(max(lastWordEndTime, ttsMs), sceneDuration) |
| **回调复用 timeline** | ❌（basic 不重跑 TTS） | ✅ `processAdvanceAvatarAudio` force-align |
| **额外落 speaking audio asset** | ❌ | ✅ `generateAvatarSpeakingAsset` + `media_synthetic_voice` |
| **位置重算** | ❌ | ✅ `resetTo{16_9,9_16,1_1}Advance` 按 videoFormat 重算 scale / coordOffset |
| 预生成 AI 请求体预填 | ❌（一个长 video 由 AI 拼） | ✅ 每 scene 独立 `AvatarScene`，预发 `assetId` + `assetUrl/mattingUrl/originUrl` |
| 预览失败 | renderStatus=FAILED，不计费 | renderStatus=FAILED，不计费（外加 entityErrorLog） |
| 预览前置 image 占位来源 | INIT 或 reused=FALSE | INIT 或 reused=FALSE（多 scene 独立，需多走一次 AI） |

---

## 9. 跨项目职责边界

| 维度 | visla-api | vcjs |
| ------ | ----------- | ------ |
| 定位 | 同步、面向用户 | 异步、回调与批处理 |
| 对外接口 | WorkspaceAvatarController（CRUD/生成/变声/收藏）<br>ProjectEditingV3Controller（regenerate/delete unrendered） | — |
| Avatar/Look CRUD | ✅ 主写 | 读 + 回调局部更新 |
| AI 请求发起 | ✅ AiAvatarService 构造 BO | advance 场景视频由 vcjs AiAgentService 发起 |
| Look 后处理 | 发起 matting(SQS) | ✅ matting/sampling/advance 回调消费 |
| 项目 avatar 编辑 | ✅ 编辑态（绑定/构图/位置/合并拆分/宽高比） | ✅ 预览生成 + 回调 + 计费 |
| TTS→avatar duration 同步 | — | ✅ syncAvatarDurationsFromScenes |
| 预览视频生成 | ✅ 同步入口（用户手动） | ✅ 异步主链路（消息驱动） |
| 计费 | 额度/权限 | ✅ 上报 |

> ⚠️ 两项目都有 `ProjectAvatarService` / `AvatarLookService` / `AvatarService` / `AiAvatarService` 同名类，职责不同，改动时务必确认是哪一侧。

---

## 10. 风险与注意事项

1. **Azure regenerate voice 不触发 avatar 视频重生成**：用户用 Azure 重新配音后，avatar 停留在「image 占位 + duration 已更新」状态，直到导出或下次流程才重生成。OpenAI/MiniMax regenerate 则主动重生成。**改 Azure 链路时注意**。
2. **ADVANCE look 在连续链重置被跳过**（§6.2 #11/12）：场景编辑/合并后 advance 的连续 scene **不会重置也不会 mark**，代码直接 `return`，保留原 `ProjectAvatar` 状态（assetId / mediaType / reused / renderStatus 都不变）。
   - **业务上不安全**：后续若有 regenerateUnrenderedAvatars 之类的全项目重生成，advance 依赖 `reused=FALSE` 或 `assetId=0` 判定，**advance 场景编辑后不会触发重生成**，除非用户手动 regenerate。NORMAL 路径下默认接受这个行为；PJSON 路径下不一致，参见 `vme-pjson/pjson-avatar-design.md` §6.3 业务差异段。
3. **枚举跨项目不同步**：`AvatarProcessEnum`（vcjs 多 `ADVANCED_AVATAR(5)`）；`AiMessageTypeEnum`（vcjs 多 `AVATAR_CHANGE_VOICE(3004)`，3002 name 两边不同）。
4. **advance 预览不计费**：仅 basic 计费，确认是否符合业务预期。
5. **场景裁剪不触发重生成**（§6.3 #14）：裁剪后 avatar asset 不变，只调 offset/duration。
6. **生成失败隔离**：avatar 失败不影响项目主流程，仅标记 FAILED + 通知，导出预检会拦住未完成的。
7. **look video 回调 processStatus 仍 PROCESSING**：需 matting+sampling 全部完成才 COMPLETED，前端轮询注意中间态。

---

## 11. 变更同步提醒

涉及 Avatar 共享表 / 枚举变更时，**必须双向同步检查**（详见根 AGENTS.md §4.1）：

- DB 列变更：两边 Entity + Mapper XML + Flyway（visla-api 负责）
- 枚举值变更：两边 enums 包
- 消息类型：两边 AiMessageTypeEnum（当前已有不一致，见 §10.3）

---

*本文档基于 visla-api / vcjs 源码梳理，实现细节与行号见 [technical.md](./technical.md)。*
