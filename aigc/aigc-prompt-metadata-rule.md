# AIGC Prompt 与 Storyboard Metadata 规则

> 范围：scene 级 AIGC 视频生成（generate-video / regenerate-video / get-storyboard-metadata）  
> 视角：prompt 存哪、selected rel 的生命周期、metadata 的回退规则  
> 适用：NORMAL（DB）与 PJSON（PES + DB）双路径，标注差异

---

## 1. 涉及的表与字段

| 表 | 关键字段 | 作用 |
|----|---------|------|
| `ai_generated_content` | `motion_prompt`、`gen_model`、`aspect_ratio`、`video_length`、`status` | **prompt 主存储**；每次生成一条记录（regenerate 复用原行，见 §3） |
| `ai_generated_content_rel` | `aigc_id`、`project_id`、`scene_id`、`asset_id`、`scene_selected`、`process_status`、`type`(USER/SYSTEM)、`status`(ACTIVE/DELETED) | scene↔aigc↔asset 关联；`scene_selected=true` 表示「当前 scene 选用的那条」 |
| `project_recommend_candidate` | `ai_storyboard_metadata`（JSON，含 `motionVideo.promptJson` / `motionPrompt`） | AI storyboard 推荐；**metadata 回退数据源** |
| `project_asset` + `visla_media` | 素材链接、尺寸 | 生成的视频文件实体 |

> **prompt 只存在 `ai_generated_content.motion_prompt` 一个地方**，不进 PES、不进 candidate 表。`promptJson`（enhance 后的结构化 prompt）不入库，请求时临时算。

---

## 2. generate-video（首次生成）

### 2.1 流程

```
generate-video
  saveAndAssembleAigcData:
    1. saveAigc → insert ai_generated_content（motion_prompt = reqBody.prompt, status=INIT）
    2. setSceneNotSelectedByScene → 该 scene 下所有 rel.scene_selected=false
    3. addAigcRel → insert ai_generated_content_rel（scene_selected=true, type=USER, process_status=INIT）
    4. assemble ref images → ai_generate_content_ref_image
    5. build asset/media（insert project_asset + visla_media）
  afterCommit → callAigcGatewayAndSyncStatus → AI 网关，回写 process_status(UPLOADING/FAILED)
```

### 2.2 结果

- `ai_generated_content`：**新增 1 行**（prompt + 模型 + 状态）
- `ai_generated_content_rel`：**新增 1 行**（`scene_selected=true`）
- 此时 `getSelectedByScene` 能查到该 rel

---

## 3. regenerate-video（重新生成）

### 3.1 关键规则：**复用原行，不新增**

regenerate 走 `retryAigc`，**update 原 aigc / asset / media 行**，不 insert 新记录：

```java
// retryAigc（ProjectAssetService）
existAigc.setMotionPrompt(paramBo.getPrompt());     // update 原行 prompt
existAigc.setStatus(INIT);                           // 重置状态
aiGeneratedContentService.updateByIdSelective(existAigc);   // ← update，不是 insert

// asset/media 也 update（reset process_status=INIT，换 originalLink）
```

rel 行**也复用**（`existRel` 不变，`scene_selected` 仍是 true）。

### 3.2 流程

```
regenerate-video（reqBody 带 aigcUuid）
  1. 校验 existAigc（aigcUuid）/ existRel（归属本 project+scene）
  2. checkUnfinishedSceneRecords（不允许并发未完成）
  3. retryAigc:
       update ai_generated_content（prompt/model/status=INIT）
       update project_asset / visla_media（reset=INIT）
       （rel 不动）
  4. assembleAigcData（用 existAigc/existRel 重组）
  5. afterCommit → callAigcGatewayAndSyncStatus
```

### 3.3 结果

- `ai_generated_content`：**原行 update**（prompt 被新值覆盖，status=INIT）
- `ai_generated_content_rel`：**原行不变**
- 不会产生多条候选——同一 scene 同一时刻只有一条 `scene_selected=true` 的 rel

> ⚠️ **修正常见误解**：regenerate **不是**「新增一条记录」。它复用原 aigc/rel/asset，覆盖 prompt 后重试。想要「换一个 prompt 生成新的、保留旧的」要用 generate（新建），不是 regenerate。

---

## 4. selected rel 的生命周期与删除

### 4.1 selected 的含义

`ai_generated_content_rel.scene_selected = true` 表示「该 scene 当前选用这条 AIGC」。一个 scene 同时**最多一条** `scene_selected=true`（`addAigcRel` / `selectRelByScene` 都会先 `setSceneNotSelectedByScene` 全清再置一条）。

### 4.2 删除（delete-aigc）

| 路径 | DB 操作 | selected 变化 |
|------|---------|--------------|
| NORMAL `deleteSceneAigcRel` | `scene_selected=false`（软取消选中，行仍在） | true→false |
| PJSON `deleteSceneAigcRel` | `status=DELETED`（逻辑删除整行） | 行消失 |

两种结果都会导致 `getSelectedByScene` **返回 null**。

> 注意：delete-aigc **不动** `project_timeline`（画面）和 `project_recommend_candidate`（推荐数据）。
> - NORMAL 下画面要单独 `delete-footage` 才删
> - candidate 数据始终保留在 DB，是 metadata 回退的基础

---

## 5. get-storyboard-metadata 的回退规则（核心）

### 5.1 入口

`GET /scene/get-storyboard-metadata` → `ProjectV2Service.getStoryboardMetadataVo(consumptionType, enableAudio, scene)`

### 5.2 判定逻辑（三级回退）

```
getStoryboardMetadataVo(scene):
  rel = getSelectedByScene(projectId, sceneId)   // scene_selected=true 且 status=ACTIVE 的 rel

  ① rel != null:
       aigc = getById(rel.aigcId)
       if aigc != null && aigc.mediaType == VIDEO:
           return convertToVo(aigc)              // ← 用选中 AIGC 的 prompt
       // aigc 不存在或非 VIDEO → 落 ②
  ② 回退:
       return getSceneDefaultAigc(scene)         // ← 用 candidate 数据
```

### 5.3 回退数据源：`getSceneDefaultAigc`

```
getSceneDefaultAigc(scene):
  candidate = getByProjectIdAndSceneIdWithStoryboardMetadata(projectId, sceneId)
  return getByProjectRecommendCandidate(candidate)   // 从 candidate.aiStoryboardMetadata 取
```

`getByProjectRecommendCandidate` 从 `project_recommend_candidate.ai_storyboard_metadata.motionVideo` 取：
- `motionPrompt` / `promptJson`（推荐 prompt）
- `mode`（PROMPT_TO_VIDEO / FIRST_FRAME / ...）
- `keyframes` / `ingredientsRefImages`（参考图）
- `negativePrompt`

### 5.4 何时触发回退

`getSelectedByScene` 返回 null 的场景：

| 场景 | 原因 |
|------|------|
| **从未生成过 AIGC** | 无 rel |
| **delete-aigc 后** | NORMAL: scene_selected=false；PJSON: status=DELETED |
| **rel 对应的 aigc 不存在/非 VIDEO** | 数据异常 |

→ 这些场景下，get-storyboard-metadata 都**回退到 `project_recommend_candidate`**，返回推荐的默认 prompt + 模式。

---

## 6. 完整时序示例

```
场景：首次 generate-video → 删除 → get-storyboard-metadata

t1  generate-video（prompt="A cat"）
      DB: ai_generated_content{prompt="A cat", status=INIT}
          ai_generated_content_rel{scene_selected=true}
      → getSelectedByScene → rel ✓
      → getStoryboardMetadataVo 返回 "A cat"

t2  生成完成（异步回调）
      rel.process_status=COMPLETED, aigc.status=SUCCESS
      → getStoryboardMetadataVo 仍返回 "A cat"

t3  delete-aigc
      NORMAL: rel.scene_selected=false
      PJSON:  rel.status=DELETED
      → getSelectedByScene → null

t4  get-storyboard-metadata
      → getSelectedByScene=null → 回退
      → getSceneDefaultAigc → candidate.aiStoryboardMetadata.motionVideo.promptJson
      → 返回推荐默认 prompt（不是 "A cat"）
```

---

## 7. PJSON 下的差异

| 环节 | NORMAL | PJSON | 差异说明 |
|------|--------|-------|---------|
| prompt 存储 | DB `ai_generated_content.motion_prompt` | **同（DB 不变）** | PES 不存 prompt |
| generate 提交 | insert aigc/rel | 同 + notify PES `update-scene-aigc`（aigcRel，无 prompt） | PES 多一步登记 |
| regenerate | update 原行（retryAigc） | **同（复用原行）** | 一致 |
| delete | `scene_selected=false` | `status=DELETED` + PES `clear-scene-aigc` | PJSON 更彻底 |
| selected 判断 | `getSelectedByScene`（DB） | **同（DB）** | 都读 DB rel |
| metadata 回退 | candidate（DB） | **同（DB）** | candidate 永远在 DB |
| get-storyboard-metadata scene 来源 | DB `project_scene` | PES scene（伪 ProjectScene，含 duration） | 仅取数分叉，逻辑一致 |

**关键不变量**：
- prompt 永远在 DB `ai_generated_content.motion_prompt`
- selected 判断永远查 DB `ai_generated_content_rel`
- metadata 回退永远查 DB `project_recommend_candidate`
- PES 只管「画面结构」（brolls/duration/aigcRel 引用），不管 prompt/metadata/selected

---

## 8. 要点速记

1. **prompt 只在 DB `ai_generated_content.motion_prompt`**，PES 不存。
2. **generate 新增** aigc/rel 行；**regenerate 复用原行**（update，不新增）。
3. **selected = `scene_selected=true` 且 `status=ACTIVE`**，一个 scene 最多一条。
4. **delete-aigc**（NORMAL 软取消 / PJSON 逻辑删除）→ `getSelectedByScene` 返回 null。
5. **没有 selected 时，get-storyboard-metadata 回退 `project_recommend_candidate`**（推荐默认 prompt）。
6. PJSON 下 prompt/selected/candidate 三者仍在 DB，PES 只管画面结构——这是理解整套数据流的关键。
