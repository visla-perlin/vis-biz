# 上传配音 Aligner + 项目翻译 API 业务文档

> 范围：两个面向编辑态用户的运营级 API
>
> - `POST /v3/{projectUuid}/add-voice-over-aligner`：上传本地配音走 force-aligner 对齐到场景字幕
> - `POST /v3/{projectUuid}/duplicate-for-translation`：复制（或不复制）项目并翻译为目标语言
>
> 视角：业务定位 + 接口契约 + 链路 + PJSON 影响
> 配套：
>
> - [tts-flow.md](./tts-flow.md)（TTS 主链路 / SCRIPT_TRANSLATION 是同一类 AI 异步任务）
> - [tts-pes-sync-contract.md](./tts-pes-sync-contract.md)（PES 同步契约）
> - [tts-pjson-design.md](./tts-pjson-design.md)（TTS 在 PJSON 下的设计）

---

## 1. 业务定位对比

| 维度 | add-voice-over-aligner | duplicate-for-translation |
| ------ | ------------------------ | --------------------------- |
| 用户意图 | "我自录了一段配音，让字幕跟着我的语音走" | "把这个项目翻成另一种语言" |
| 输入 | 用户上传的 audio asset（可选指定 scene） | 目标语言（language + region）、是否重配 voice |
| 输出产物 | scene.transcript + scene.duration 变更，timeline 加 RECORD_VOICE_OVER | 翻译后 transcript + 新 voice（可选）+ 新 project（若 duplicate=true） |
| 触发链 | sync → SQS audio-aligner-req → aligner → audio-aligner-rsp → 写 DB | sync → aiGateway sendOtherTask → AI 网关回调 → vcjs 写 DB |
| 异步? | ✅ 是 | ✅ 是 |
| 可撤销? | ✅（手动删除 timeline / asset） | ❌（翻译已应用） |

---

## 2. `POST /v3/{projectUuid}/add-voice-over-aligner`

### 2.1 接口契约

#### 请求路径

```
POST /v3/{projectUuid}/add-voice-over-aligner
```

#### 请求体 `ProjectUploadVoiceOverReqBody`

```json
{
  "sceneUuid": "1532712462409801728",  // 可选，不填=全量；填了=只对该 scene
  "assetUuid": "1532811128252456960"   // 必填，用户上传的 audio asset
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|:---:|------|
| `sceneUuid` | String | 否 | 限定只对该 scene 走 aligner；不填=全量 |
| `assetUuid` | String | ✅ | 用户上传的 audio asset ID（必须 active） |

#### 响应

```json
{ "code": 0, "data": true }
```

#### 权限与锁

- `@PermissionRequired`：`PROJECT_CAN_EDIT`
- `@ProjectOperateLockRequired`：操作锁 `PROJECT_OP_GROUP`，`needEditLock=true`（要持编辑锁）

### 2.2 业务逻辑（同步阶段）

```java
public void addUploadVoiceOver(String projectUuid, ProjectUploadVoiceOverReqBody reqBody) {
    Project project = projectService.getActiveByUuid(projectUuid);

    // 1. 更新 project 状态为「配音对齐中」
    Project updateProject = new Project();
    updateProject.setId(project.getId());
    updateProject.setAiRecommendStatus(AiRecommendStatusEnum.PROCESSING_VOICE_ALIGN);
    projectService.updateByIdSelective(updateProject);

    // 2. 构造 alignerBo（SQS 消息体）
    VoiceOverAlignerBo alignerBo = new VoiceOverAlignerBo();
    alignerBo.setProjectId(projectUuid);
    alignerBo.setTraceId(TraceUtil.getTraceId());
    ProjectAiProperty projectAiProperty = projectAiPropertyService.getByProjectId(project.getId());
    if (projectAiProperty != null) {
        alignerBo.setLang(projectAiProperty.getAudioLanguage());
    }

    // 3. 把 input content（audios + scenes）序列化到 S3
    String fileKey = s3Service.generateTmpScenesFileKey(project.getId());
    s3Service.saveTextAsFile(fileKey, JsonUtil.asJson(constructUploadVoiceOverContent(projectUuid, reqBody)));
    alignerBo.setTextFile(s3Service.getS3Url(fileKey));

    // 4. 发 SQS audio-aligner-req（@Async 发，源 = "voice_aligner"）
    sqsService.sendUploadVoiceOverProcess(alignerBo);
}
```

### 2.3 SQS 消息体 `VoiceOverAlignerBo`

```jsonc
{
  "project_id": "<projectUuid>",
  "trace_id": "<traceId>",
  "source": "voice_aligner",     // 标识上传配音来源
  "lang": "en-US",                // 来自 project_ai_property.audio_language
  "text_file": "s3://...",        // 输入文件 URL（含 audios + scenes）
}
```

#### text_file 内容（`VoiceOverAlignerContentBo`）

```jsonc
{
  "audios": [                     // 通常 1 个 = 用户上传的配音 asset
    {
      "id": "1532811128252456960",
      "url": "https://cdn.../audio.mp3"
    }
  ],
  "scenes": [                     // 涉及的 scene（默认全量，过滤 intro/outro；可按 sceneUuid 过滤到 1 个）
    {
      "id": "1532712462409801728",
      "duration": 7830,
      "sentences": [               // 取 scene.transcript.sentences；缺失则从 items 转换
        { "duration": 2160, "content": "A cat's quiet day isn't empty.", "type": "" }
      ]
    }
  ]
}
```

### 2.4 异步链路

```
visla-api
  ├─ addUploadVoiceOver
  │   ├─ 更新 project.ai_recommend_status = PROCESSING_VOICE_ALIGN
  │   ├─ 构造 VoiceOverAlignerBo
  │   └─ sendUploadVoiceOverProcess（@Async vislaAsyncExecutor）
  │       └─ SQS: ${env}-audio-aligner-req（reqQueueName）
  ↓
外部 aligner 服务（消费 audio-aligner-req，运行 force-align）
  ↓
外部 aligner 写结果 → SQS audio-aligner-rsp
  ↓
vcjs
  ├─ VoiceoverAlignerService.handleMessage  @SqsConsumer("audio-aligner-rsp")
  ├─ VoiceOverAlignerHandlerService.processHandler
  │   ├─ success=true && totalMatch=true?
  │   │   ├─ insertSceneVoice（按 RECORD_VOICE_OVER 分类，写 timeline）
  │   │   │   ├─ PJSON：alignPjsonComputeService.computeAlignResult + VoiceOverResult
  │   │   │   └─ NORMAL：parseSceneData + 写 project_timeline / project_scene
  │   │   ├─ syncAvatarDurationsFromScenes（avatar 时长联动）
  │   │   ├─ statusTransitionService.transitionToEditingWithNormalStatus
  │   │   └─ notificationService.pushVoiceOverAlignerResult（afterCommit）
  │   └─ 否则 handleFailure
  └─ updateByIdSelective(project ai_recommend_status=NORMAL)
```

### 2.5 关键代码索引

| 角色 | 位置 |
| ------ | ------ |
| Controller | `visla-api/.../controller/v3/ProjectEditingV3Controller.java` `addUploadVoiceOver` (line 1227) |
| Service 主逻辑 | `visla-api/.../service/ProjectV2Service.java` `addUploadVoiceOver` (line 2908) |
| 内容构造 | `ProjectV2Service.constructUploadVoiceOverContent` (line 2980) |
| SQS 发送 | `visla-api/.../service/async/AwsSqsService.java` `sendUploadVoiceOverProcess` (line 486) |
| SQS 消费 | `vcjs/.../sqs/consumer/service/voice/VoiceoverAlignerService.java` `handleMessage` |
| 业务处理 | `vcjs/.../service/VoiceOverAlignerHandlerService.java` `processHandler` (line 120) |
| Insert 链路 | `VoiceOverAlignerHandlerService.insertSceneVoice`（NORMAL/PJSON 分流） |

### 2.6 PJSON 影响（断裂点）

| 链路环节 | 代码位置 | NORMAL | PJSON |
| --------- | --------- | :---: | :---: |
| 同步状态更新 | `addUploadVoiceOver` | ✅ DB project | ✅ DB project |
| 读 project_scene.transcript | `constructUploadVoiceOverContent` | ✅ DB | ❌ **PES**（PJSON 无 DB scene） |
| 读 asset URL | `mediaAssembleService.getById` | ✅ DB | ✅ DB（asset 在 DB） |
| 写 project_timeline RECORD_VOICE_OVER | `insertSceneVoice` → `parseSceneData` | ✅ DB | ❌ → notify PES apply-voice-over |
| 写 project_scene.transcript / duration | `insertSceneVoice` | ✅ DB | ❌ → notify PES apply-voice-over |
| 写 status / notification | `processHandler` | ✅ DB + MQ | ⚠️ PES 侧 |

**结论**：add-voice-over-aligner 在 PJSON 下**完全无 PES 感知**（`constructUploadVoiceOverContent` 走 `projectSceneService.listAllByByProjectId`，`insertSceneVoice` NORMAL 分支直接写 DB）。

**修复路径**：需要把以下两处加 `@PjsonProjectOp` 切面：

1. `constructUploadVoiceOverContent` — PJSON 下从 PES 读 scene transcript
2. `insertSceneVoice` — PJSON 下走 `alignPjsonComputeService` + notify PES apply-voice-over

---

## 3. `POST /v3/{projectUuid}/duplicate-for-translation`

### 3.1 接口契约

#### 请求路径

```
POST /v3/{projectUuid}/duplicate-for-translation
```

#### 请求体 `ProjectTranslationReqBody`

```json
{
  "language": "es",                  // 目标语言（必填，且必须与源语言不同）
  "region": "es-ES",                 // 目标地区（可选，server 端按 locale 推算）
  "duplicate": true,                 // 是否复制新项目；false = 在原项目上翻译（覆盖）
  "voiceRedo": true,                 // 翻译后是否重配 voice（TTS）
  "voiceUuid": "1234"                // 重配 voice 时指定 voiceId（可选）
}
```

| 字段 | 类型 | 必填 | 说明 |
| ------ | ------ | :---: | ------ |
| `language` | String | ✅ | 目标语言 code（不能与源语言相同） |
| `region` | String | 否 | 不填则按 `language` 查 `synthetic_voice_metadata.locale_name` 推算 |
| `duplicate` | boolean | 否 | true=新建项目副本并翻译；false=在原项目上翻译覆盖 |
| `voiceRedo` | boolean | 否 | 翻译完成后是否重配 voice（TTS 重新合成） |
| `voiceUuid` | String | 否 | 重配 voice 时的 voice ID |

#### 响应 `ProjectVo`

```json
{
  "code": 0,
  "data": {
    "id": "1532711043975975280",
    "uuid": "...",
    "title": "...",
    "language": "es-ES",
    "ai_recommend_status": "PROCESSING_TRANSLATION",
    ...
  }
}
```

#### 权限与锁

- `@PermissionRequired`：`OWNER / MEMBER`（keyName=projectUuid）
- `@ProjectOperateLockRequired`：`PROJECT_DUPLICATE`，`lockTime=30s`，`needEditLock=true`
- `@EntityParentRecordAnnotation`：operation=`DUPLICATE`（仅当 duplicate=true）
- `@CreditLimitationRequired`：`PROJECT_RECOMMEND_FOOTAGE`（消耗 credit）

### 3.2 业务逻辑（同步阶段）

#### 3.2.1 Controller 入口

```java
@PostMapping("/duplicate-for-translation")
public Result<ProjectVo> duplicateProjectForTranslation(
    @PathVariable String projectUuid,
    @RequestBody ProjectTranslationReqBody reqBody) {
    if (StringUtils.isBlank(reqBody.getLanguage())) {
        throw new ApiException(ApiMessage.ILLEGAL_PARAMS);
    }
    Project project = projectService.getAndEnsureProjectExisting(projectUuid);
    aclService.checkPermission(project, CapabilityAclParamsV3Enum.PROJECT_CAN_USE_ROUGH_CUT.getParams());
    return Result.success(
        projectV2Service.duplicateProjectAndDoTranslation(AuthUtil.getUserId(), project, reqBody));
}
```

#### 3.2.2 Service 主逻辑

```java
@Transactional
public ProjectVo duplicateProjectAndDoTranslation(Long userId, Project project, ProjectTranslationReqBody reqBody) {
    Long projectId = project.getId();
    Long newProjectId = projectId;

    // 1. 解析 region（按 language 查 synthetic_voice_metadata）
    String region = getLanguageRegion(reqBody.getLanguage());
    reqBody.setRegion(region);

    // 2. 取源语言
    String originLanguage = "";
    ProjectAiProperty projectAiProperty = projectAiPropertyService.getByProjectId(projectId);
    if (Objects.nonNull(projectAiProperty)) {
        originLanguage = LanguageUtil.getCorrectLanguage(projectAiProperty.getAudioLanguage());
    }

    // 3. 校验：不能翻成同语言
    if (StringUtils.equalsIgnoreCase(originLanguage, reqBody.getLanguage())) {
        throw new ApiException(ApiMessage.PROJECT_LANGUAGE_SAME_AS_SOURCE_LANGUAGE);
    }

    // 4. duplicate=true 时先复制项目（完整复制走 duplicateProjectCore + PjsonAspect）
    if (reqBody.isDuplicate()) {
        ProjectVo projectVo = duplicateProject(userId, project);
        newProjectId = projectVo.getId();
    }

    // 5. 翻译（实际写 project + 发 AI 任务）
    return translateProjectInner(userId, project, reqBody, projectAiProperty);
}
```

#### 3.2.3 translateProjectInner

```java
public ProjectVo translateProjectInner(Long userId, Project project, ProjectTranslationReqBody reqBody, ProjectAiProperty projectAiProperty) {
    Long projectId = project.getId();
    String region = reqBody.getRegion();

    // 1. 更新 project_ai_property 语言
    if (Objects.nonNull(projectAiProperty)) {
        projectAiProperty.setAudioLanguage(reqBody.getLanguage());
        projectAiProperty.setTextOverlayLanguage(reqBody.getLanguage());
    }

    // 2. project.version++ + ai_recommend_status = PROCESSING_TRANSLATION
    Project update = new Project();
    update.setId(projectId);
    update.setVersion(project.getVersion() + 1);
    update.setAiRecommendStatus(AiRecommendStatusEnum.PROCESSING_TRANSLATION);
    projectService.updateByIdSelective(update);

    // 3. 清高亮（翻译后 highlight 失效）
    projectStyleService.deleteByProjectIdAndType(projectId, ProjectStyleTypeEnum.HIGH_LIGHT_STYLE);
    projectSceneHighlightService.clearByProjectId(projectId);
    project.setVersion(update.getVersion());

    // 4. 组装 translationInfo + 发 AI 任务
    AiRecommendV2FileContentBo.TranslationInfo translationInfo = new AiRecommendV2FileContentBo.TranslationInfo();
    translationInfo.setTargetLanguageCode(reqBody.getLanguage());
    translationInfo.setTargetLanguageRegion(region);
    ProjectRecommendConf projectRecommendConf = projectSettingsService.getRecommendConf(projectId);
    if (Objects.isNull(projectRecommendConf)) {
        projectRecommendConf = new ProjectRecommendConf();
    }
    aiOtherQueueTaskV3Service.translationV2(projectId, translationInfo, reqBody.getVoiceUuid(), reqBody.isVoiceRedo(), projectRecommendConf);

    // 5. 返回最新 ProjectVo（带 language 字段）
    ProjectVo resultProject = projectService.getProjectDetail(userId, String.valueOf(projectId));
    resultProject.setLanguage(reqBody.getLanguage());
    return resultProject;
}
```

### 3.3 异步链路（translationV2）

```java
public void translationV2(Long projectId, TranslationInfo translationInfo, String voiceUuid, boolean voiceRedo, ProjectRecommendConf projectRecommendConf) {
    Project project = projectService.getActiveById(projectId);
    User user = userService.getActiveUser(project.fetchOwnerID());

    // 1. 从 DB 读 scenes + groups
    List<ProjectScene> projectScenes = projectSceneService.queryProjectSceneSortedList(projectId);
    if (CollectionUtils.isEmpty(projectScenes)) {
        log.info("all the project scenes are empty.");
        return;
    }

    // 2. 构造 taskSceneInnerInfo（按 sceneId 建 candidate map）
    TaskInnerParams taskInnerParams = new TaskInnerParams(RecommendTypeEnum.ALL_SCENE_TRANSLATE, AiOtherTaskEnum.SCRIPT_TRANSLATION);
    Map<Long, AiTextRecSceneCandidateBo> sceneIdAndCandidateMap = fillUpSceneIdMapForTranslate(project, projectScenes);
    TaskSceneInnerInfo taskSceneInnerInfo =
        generateAiRecommendV2FileContentBo(project, sceneIdAndCandidateMap, projectScenes, taskInnerParams);
    taskSceneInnerInfo.getAiRecommendV2FileContentBo().setTranslationInfo(translationInfo);

    // 3. 组装 aiRecommendV2ReqBo（带 voiceRedo / voiceUuid / targetLanguage / targetLanguageRegion）
    AiRecommendV2ReqBo aiRecommendV2ReqBo = new AiRecommendV2ReqBo();
    aiRecommendV2ReqBo.setEntityId(projectId);
    aiRecommendV2ReqBo.setEntityType(EntityTypeEnum.PROJECT);
    aiRecommendV2ReqBo.setTrackingId(TraceUtil.getTraceId());
    aiRecommendV2ReqBo.setFeePlan(feePlanService.getFeePlanByDomain(project).getKey());
    aiRecommendV2ReqBo.setUserId(user.getId());
    aiRecommendV2ReqBo.setTaskType(AiOtherTaskEnum.SCRIPT_TRANSLATION.getTask());
    Map<String, Object> idsMap = new HashMap<>(4);
    idsMap.put("sceneIds", generateExtra(taskSceneInnerInfo.getSceneIndexAndIdMap()));
    idsMap.put("textIds", generateExtra(taskSceneInnerInfo.getSceneNewTextIndexAndIdMap()));
    idsMap.put("segmentIds", taskSceneInnerInfo.getSegmentIndexAndIdMap());
    idsMap.put("voiceRedo", voiceRedo);
    idsMap.put("voiceUuid", voiceUuid);
    idsMap.put("targetLanguage", translationInfo.getTargetLanguageCode());
    idsMap.put("targetLanguageRegion", translationInfo.getTargetLanguageRegion());
    aiRecommendV2ReqBo.setExtra(generateExtra(idsMap));
    aiRecommendV2ReqBo.setSearchOptions(new RecommendFootageBo(projectRecommendConf));

    // 4. 持久化版本号 + 写 json_file 到 S3
    projectService.updateProjectVersion(project, user.getId());
    String fileKey = s3Service.generateProjectTranslationFileKey(String.valueOf(user.getId()), projectId, project.getVersion());
    String fileLink = s3Service.saveTextAsFile(fileKey, JsonUtil.asJsonNoNull(taskSceneInnerInfo.getAiRecommendV2FileContentBo()));
    aiRecommendV2ReqBo.setJsonFile(fileLink);

    // 5. 发 AI 网关（异步）
    ApiMessage apiMessage = aiGatewayService.sendOtherTask(aiRecommendV2ReqBo);
    if (apiMessage != ApiMessage.SUCCESS) {
        throw new ApiException(ApiMessage.PROJECT_AI_TRANSLATION_FAILED);
    }
}
```

### 3.4 完整异步链路

```
visla-api
  ├─ duplicateProjectAndDoTranslation
  │   ├─ 解析 region、校验 source ≠ target
  │   ├─ duplicate=true? → duplicateProjectCore(@PjsonProjectOp) → PjsonAspect 切面
  │   └─ translateProjectInner
  │       ├─ 更新 project_ai_property（audio + text_overlay 语言）
  │       ├─ project.version++ / ai_recommend_status = PROCESSING_TRANSLATION
  │       ├─ 清高亮（highlight 与语言绑定）
  │       └─ translationV2（@Transactional）
  │           ├─ 读 scene + group
  │           ├─ 组装 translationInfo / voiceRedo / voiceUuid / targetLanguage
  │           ├─ 写 json_file 到 S3
  │           └─ aiGatewayService.sendOtherTask（同步发，AI 异步处理）
  ↓
外部 AI 网关（消费 translation 任务，跑翻译）
  ↓
外部 AI 网关 → vcjs OtherCB 回调（taskType = SCRIPT_TRANSLATION）
  ↓
vcjs
  ├─ OtherCBFactoryService.doTask → aiTranslateV2Service
  ├─ AiTranslateV2Service.doTask
  │   ├─ 解析 extra（voiceRedo / voiceUuid / targetLanguage）
  │   ├─ 写 project_scene.transcript / project_text / project_segment 翻译后内容
  │   ├─ voiceRedo=true → voiceGenerateService 重配 voice（TTS）
  │   ├─ project_ai_property 持久化
  │   └─ notificationService.pushNotification
  └─ project ai_recommend_status = NORMAL
```

### 3.5 duplicate=true vs duplicate=false

| 行为 | duplicate=true | duplicate=false |
| ------ | :---: | :---: |
| 是否新建项目 | ✅（完整复制） | ❌（原项目） |
| 新项目 owner | 当前 user | — |
| 新项目 teamspace | 复制 | — |
| 新项目父记录 | `entity_parent` 记录 DUPLICATE 关系 | — |
| 翻译结果写入 | 新项目 | 原项目 |
| 数据回滚方式 | 删新项目 | 无（不可逆） |
| credit 消耗 | ✅ 推荐 footage 重配 | ❌ 仅翻译 |

> **实现注意**：当 `duplicate=true` 时，`duplicateProject(userId, project)` 内部走 `duplicateProjectCore`（`@PjsonProjectOp("duplicate")`）。PJSON 项目下 PjsonAspect 自动拦截；NORMAL 直接走 `duplicateProjectInner`。

### 3.6 voiceRedo 联动

| voiceRedo | 行为 |
| :---: | ------ |
| `true` | 翻译完成后 vcjs `AiTranslateV2Service.doTask` 调用 `voiceGenerateService.generateVoice(...)` 重新合成 voice（参数从 extra 取 `voiceUuid`） |
| `false` | 不重配 voice，scene.transcript 翻译后保留旧 voice |

> voiceRedo 行为需在 vcjs `AiTranslateV2Service` 内根据 `voiceRedo` 字段判断调用链。

### 3.7 关键代码索引

| 角色 | 位置 |
| ------ | ------ |
| Controller（普通项目） | `visla-api/.../controller/v3/TeamspaceProjectController.java` `duplicateProjectForTranslation` (line 569) |
| Controller（agent 项目） | `visla-api/.../controller/v3/TeamspaceAgentProjectController.java` `duplicateAgentProjectForTranslation` (line 920) |
| Service 主逻辑 | `visla-api/.../service/ProjectV2Service.java` `duplicateProjectAndDoTranslation` (line 2033) |
| 翻译实际任务 | `translateProjectInner` (line 2063) |
| 翻译异步发任务 | `visla-api/.../internal/ai/service/AiOtherQueueTaskV3Service.java` `translationV2` (line 165) |
| 复制核心 | `ProjectV2Service.duplicateProjectCore`（`@PjsonProjectOp("duplicate")`） |
| VC 端翻译消费 | `vcjs/.../service/AiTranslateV2Service.java` `doTask` |
| 工厂分发 | `vcjs/.../service/OtherCBFactoryService.java`（SCRIPT_TRANSLATION → aiTranslateV2Service） |

### 3.8 PJSON 影响

| 链路环节 | 代码位置 | NORMAL | PJSON |
| --------- | --------- | :---: | :---: |
| duplicate 走 PjsonAspect | `duplicateProjectCore` | 直接 duplicate | ✅ `PjsonProjectEditService.duplicateHandler` |
| 读 scenes / texts / segments | `translationV2` + `fillUpSceneIdMapForTranslate` | ✅ DB | ❌ **PES**（PJSON 无 DB scene） |
| 写翻译后 scene.transcript / text / segment | `AiTranslateV2Service.doTask` | ✅ DB | ❌ → notify PES |
| 写 project_ai_property | `translateProjectInner` | ✅ DB | ✅ DB |
| 清高亮 | `projectSceneHighlightService.clearByProjectId` | ✅ DB | ⚠️ PES（高亮在 PES？） |
| voiceRedo 重配 | `voiceGenerateService` | ✅ DB | ❌ → notify PES apply-voice-over |

**结论**：duplicate-for-translation 在 PJSON 下**部分支持**：

- ✅ duplicate=true 路径已通过 `@PjsonProjectOp("duplicate")` 切面支持 PJSON
- ❌ translationV2 读 scenes、vcjs AiTranslateV2Service 写翻译结果、voiceRedo 都未做 PES 适配

**修复路径**：translationV2 读 PES scenes；AiTranslateV2Service 通过 `@PjsonProjectOp` 切面 → `PjsonProjectEditService` 处理：写 PES scene.transcript + text + segment + 触发 voice redo（同样经 apply-voice-over 通知）。

---

## 4. 完整 SQS / 异步队列索引

| 队列 | 触发方 | 消费方 | 任务类型 | 涉及 API |
| ------ | -------- | -------- | --------- | --------- |
| `${env}-audio-aligner-req` | visla-api `sendUploadVoiceOverProcess` | 外部 aligner 服务 | 配音对齐请求 | add-voice-over-aligner |
| `${env}-audio-aligner-rsp` | 外部 aligner 服务 | vcjs `VoiceoverAlignerService` | 对齐结果回写 | add-voice-over-aligner |
| AI 网关翻译任务 | visla-api `aiGatewayService.sendOtherTask` | AI 网关 | SCRIPT_TRANSLATION | duplicate-for-translation |
| AI 网关翻译回调 | AI 网关 | vcjs `AiTranslateV2Service.doTask` | SCRIPT_TRANSLATION | duplicate-for-translation |

---

## 5. 状态机

### add-voice-over-aligner

```
project.ai_recommend_status:
  NORMAL →（发起）→ PROCESSING_VOICE_ALIGN →（aligner 回调成功）→ NORMAL
                                                 →（失败）→ NORMAL + 通知失败
```

### duplicate-for-translation

```
project.ai_recommend_status:
  NORMAL →（发起）→ PROCESSING_TRANSLATION →（AI 网关回调成功）→ NORMAL
                                                →（失败）→ NORMAL + 通知失败
```

---

## 6. 跨项目契约总结

| 契约 | 描述 |
| ------ | ------ |
| 配音 aligner 输入 | scene.transcript 必须有 sentences 或 items（至少一种） |
| 配音 aligner 输出 | aligner 返回 `totalMatch` 必须为 true 才写 DB，否则走失败 |
| 配音 timeline 分类 | RECORD_VOICE_OVER（区别于 SYNTHETIC_VOICE_OVER） |
| 翻译入参 | language 必须 ≠ source language；voiceRedo=true 时 voiceUuid 可选（不填则按语言自动选） |
| 翻译结果 | 翻译后 highlight 被强制清空（多语言下高亮位置不一致） |
| 翻译异步任务类型 | `AiOtherTaskEnum.SCRIPT_TRANSLATION` |
| 翻译 taskType | `RecommendTypeEnum.ALL_SCENE_TRANSLATE` |
