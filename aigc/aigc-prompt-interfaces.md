# AIGC Prompt 接口业务解析

> 本文档解析 `ProjectEditingV3Controller` 中三个 AIGC 相关接口的完整链路：**数据来源、NORMAL/PJSON 分支判断、最终请求的 AI 接口、异常处理**。
> 基于 `v2-pjson` 分支代码。PJSON 双路径总纲见 [pjson-impl.md](../../vme-pjson/pjson-impl.md)。

---

## 1. 概述：三个接口的共同架构

三个接口都遵循同一套 **NORMAL / PJSON 双路径** 模式：

```
Controller 端点
  └─ Service 方法（标 @PjsonProjectOp）
       └─ PjsonAspect 拦截：
            ├─ storageType == NORMAL → 执行原方法（数据来自 DB）
            └─ storageType == PJSON  → SHORT_CIRCUIT 到 PjsonProjectEditService 同名 handler（数据来自 PES + DB 附属表）
```

**共同流程**：ACL 校验 → 组装请求 → 调 AI 网关 → 落库审计 → 异常标记。

| 接口 | NORMAL 数据主源 | PJSON 数据主源 | 最终 AI 调用 |
|------|----------------|---------------|-------------|
| enhance-image-prompt | DB scene.transcript + projectAiProperty | PES scene.transcript + DB projectAiProperty | `aiGateWayClient.enhanceImageStillPrompt` |
| enhance-video-prompt | DB scene + refImages | PES scene + DB refImages | `aiGateWayClient.enhanceVideoMotionPrompt` |
| animate-storyboard | DB scene + timeline + candidate | PES scene.duration + DB candidate | `aiGateWayClient.createVideos` |

---

## 2. 接口清单

Controller 类前缀 `/v3/project/{projectUuid}`（`ProjectEditingV3Controller`）：

| HTTP 端点 | Controller 方法 | Service 方法 | opName | 模式 |
|----------|---------------|-------------|--------|------|
| `POST /enhance-image-prompt` | `enhanceImageStillPrompt` | `EnhancePromptService.enhanceImageStillPrompt` | `enhanceImageStillPrompt` | SHORT_CIRCUIT |
| `POST /enhance-video-prompt` | `enhanceVideoMotionPrompt` | `EnhancePromptService.enhanceVideoMotionPrompt` | `enhanceVideoMotionPrompt` | SHORT_CIRCUIT |
| `POST /animate-storyboard` | `animateAiStoryboard` | `ProjectSceneService.animateAiStoryboard` | `animateStoryboard` / `animateStoryboardSelective` | SHORT_CIRCUIT |

> `animate-storyboard` 单个端点承载两个 opName：`reqBody.scenes` 非空 → 选择性（`animateStoryboardSelective`）；为空 → 全量（`animateStoryboard`）。

---

## 3. enhance-image-prompt 详解

### 3.1 入口

`EnhancePromptService.enhanceImageStillPrompt(projectUuid, reqBody)`：
1. `projectService.getActiveByUuid` + `aclService.checkPermission(PROJECT_CAN_EDIT)`
2. `reqBody.sceneUuid` 非空时加载 scene
3. 创建审计记录 `enhancePromptRecordService.createInitRecord(IMAGE_PROMPT_ENHANCE)`
4. 构造请求 → 调 AI → buildRecordVo

### 3.2 数据来源（NORMAL vs PJSON）

| 数据项 | NORMAL 来源 | PJSON 来源（handler） |
|--------|------------|---------------------|
| **scene.transcript → voiceOver** | DB `projectSceneService.getById(sceneUuid).transcript`，`fetchVoiceOver` 解析 | PES `loadSceneDataFromPES(projectId, sceneId).transcript`，构造**伪 ProjectScene** 后同样走 `fetchVoiceOver` |
| targetVideoDesc | DB `projectAiProperty.videoIdea` | 同 NORMAL（projectAiProperty 仍在 DB） |
| videoProperties | DB `projectAiPropertyService.generateVideoProperties` | 同 NORMAL |
| visualPlan | DB `projectAiProperty.visualPlanLink` | 同 NORMAL |
| refImages | DB `projectAssetService.checkAndGetRefImageAsset`（project_asset） | 同 NORMAL |

**PJSON 关键差异**：scene 不查 DB，从 PES JSON 取 `transcript`，序列化成 String 塞进伪 `ProjectScene`，复用 `fetchVoiceOver`。其余附属数据（projectAiProperty、asset）都还在 DB，两条路径一致。

### 3.3 请求构造

`EnhancePromptRequestBuilder.buildEnhanceImageStillPromptReqBo` 组装 `EnhanceImageStillPromptReqBo`：
- `stillPrompt`（用户输入）+ `targetVideoDesc` + `videoProperties` + `visualPlan` + `voiceOver`
- `refImages` → `List<AigcImageBo>`（assetId / assetUrl / mediaType / metadata）
- `entityId = project.agentProjectId`，`entityType = AGENT_PROJECT`
- `extra.uuid = record.id`（回溯审计用）

### 3.4 AI 接口调用

```
aiGatewayService.enhanceImageStillPrompt(reqBo)
  └─ buildEnhanceWrapper: 把 reqBo 序列化 → 写 S3 json 文件 → 包装成 AiAgentBaseBo（含 jsonFile 链接）
  └─ aiGateWayClient.enhanceImageStillPrompt(wrapper)   ← 实际 AI 调用（同步）
```

> AI 请求体通过 **S3 json 文件中转**（不走内联 body），AI 侧按 msgType=`AIGC_IMAGE_PROMPT_ENHANCE` 消费。

### 3.5 异常处理

```
record = createInitRecord(...)            // 先建审计（INIT）
try {
    rspVo = aiGatewayService.enhanceImageStillPrompt(reqBo)
    return buildRecordVo(record, rspVo)   // 成功 markSuccess，rspVo.success=false 则 markFailed
} catch (Exception e) {
    enhancePromptRecordService.markFailed(record.getId(), userId)  // 标记失败
    throw e                              // 异常上抛（Controller 捕获后返回错误码）
}
```

- **AI 调用异常**：记录 `markFailed` + 重新抛出，前端收到错误。
- **AI 返回但 success=false**：`buildRecordVo` 内 `markFailed`，返回失败 VO（不抛异常）。

---

## 4. enhance-video-prompt 详解

结构与 image-prompt 几乎一致，**核心差异是 `mode`（运动模式）分支判断**。

### 4.1 mode 推断逻辑

`MotionVideoModeEnum` 决定生成方式，两个路径都有推断：

| 条件 | mode |
|------|------|
| `reqBody.mode` 显式指定 | 用指定值（`MotionVideoModeEnum.fromName`） |
| 无 refImages | `PROMPT_TO_VIDEO` |
| refImages.size == 1 | `FIRST_FRAME_TO_VIDEO` |
| refImages.size == 2 | `FIRST_AND_LAST_FRAME_TO_VIDEO` |
| refImages.size > 2 | `INGREDIENTS_TO_VIDEO` |

> NORMAL 在 `EnhancePromptService` 内推断；PJSON 在 handler 的 `resolveMotionMode(reqBody)` 推断，逻辑相同。

### 4.2 refImages 按 mode 组装（buildEnhanceVideoMotionPromptReqBo）

```
mode == FIRST_FRAME_TO_VIDEO / FIRST_AND_LAST_FRAME_TO_VIDEO:
    refImages → List<AigcFrameBo>（keyframes）
        - framePosition: i==0 → FIRST_FRAME, 否则 LAST_FRAME
        - AI_GENERATE 类型 asset 调 enrichAigcImageBoIfAiGenerate 补 stillPrompt + refImageList
    ingredientsRefImages = 空

mode == INGREDIENTS_TO_VIDEO:
    refImages → List<AigcKitBo>（slot = i+1）
    keyframes = 空
```

### 4.3 其余数据来源

与 image-prompt 完全一致（scene.transcript、projectAiProperty、asset 都来自 DB / PES transcript）。**PJSON 路径同样从 PES 取 transcript 构造伪 scene**。

### 4.4 AI 接口与异常

- AI 调用：`aiGatewayService.enhanceVideoMotionPrompt` → `aiGateWayClient.enhanceVideoMotionPrompt`（msgType=`AIGC_VIDEO_PROMPT_ENHANCE`，S3 json 中转）。
- 异常处理模式与 image-prompt 完全相同（markFailed + rethrow / buildRecordVo）。

---

## 5. animate-storyboard 详解

这是最复杂的接口：批量场景动画生成，涉及 candidate 驱动、全量/选择性、清除重建/补漏两种策略。

### 5.1 入口与分支

```
animateAiStoryboard(projectId, consumptionType, ignoreModel, enableAudio)     // 全量
animateAiStoryboard(projectId, consumptionType, scenes)                        // 选择性
```

两个重载都标 `@PjsonProjectOp`，PJSON 项目走 handler `animateAiStoryboardInternal`；NORMAL 走原方法。

### 5.2 数据来源

| 数据项 | NORMAL 来源 | PJSON 来源（handler） |
|--------|------------|---------------------|
| **scene 列表** | DB `projectSceneDao.queryProjectSceneListSorted` | PES `loadSceneDataFromPES` → `sceneDurationMap`（只取 duration） |
| **scene.duration** | DB project_scene.duration | PES JSON 的 duration |
| **candidate / motionVideo** | DB `candidateService.getByProjectIdWithStoryboardMetadata` | 同 NORMAL（candidate 在 DB） |
| aigcRel | DB `aiGeneratedContentRelService.getByProjectId` | 同 NORMAL |
| timeline（FOOTAGE） | DB `timelineService.getByProjectId` | **不读**（PJSON 下 timeline 在 pjson） |
| voice | DB `getSceneVoiceMetadataMap`（RefAudio） | `buildGlobalVoiceMap`（仅 global voice，agentProjectAttributes.voiceId） |
| assets | DB `projectAssetService.getAllAssetWithNewMediaInfo` | 按需 `saveAigcAsset` 新建 |

**PJSON 关键差异**：
- scene.duration 从 PES JSON 取（不在 DB）。
- voice 只用 global voice（不读 timeline 级 voice，因为 timeline 在 pjson）。
- 不依赖 `project_timeline` 表。

### 5.3 核心分支判断（clearFlagBoListPair，仅全量）

全量动画先计算 `clearRecommendFootage` 标志，决定走哪条组装策略：

**`clearRecommendFootage` 语义**：所有 scene 的推荐 footage 是否都已被采纳（即 `project_timeline`(FOOTAGE) 的 asset 能在 candidate 列表里匹配到）。

```
clearRecommendFootage = candidates 全部满足:
    scene 有 FOOTAGE timeline ∧ timeline.asset 能在 candidate.candidates 中匹配
```

| clearRecommendFootage | 策略 | 行为 |
|:---:|------|------|
| **true** | **全量重建** | 每个 scene 直接用 candidate 的 motionVideo 组装（`assembleBo`），不检查已有 aigcRel，check=true（只校验不落库？见 assembleBo check 参数） |
| **false** | **补漏** | 每个 scene 先检查是否已有 `sceneSelected && SYSTEM` 的 aigcRel；已有则跳过；FAILED 的走 `reAnimateScene` 重试；否则新增 |

**`sceneCannotAnimate` 判断**（决定 scene 是否可动画）：
- candidate 存在且 `aiStoryboardMetadataBo.motionVideo` 非空。
- 按 motionVideo.mode 检查 keyframes 数量与 errorCode（PJSON 版已确认：FIRST_FRAME 需 1 帧、FIRST_AND_LAST 需 2 帧、INGREDIENTS 需有 refImages）。
- 不满足则该 scene 被过滤（返回 null）。

> **选择性动画**（`animateStoryboardSelective`）不走 clearFlag 逻辑，直接按 `reqBody.scenes` 逐个组装，每个 scene 新建 asset + `saveAndAssembleAigcData`。

### 5.4 AI 接口调用（createVideo）

```
// 事务提交后异步触发（TransactionCommitUtil.afterCommit）
Boolean result = aiGeneratedContentService.createVideo(project, PROJECT, workspace, boList, extra);
```

`createVideo` 内部（同步）：
```
1. buildAigcReqBo(...)              // 组装 AigcGenerateVideoReqBo
2. aiGateWayClient.createVideos(reqBo)   ← 实际 AI 调用（批量）
   catch Exception → rspVo.success = FALSE
3. success == false → saveFailData(boList); return false
4. 读 rspVo.msgFile（S3 链接）→ 解析 AiAnimateSceneVo
5. 把 asset.taskId 回写到 AiGeneratedContent（按 aigc_id 匹配）
6. return success
```

> 注意：createVideo 在 `afterCommit` 中执行，**事务已提交**，boList 中的 asset/aigc/rel 已落库。AI 返回的是异步任务的 taskId，实际视频生成由 vcjs 侧 AIGC 回调链路完成。

### 5.5 异常处理与状态机

```
afterCommit:
  result = createVideo(...)
  boList.forEach(bo):
      status = result ? UPLOADING : FAILED
      更新 asset.processStatus = status
      if (result == false):
          更新 aigc.status = FAILED
      更新 rel.processStatus = status

  if (result == false):
      projectService.setProjectNormal(project)              // 项目回退到正常态
  else:
      sysTaskService.addCheckProjectAiRecommendStatusTask() // 加异步检查任务
```

**前置校验**（在调 AI 之前，事务内）：
- `projectAssetService.checkCredit(workspace, consumptionType, enableAudio, totalDuration)` → 额度不足抛 `ApiException`（事务回滚，不会进 afterCommit）。
- boList 为空 → 直接返回 project 详情，不调 AI。

**状态流转**：
- 进入：`setProjectProcessingAiStoryboard`（项目置为 AI storyboard 处理中）
- 成功：asset/aigc/rel → UPLOADING，加检查任务
- 失败：asset/aigc/rel → FAILED，项目回 NORMAL

---

## 6. 异常处理模式总结

| 接口 | AI 调用方式 | 异常捕获 | 失败标记 | 是否回滚事务 |
|------|-----------|---------|---------|:---:|
| enhance-image/video-prompt | 同步（网关内 try-catch） | Service 层 try-catch | `enhancePromptRecord.markFailed` | 否（AI 调用前 record 已建） |
| animate-storyboard | `afterCommit` 同步 | `createVideo` 内 try-catch | asset/aigc/rel → FAILED + 项目回 NORMAL | 否（已在 afterCommit，事务已提交） |

**共同点**：
- AI 调用前先建审计/业务记录，确保失败可追溯。
- AI 调用本身用 try-catch 包裹，失败不抛原始异常到最外层，而是转成业务失败标记。
- enhance 系列：异常上抛 → 前端拿错误码；animate 系列：异常吞掉转状态机。

---

## 7. 关键文件索引

| 角色 | 文件 |
|------|------|
| Controller | `controller/v3/ProjectEditingV3Controller.java`（animate-storyboard / enhance-image-prompt / enhance-video-prompt 端点） |
| enhance 编排 | `service/EnhancePromptService.java`（NORMAL） |
| enhance 请求构造 | `service/EnhancePromptRequestBuilder.java`（buildEnhanceImage/VideoPromptReqBo） |
| animate 编排 | `service/ProjectSceneService.java`（animateAiStoryboard + clearFlagBoListPair） |
| PJSON handler | `service/pjson/PjsonProjectEditService.java`（同名 handler，PES 数据源） |
| AI 网关 | `internal/ai/service/AiGatewayService.java`（enhanceImageStillPrompt / enhanceVideoMotionPrompt，S3 json 中转） |
| AI 客户端 | `internal/ai/.../AiGateWayClient`（enhanceImageStillPrompt / enhanceVideoMotionPrompt / createVideos） |
| AIGC 落库 | `service/AiGeneratedContentService.java`（createVideo / saveAndAssembleAigcData） |
| 审计记录 | `service/EnhancePromptRecordService.java`（createInitRecord / markSuccess / markFailed） |
| PES 数据读取 | `service/ProjectPjsonService.java`（loadSceneDataFromPES） |

---

## 8. 已知问题

- **`PjsonProjectEditService` 编译错误**（v2-pjson 分支既有）：animate 相关 handler（`assembleBoFromMetadata` / `reAnimateScene` / `assembleBoForSelectiveScene`）调用 `aiGeneratedContentService.saveAndAssembleAigcData` / `assembleAigcData` 时，voice 参数传 `SyntheticVoiceMetadata`，但方法签名要求 `RefAudio`（NORMAL 路径 `getSceneVoiceMetadataMap` 返回 RefAudio）。PJSON 的 `buildGlobalVoiceMap` 返回 `SyntheticVoiceMetadata`，类型未适配，需统一 voice 参数类型或加转换。位置：`PjsonProjectEditService.java:530 / 575 / 599`。
