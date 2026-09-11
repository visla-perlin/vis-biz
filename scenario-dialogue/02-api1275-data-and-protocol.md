# Scenario Dialogue Video — API-1275 数据与协议变更明细（vs visla-v2）

> 调研基线：`visla-api` / `vcjs` 的 **API-1275 分支**（2026-09-03 已按当前分支重新核对 create-videos / scene script 数据链）对比 **visla-v2**。
> 上游主文档：[README.md](./README.md)（三步主流程 / plan-kits vs plan-video / 子 issue 地图）。本文聚焦其后的增量演进：**新增字段管理、AI 推荐数据体系、create-videos 使用方式、业务/数据/协议三类变更总结**。
> 注意：README 中数据模型小节写于早期版本，本文 §2 为最新口径（含 `environment_id` → `ref_image_kit_id` 改名等修正）。

---

## 1. 业务确认（最新链路）

Scenario Dialogue Video（`ProjectTypeEnum.SCENARIO_DIALOGUE_VIDEO(12, "scenario-dialogue-video")`，`supportAigc = true`）在 API-1275 后期演进后的完整闭环：

```text
Step 1  创建项目（fn=scenario-dialogue-video 或 AI detect 候选）
          │
Step 2  AI 编排 aigc_plan_video（2010，一次回调带回四件套）
          │   normalized_video_script + kits[] + coverage_spatial_plan + spatial_units
          │
          ├─ kits 落库：detail.ai_kit_id ← AI 语义 kitId（§4）
          ├─ scene script 落库：AI 语义 id → 内部 kit 主键映射（§4.3）
          ├─ coverage kit 落库：ai_kit_id ← master_coverage_shot.coverage_shot_id（§4.2）
          └─ agent_project_aigc.scene_script_status 状态机守护（§5.3）
          │
Step 2.x 用户编辑（Web 端）edit-script → EDITING → renormalize → PROCESSING → NORMAL/FAILURE
          │
Step 3  create-videos（§5）
          payload = attribute_info + footage/bgm options
                   + kits[]（COMPLETED 且非 COVERAGE）        ← §5.1
                   + normalized_video_script.scenes[]          ← §5.2 协议
                   + coverage_spatial_plan（attributes link）
                   + spatial_units[]（coverage 数据单独通道）
```

**fetch pipeline（OpenAPI）现状**：create-video 前由 fetch 补发 plan 触发；scenario 项目须升级为 `aigcPlanVideo`（API-1286 改动点，见 README §7）。vcjs 侧 `disableUnsupportedAigcBeforeCreateVideo` 对 scenario **不**降级（supportAigc=true）。

---

## 2. 数据变更（vs visla-v2 全量新增）

### 2.1 Migration

| 文件 | 状态 | 内容 |
|------|------|------|
| `v0.3.6.5__scenario_dialogue_video.sql` | **已在 visla-v2** | scenario 基础表（scene_script 建表 + aigc 两列） |
| `v0.3.6.6__scenario_dialogue_video.sql` | **API-1275 独有**（160 行，`d8d0f59bfc` 由 v0.3.6.5 重命名而来，幂等存储过程写法） | 全量 scenario DDL，与 v2 版本差异见下表新列 |

> 排查要点：同名 migration 在两分支内容不同属**正常演进**（API-1275 在原文件上追加列）；发布合并时走 Flyway 版本号顺序，v0.3.6.6 幂等可重复执行。

### 2.2 新增/变更列清单（管理行）

| 表 | 列 | 类型 | 管理方（写入） | 说明 |
| ---- | ---- | ------ | ---------------- | ------ |
| `agent_project_aigc` | `style_detail` | varchar(2000) | vcjs plan 回调 | ai storyboard style detail |
| `agent_project_aigc` | `use_scene_text`（README 旧名 use_text_overlay） | tinyint(1) def 1 | 创建参数 | 图内是否渲染精确文字 |
| `agent_project_aigc` | **`scene_script_status`** | tinyint(1) def 1 | 双仓（vcjs 落库回写 / visla-api Web 编辑流转） | **1 normal / 2 processing / 3 failure / 4 editing**（`8a77ddafb7` 引入，`SceneScriptStatusEnum`） |
| `agent_project_scene_script` | （整表，visla-api+vcjs 双 Entity/Mapper） | — | vcjs `handlePlanVideoResult` 硬删重建；visla-api Web CRUD | 场景脚本，字段见 §2.3 |
| `agent_project_kits` | `spatial_rules` | varchar(512) | plan 回调 | kit 空间规则 |
| `agent_project_kits_detail` | **`ai_kit_id`** | varchar(128) | 双仓落库时 `resolveAiKitId` 写入 | **AI 侧语义 kit id**，AI 推荐数据主键（§4.1） |
| `agent_project_kits_relation` | （整表） | — | coverage shot 复用管理 | `kit_detail_id` / `source_kit_id` / `slot`（**1 起始**，`875b4baef0`）/ `source_kit_modified`（0 normal/1 regenerate/2 delete，`KitModifedEnum`） |

**双仓同步硬规则**：以上每列在 visla-api 与 vcjs 的 Entity + Mapper XML 必须成对存在（本次 API-1286 合并已双向核对）。

### 2.3 `agent_project_scene_script` 字段（最新口径）

| 字段 | 类型 | 说明 |
| ------ | ------ | ------ |
| `ref_image_kit_id` | bigint | 参考图 kit（**由 `environment_id` 改名**，`3f7284b9f`）；指向 `agent_project_kits.id`（人物/场景/Coverage 均可） |
| `speaker_id` | bigint | 说话人 kit → `agent_project_kits.id`（type=1 人物） |
| `audio_script` / `visual` / `scene_text` | text | 台词（TTS 输入）/ 视觉动作描述 / 屏幕文字 JSON |
| `duration` / `sorting` / `status` | — | 时长 / 排序 / 0 created·1 active·2 inactive·99 deleted |

---

## 3. 枚举变更（vs visla-v2）

| 枚举 | 取值 | 用途 |
| ------ | ------ | ------ |
| `SceneScriptStatusEnum`（新） | NORMAL(1) / PROCESSING(2) / FAILURE(3) / EDITING(4) | scene script 生命周期守护（§5.3） |
| `KitsTypeEnum` | 新增 **COVERAGE** | coverage shot 类 kit（区别于普通素材 kit） |
| `KitModifedEnum`（新） | NORMAL(0) / REGENERATE(1) / DELETE(2) | coverage 复用源 kit 修改状态 |
| `KitsSourceTypeEnum`（新） | — | kit 来源分类 |
| `PositionLockEnum` / `SpatialContinuityModeEnum`（新） | none / coverage_locked 等 | 空间连续性控制 |
| `ProjectSpeakingModeEnum`（新） | voiceover / monologue / **dialogue** / group / no_audio | 音频形态 |
| `ProjectTypeEnum` | **SCENARIO_DIALOGUE_VIDEO(12)** supportAigc=true；`de2b84999c` 调整枚举序号/文案 | 项目类型 |
| `VideoCreationTemplateEnum` | scenario 模板项（`de2b84999c`） | 创建模板 |
| `ApiMessage` | 新增 **4609** `AGENT_PROJECT_SCENE_SCRIPT_NOT_COMPLETED`（`48c6a454c4`）等 | create-video 前置校验错误码 |

---

## 4. AI 推荐数据（kits 语义 id 体系）

### 4.1 `ai_kit_id` 生成规则（双仓一致）

`KitsBo.resolveKitId(detail)`（vcjs `11c0170cb` 后 kitIndex null→"0"）：

```text
detail.ai_kit_id 非空  → 直接用（AI 返回的语义 id）
否则                    → "{name 空格转下划线}_{kitIndex ?? 0}"   ← 兜底合成 id
```

写入时机：

- **普通 kit**：plan 回调落库 / Web add-kit 时，若 AI 返回 `kits[].kitId` 非空则直接存入 `detail.ai_kit_id`（`KitsBo.of` 组装链）
- **Coverage kit**（`buildCoverageKitDetail`）：`detail.ai_kit_id = master_coverage_shot.coverage_shot_id`（AI 推荐主图即锚点）
- **出向组装时兜底**：`resolveAiKitId(detail)` 在读路径（`queryActiveKitsAssemblyVoList` / `queryKitsByAgentProjectId`）再次回填，保证旧数据也有语义 id

### 4.2 Coverage shot 推荐链

```text
plan_video 回调 coverage_spatial_plan + spatial_units
    → 每个 spatial unit 生成 COVERAGE 型 kit + detail（ai_kit_id=coverageShotId, processStatus=INIT）
    → master coverage shot 主图生成后 detail → COMPLETED
    → kits_relation 记录复用关系（slot 1..5, source_kit_modified）
    → 失败的 coverage shot：status→INACTIVE（vcjs `11c0170cb` 增强失败处理）
```

### 4.3 Scene script 的 kit 引用映射（落库入向）

`AgentProjectSceneScriptService.getKitId()`（vcjs）：

```text
AI 返回 normalized_video_script.scenes[] 中：
  speaker_kit_id / environment_kit_id / coverage_shot_id  ← AI 语义 id（字符串）
      │
      ├─ "-1"          → 存 -1L（哨兵：显式无 kit）
      ├─ 命中 kitsVoMap → 存内部 kit 主键（agent_project_kits.id）
      └─ 未命中         → null
落库：script.speaker_id / script.ref_image_kit_id
```

### 4.4 出向反解（§5.2 的基础）

create-videos 组装时按 kit 主键反查 detail，`aiKitId` 重新作为协议字段发出——**入向存主键、出向发语义 id**，`ai_kit_id` 是双向翻译层。

---

## 5. create-videos 如何使用

### 5.1 kits payload 过滤规则（双仓对齐，`2602228a7`）

`queryKitsByAgentProjectId`（visla-api `AgentProjectKitsService` ≈ vcjs 同名）：

```java
.filter(processStatus == COMPLETED && type != COVERAGE)
```

- **COVERAGE kit 不进 kits[]**：其数据经 `spatial_units` 单独通道（`querySpatialUnitsByAgentProjectId`），混入会致 AI 校验失败
- **未完成 kit 不发**：缺 standardLink 会破坏素材完整性
- kits 仅在 `footageSearchOptions.useAigc=true` 时携带，否则空列表

### 5.2 normalized_video_script 出向协议（`PlanVideoScriptVo` / NormalizedSceneScript）

`assembleNormalizedVideoScript`（以当前 visla-api `API-1275` 分支实现为权威契约）：

| 字段 | 取值规则 |
| ------ | --------- |
| `scene_index` / `duration` / `visual` / `audio_script` | scene script 表直接透传（visla-api Mapper 按 `sorting` 排序） |
| `scene_text` | 从 `scene_text` JSON 数组反序列化；不按逗号拆分 |
| `environment_kit_id` | refImage kit detail 的 **aiKitId**；无 detail → null |
| `environment_kit_index` | detail.kitIndex；无 detail → **-1**（哨兵）；refImage 为 **COVERAGE 型 → -1**（env 位让给 coverage） |
| `coverage_shot_id` | refImage kit 为 COVERAGE 型时 = 该 kit detail 的 **aiKitId**（即 coverageShotId）；否则不设置（NON_NULL 省略） |
| `speaker_kit_id` / `speaker_kit_index` | 同 environment 口径：aiKitId / kitIndex，无 detail → null / **-1** |
| `visible_kit_indexes` | `[environment_kit_index, speaker_kit_index]` 去重后 **过滤 `> 0`**（排除 -1 哨兵） |

```text
 title/summary ← groupScript（无则 project 名/描述）
```

> ✅ **双仓实现一致（当前分支）**：两侧对缺失 kit 使用 `-1` 哨兵；Coverage 场景出向 `coverage_shot_id=coverage detail.ai_kit_id`、`environment_kit_id=null`、`environment_kit_index=-1`，并从 `visible_kit_indexes` 过滤 `<=0`。vcjs 不再把 Coverage kit 转成关联 Environment kit。

### 5.3 前置校验（新增）

| 校验 | 触发点 | 行为 |
| ------ | -------- | ------ |
| **scene_script_status 守护**（`48c6a454c4`） | visla-api `AgentProjectService` createVideos 入口 | `aigc.sceneScriptStatus != NORMAL` → 抛 **4609** `AGENT_PROJECT_SCENE_SCRIPT_NOT_COMPLETED` |
| **coverage shot regenerate 检查**（`cb81a206eb`） | `AgentProjectKitsService`（kit 完整性检查） | COVERAGE kit 的 relation 集合中 `source_kit_modified` 最大值 != NORMAL → 视为 incomplete，阻断 create |
| create-video 时长校验（hot-fix `bcf30ecd75`） | attribute 组装 | 上限钳制（1200s） |
| regenerate relation 幂等（`a258d180b6`） | kits relation 写入 | 修复重复 relation id |

### 5.4 状态机：scene_script_status 流转

```text
                    Web edit-script / add / delete
  NORMAL(1) ────────────────────► EDITING(4)
     ▲                                │ renormalize 发起
     │ 回调成功                        ▼
  FAILURE(3) ◄────────────────── PROCESSING(2)
     │  用户重试 re-normalize           │
     └────────────────────────────► PROCESSING(2)
```

- 编辑类操作（`AgentProjectSceneScriptService` 三处）先置 **EDITING**
- `renormalizeVideoScript`：PROCESSING 中拒绝重复发起（L418 守护）→ 置 PROCESSING → 回调成功回 NORMAL / 失败 FAILURE
- vcjs plan 回调链同样回写（`AgentProjectAigcService.updateSceneScriptStatus`，EDITING 仅允许从 NORMAL 进入）

### 5.5 scene_script 与成片侧的关联机制（核心：project_recommend_candidate 固态链）

**一句话业务**：plan-video 回调把 AI 编排结果落 agent_project 层（scene_script + kits 体系）；createVideos 把 scene_script 值拷贝成出向协议（语义 kit id）发给 AI；**AI 回调后逐 scene 创建 project_scene，同时把该 scene 的 storyboard/motionVideo 规划数据以 `ai_storyboard_metadata` JSON 固化进 `project_recommend_candidate`（按 `scene_id` 固态关联）**；此后一切生成（storyboard 图、motion video、generate-motion-video 的默认值）一律以 candidate metadata + `ai_generated_content_rel.scene_id` 驱动，**与 scene_script 再无交互——scene_script 只是首轮导演稿的输入侧快照**。

#### 5.5.1 四步链路（接收→存储→出向→回调）

**① 接收（plan-video 回调 `AigcPlanVideoResBo` 四件套，全是 AI 语义 id 非库主键）**：
`normalized_video_script`（scenes[]：speaker_kit_id / environment_kit_id / coverage_shot_id / audio_script / visual / duration / scene_text）+ `kits[]`（语义 kitId + 形象描述 + seed + voiceProfile）+ `coverage_spatial_plan` + `spatial_units`

**② 存储（agent_project 层，一次性映射）**：
scene_script 落库时语义 id 映射为内部 kit 主键（`getKitId()`："-1"→-1L，命中→主键，未命中→null）；`agent_project_kits_detail.ai_kit_id` 保留语义 id。**这一层与下层无任何外键，映射是脆弱的一次性快照**。

**③ create-videos 出向（DB 现查现拼，无状态回写）**：
`assembleNormalizedVideoScript` 查 scene_script 表（按 sorting）→ 反查 kit detail 拿 aiKitId → scenes[]；kits 查 `queryKitsByAgentProjectId`（COMPLETED 且非 COVERAGE）；coverage 走 attributes link + spatial_units。

**④ create-videos cb（scene + aigc 落库，固态关联在此建立）**：
`TextToVideoProjectRecommendServiceImpl.handleProjectRecommend` 逐 sceneBo（scenario → creationTemplate=TEXT_TO_VIDEO + DIRECT_SCRIPT，`174755866` 与 SCRIPT_TO_VIDEO 同 case）：

- 创建 `project_scene`（transcript=body_text，intro/outro 包夹）+ timeline
- **`constructCandidate(projectId, sceneId, timelineId, candidates, sceneBo.getAiStoryboardMetadata())` → 固化 candidate**（`RecommendService#handleCandidates`）
- `assembleAiStoryboardBo(project, scene, metadata, kit2AssetMap)`：storyboard 首图生成 → `ai_generated_content` + `rel`（scene_id、scene_selected=true）
- motion video **不在此时生成**——candidate 里的 motionVideo 规划等后续触发

#### 5.5.2 固态关联的真正载体：project_recommend_candidate（存量表，v0.2.3.7 加 scene_id）

```text
project_recommend_candidate
  ├─ scene_id    → project_scene.id      ← scene 级固态关联（成片侧唯一可靠锚点）
  ├─ timeline_id → project_timeline.id
  ├─ candidates JSON（broll 推荐候选）
  └─ ai_storyboard_metadata JSON（该 scene 完整生成规划：
        storyboardImage{stillPrompt, refImageList[]} +
        motionVideo{motionPrompt, mode, promptJson, keyframes, speechPlan}）
```

后续生成全部以此为准：

- storyboard 图：candidate metadata + kit2AssetMap 参考图
- motion video：scene-aigc-info 规划态（processStatus=NONE）= 把 candidate 的 motionVideo 规划返回给调用方，调用方显式回传 generate-motion-video（**generate 不做隐式继承**，OpenApiProjectService L2801 `getCandidateMotionVideo` 按 projectId+sceneId 读 candidate）
- 回查：rel(selected) → aigc（projectKitsDetailId / speechPlan.speakerKitIndex）→ project_kits_detail → scene.kit

**add-scene / regenerate / generate-motion-video 根本不经过 scene_script**：直接对 `project_scene.id` 操作，参数来自 candidate metadata 或当次请求入参，结果落 aigc + rel(scene_id)。**scene_script 体系只存在于首轮 AI 导演编排（Step 2→3 一次性快照），成片后的世界只有 scene_id + candidate + rel。**

#### 5.5.3 辅助：kit 语义链（编排层→编辑层的 kit 投射）

```text
scene_script.speaker_id / ref_image_kit_id → agent_project_kits_detail（kit_index + ai_kit_id）
    │ createVideos 出向：speaker_kit_id(=ai_kit_id) / speaker_kit_index(=kit_index)
    │ AI 回调 saveProjectKitsDetail（`803a64d1a`）：agentKitDetailByIndex.get(bo.getIndex())
    │      ← 对齐键 = kitIndex
    ▼
project_kits_detail（编辑层新 ID，同语义）+ copyAgentProjectKitVoice 声音绑定 + kit2AssetMap
    │ storyboard：AigcImageBo.refImageList[]（AigcKitBo{kitIndex,assetId,slot}）按 kitIndex 取参考图
    │ motion：speechPlan.speakerKitIndex → ProjectSceneKitVoice → aigc.project_kits_detail_id（表级直连）
    ▼
同一语义 kit 的两个投影（agent 侧 ↔ aigc 侧），靠 kit_index/ai_kit_id 对齐，无 ID 直连
```

#### 5.5.4 要点

| 维度 | 说明 |
| ---- | ---- |
| **固态关联** | `project_recommend_candidate.scene_id`（生成规划载体）+ `ai_generated_content_rel.scene_id`（生成结果载体）→ project_scene.id；**均与 scene_script 无外键** |
| scene_script 定位 | 首轮导演稿输入侧快照；createVideos 时刻被固化；之后编辑（re-normalize）需重新 createVideos 才反映到成片 |
| 新增场景 | add-scene 类操作无 scene_script，直接走 scene_id + candidate/rel 链，参数来自当次请求或 candidate metadata |
| kit 关联 | 语义键对齐：agent kit 与 aigc 侧 kit 标记（project_kits_detail_id 直连列 / metadata.kitIndex / speechPlan.speakerKitIndex）是同一 kit 的两个投影 |
| 顺序风险 | AI storyboard 阶段可能 merge/split scenes（tryMergeScene），index 对齐会漂移——回查一律走 rel.scene_id，不按 index |
| rel 语义 | 多次生成多条 rel，仅 **scene_selected=true** 是当前版本（`a258d180b6` 修复后 regenerate 不再重复 relation） |

### 5.6 normalize / create-videos 字段级明细（重点新增字段）

#### 5.6.1 scene-normalize 入向字段（plan-video 回调 `normalized_video_script.scenes[]`，AI 语义 id）

| 字段 | 语义 | 使用（落库映射 `buildSceneScript`） |
| ---- | ---- | ---- |
| `scene_index` | 场景序号（0 起） | → `scene_script.sorting = scene_index + 1`（DB 侧 1 起排序） |
| `duration` | 目标时长（秒，Double） | → `scene_script.duration`；storyboard/motion 时长基准 |
| `speaker_kit_id` | 说话人 kit 语义 id（字符串） | `getKitId()` 映射：命中 kits → `speaker_id`（内部主键）；`"-1"` → -1L；未命中 → null |
| `environment_kit_id` | 环境/场景 kit 语义 id | 与 `coverage_shot_id` **二选一优先**映射 → `ref_image_kit_id` |
| `coverage_shot_id` | coverage 锁定锚（spatial planning 后 AI 填充；未锁定 null） | 优先于 environment 映射 → Coverage kit 主键并落 `ref_image_kit_id`；锚点语义保存在 Coverage kit `detail.ai_kit_id`，不落独立 scene_script 列 |
| `audio_script` | 台词（TTS 输入） | → `audio_script`（2000 字截断保护） |
| `visual` | 视觉动作/走位/运镜描述 | → `visual`（2000 保护）→ storyboard prompt 素材 |
| `scene_text` | 图内精确文字（字符串数组） | `JsonUtil.asJson` → `scene_text`（JSON 存储） |

管理列（`agent_project_aigc` 新增，控制 normalize 生命周期）：

| 列 | 语义 | 使用 |
| -- | ---- | ---- |
| `scene_script_status` | 1 normal / 2 processing / 3 failure / 4 editing | renormalize 守护（PROCESSING 中拒绝）；edit/add/delete 置 EDITING；**createVideos 前置校验 ≠NORMAL → 4609** |
| `use_scene_text` | 是否把 scene_text 渲染进图（默认 1） | createVideos 出向渲染开关 |
| `style_detail` | storyboard 风格描述（2000） | storyboard 生成风格约束 |

renormalize（Web 再规范化）：请求发回 AI 的 payload（`RenormalizeScriptReqBo.Data`）= `target_video_properties` + `kits` + **`edited_normalized_video_script`**（用户编辑后的 DB scene_script 重新组装回 PlanVideoVo 发给 AI 重排）；回调后 scene_script **硬删重建**，status PROCESSING → NORMAL/FAILURE。

#### 5.6.2 create-videos 出向字段（scenes[]，组装规则 vs 入向的关键差异）

| 字段 | 语义 | 如何算出 |
| ---- | ---- | ---- |
| `speaker_kit_id` | 语义 id（非数字主键，契约 "Same as UnifiedKitItem.kit_id"） | `detail.aiKitId`；无 detail → null |
| `speaker_kit_index` | kit 序号，**缺失时 -1 哨兵** | `detail.kitIndex` / -1 |
| `environment_kit_id` | 环境 kit 语义 id | 非 Coverage refImage → `detail.aiKitId`；缺失 / Coverage 场景 → null |
| `environment_kit_index` | 环境 kit 序号 | 非 Coverage refImage → `detail.kitIndex`；缺失 / Coverage 场景 → **-1 哨兵** |
| `coverage_shot_id` | coverage 锁定标记 | refImage 为 COVERAGE 型时 = Coverage `detail.aiKitId`（即 `master_coverage_shot.coverage_shot_id`）；非 Coverage / 缺失场景**省略**（NON_NULL） |
| `visible_kit_indexes` | 画面内可见 kit 序号集（AI 据此决定画面出现哪些角色/场景） | `[envIdx, spkIdx]` 去重后**过滤 >0**（排除 -1 哨兵） |
| `scene_text` | 图内文字数组 | 从 `scene_text` JSON 数组反序列化；不按逗号拆分 |

#### 5.6.3 kits[] 出向字段（KitsBo 新增重点）

| 字段 | 语义 | 使用 |
| ---- | ---- | ---- |
| `kit_id` | 语义 id（`ai_kit_id` 优先，兜底 `name_kitIndex`） | 与 scenes[].speaker_kit_id 同键关联——AI 侧靠它把 scene 绑到 kit |
| `index` | kitIndex | 与 `speaker_kit_index` / `visible_kit_indexes` / `spatial_units.environment_kit_index` 的**对齐键** |
| `speech_capability` | `speaker_eligible` / `silent_visual_only` / UNDEFINED（新增枚举） | **AI 据此决定该 kit 能否被选为 speaker** |
| `voice_profile` | TTS 音色描述（plan 时 AI 推荐） | 配音选择 |
| `kit_source.seed` | 形象生成种子 | 跨场景形象一致性 |
| 发送条件 | `useAigc = !FALSE.equals(aigc.aiStoryboardEnable)`；过滤 `COMPLETED && type≠COVERAGE` | useAigc=false → kits 空数组 |

#### 5.6.4 spatial/coverage 独立通道（不进 kits[]）

| 字段 | 语义 |
| ---- | ---- |
| `coverage_spatial_plan` | spatial 规划（attributes link → S3 JSON） |
| `spatial_units[].spatial_unit_id` | 空间单元 id（与 COVERAGE kit 的 `ai_kit_id` 同源） |
| `spatial_units[].environment_kit_index` | 该单元的环境 kit——对齐键指向 `kits[].index` |
| `spatial_units[].camera_axis_rule` / `anchors` | 机位轴规则 / 演员位槽（PositionLock）——机位一致性 |
| `master_coverage_shot.{coverage_shot_id, ref_images, still_prompt, position_lock}` | 主 coverage 图锚 id / 参考图 / 生成 prompt / 构图锁 |
| `attribute_info.requires_spatial_continuity` | detect 时 targetVideo 给出（AI Director → true，否则 false）——AI 侧空间连续性路由开关 |

#### 5.6.5 字段驱动的前置校验

- `scene_script_status ≠ NORMAL` → **4609** `AGENT_PROJECT_SCENE_SCRIPT_NOT_COMPLETED`
- COVERAGE kit 的 relation `source_kit_modified ≠ NORMAL` → 视为 incomplete 阻断 create

---

## 6. 协议变更汇总（对外）

| 协议面 | 变更 | 影响 |
| -------- | ------ | ------ |
| **NormalizedSceneScript（AI 网关 create-videos 出向）** | `speaker_kit_id` / `environment_kit_id` 全量切语义 aiKitId（非数字主键）；index 引入 **-1 哨兵**；新增 `coverage_shot_id`、`visible_kit_indexes`；`scene_text` 数组化 | AI 侧按 kit_id 关联 kits[]；-1=显式无 kit |
| **OpenAPI `scene-aigc-info`（`OpenSceneAigcStatusVo`）** | 新增 scene 级 `kit`（kit_uuid/kit_index/name/role/…含 voice 元数据）；`storyboard`/`motionVideo` 带 `selected`（多 rel 时仅 selected 为当前版）+ `delivery_mode`（on_camera_speaker/off_screen_vo/silent）+ `refImages`；motionVideo 未生成时**无虚拟占位**（absent = not generated，规划态返回 NONE+candidate metadata） | 第三方按 selected 读当前版；直接回传规划态数据即可显式生成 |
| **Web v3 `TeamspaceAgentProjectSceneScriptController`**（+150 行） | list/add/edit/delete-script 四端点 + renormalize | UE-1368 编辑入口 |
| **OpenAPI generate-video / generate-aigc-video** | scene script 未 NORMAL → **4609**（新错误码） | 调用方需处理该错误分支（openapi-test 已加回归 case） |
| **`OpenApiSceneVideoGenerateReqBody`** | `kit_voice_map`（按 kit 覆盖 voice）、scene 级 `kit_uuid` / `ref_images` / `motion_video_mode` / `enable_audio` | scene 级显式生成入参 |

---

## 7. 变更总结（业务 / 数据 / 协议）

### 业务

1. scenario-dialogue-video 从 Web 端专属走向全链路（创建校验 → plan-video 编排 → 编辑守护 → create-videos 校验）双仓闭环
2. kit 体系升级为「**AI 语义 id（ai_kit_id）为主键翻译层**」：入向映射存主键、出向发语义 id
3. coverage shot 成为独立 kit 类型 + 独立数据通道（spatial_units），与普通素材 kit 分流
4. scene script 获得完整状态机（normal/processing/failure/editing），create-video 前置守护

### 数据

1. 新表 ×2：`agent_project_scene_script`、`agent_project_kits_relation`
2. 新列：`aigc.scene_script_status` / `style_detail` / `use_scene_text`、`kits.spatial_rules`、`kits_detail.ai_kit_id`
3. 改名：`scene_script.environment_id` → `ref_image_kit_id`
4. Migration：v0.3.6.6（API-1275 独有，160 行幂等）；v2 已含 v0.3.6.5 旧版

### 协议

1. NormalizedSceneScript：语义 kit id + -1 哨兵 + coverage_shot_id + visible_kit_indexes（§5.2 表）
2. scene-aigc-info：scene 级 kit + selected/delivery_mode/absent 语义（§6）
3. 新错误码 4609；`kit_voice_map` / scene 级生成入参
4. coverage / 未完成 kit 从 create-videos kits[] 剥离（spatial_units 通道）

---

## 8. 双仓同步状态与风险

| 项 | 状态 | 动作 |
| ---- | ------ | ------ |
| DB 列 / Entity / Mapper | ✅ 双仓同步（本次 API-1286 合并已核对） | — |
| `resolveAiKitId`（kitIndex null→"0"） | ✅ 双仓一致（vcjs `11c0170cb`） | — |
| `toPlanVideoScriptVo` Coverage / 哨兵结构 | ✅ **双仓一致**；vcjs 已补 Coverage、kit_index=-1、coverage_shot_id、visible_kit_indexes 回归断言 | 后续 AI 网关联调用例仍需覆盖 Coverage 场景 |
| create-videos 前置校验（4609） | ✅ visla-api 已上；vcjs 侧由 fetch pipeline StageGuard 兜底 | 回归 case：`aigc_create_video_scene_script_guard` |

## 9. 相关代码索引

| 域 | visla-api | vcjs |
| ---- | ----------- | ------ |
| scene script 组装 | `internal/ai/service/AiAgentService#assembleNormalizedVideoScript` / `toPlanVideoScriptVo` | 同名类同名方法（L951/L975） |
| kits 出向过滤 | `AgentProjectKitsService#queryKitsByAgentProjectId` | 同名（+COMPLETED/非COVERAGE 过滤对齐） |
| ai_kit_id 解析 | `bo/KitsBo#resolveKitId`（静态） | 同名 + `AgentProjectKitsService#resolveAiKitId` |
| plan 回调落库 | — | `AiAgentCreatorCbService#handlePlanVideoResult`（L426）+ `AgentProjectSceneScriptService`（getKitId 映射 L250 附近） |
| coverage kit 构建 | — | `AgentProjectKitsService#buildCoverageKitDetail`（L349） |
| 状态机流转 | `AgentProjectSceneScriptService`（EDITING/PROCESSING）+ `AgentProjectAigcService#updateSceneScriptStatus` | 同 |
| create 校验 | `AgentProjectService` L1712（4609）+ `AgentProjectKitsService`（coverage shot modified 检查） | fetch 侧 StageGuard |
