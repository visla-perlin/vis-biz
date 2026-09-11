# Scenario Dialogue Video 业务域 — 多角色情景对话视频

> 跨 visla-api / vcjs 共用的**业务规则 / 契约**文档。
> Linear 父 issue：[AI-1365](https://linear.app/visla/issue/AI-1365/ai-director-mode-scenario-dialogue-video)（AI Director Mode: Scenario Dialogue Video）
> 本文基于 **API-1275 分支**的双仓审查：visla-api `25c58e3e9f`、vcjs `4b2590cf5d`，于 2026-09-03 对比各自 `origin/visla-v2`。该分支已包含 API-1286 的 OpenAPI 适配提交。
> **2026-09-03 更新**：API-1275 后续演进（`ai_kit_id` 语义体系 / `scene_script_status` 状态机 / coverage shot 独立通道 / NormalizedSceneScript -1 哨兵协议）及当前发布前置条件见 **[02-api1275-data-and-protocol.md](./02-api1275-data-and-protocol.md)**。

---

## 1. 业务定位

Scenario Dialogue Video（情景对话视频）是 AIGC「AI Director Mode」下的新项目类型：

- 用户给出一段**情景描述 / 剧本素材**（客服培训、角色扮演、合规演练、广告脚本等），AI 自动产出**多角色**情景视频。
- 角色拥有独立的 **Kit**（人物 / 场景 / 道具形象），跨场景保持一致性。
- 对白（dialogue）由不同 speaker 分别 TTS，画面按场景脚本（scene script）分镜生成。
- 整条链路复用 AIGC pipeline：storyboard（分镜图）→ motion video（动态视频），与现有 AIGC 项目共用状态机与轮询调度。

**项目类型**：`ProjectTypeEnum.SCENARIO_DIALOGUE_VIDEO(12, "scenario-dialogue-video")`，`supportAigc = true`（API-1275）。

## 2. 三步主流程（AI Director Mode，实际实现）

```text
Step 1  创建项目
        └─ projectType = scenario-dialogue-video（用户指定 或 AI detect 出候选）

Step 2  AI 编排 —— 对外是【一个】网关消息 AIGC_PLAN_VIDEO(2010, "aigc_plan_video")
        AI 侧内部：normalize_script → plan_kits → plan_spatial
                          │
                          ▼ 一次回调返回（AigcPlanVideoResBo）
        normalized_video_script + kits[] + coverage_spatial_plan + spatial_units
             │                    │              │                    │
             ▼                    ▼              ▼                    ▼
        scene script 表       kits 体系     attributes link     kits_relation
        （vcjs handlePlanVideoResult 统一落库，硬删重建）

        ├─ Step 2.1（UE-1368）：normalized script 展示 + 用户编辑 + re-normalize
        ├─ Step 2.2（AI-1366）：为空间连续性生成 coverage shot
        └─ Step 2.3（AI-1376）：更新 spatial coverage images

Step 3  Create Video（AI-1367）
        └─ createVideos payload 携带 normalized_video_script + kits
           + coverage_spatial_plan + spatial_units → 生成 scene/timeline
```

### 2.1 plan-kits vs plan-video（两个不同的 AI 网关消息）

| | `aigcPlanKits`（AIGC_PLAN_KITS 2005，旧） | `aigcPlanVideo`（AIGC_PLAN_VIDEO 2010，新） |
| --- | --- | --- |
| 请求 | targetVideo + assets + brand | 同左 **+ functionName** |
| AI 产出 | 仅 `kits[]` | `kits[]` + `normalized_video_script` + `coverage_spatial_plan` + `spatial_units` |
| 回调落库 | `handlePlanKitsResult`：save kits | `handlePlanVideoResult`：save kits + **scene scripts**（speaker/environment 名 → kit id 映射）+ spatial link + spatial units |
| 阶段状态 | 均回写 `PLAN_KITS` 阶段（markCompleted/markFailed），fetch 轮询兼容 | 同左 |
| 当前调用方 | 非 scenario 的兼容路径 | Web 端，以及 OpenAPI fetch 的 scenario 分支 |

> 含义：plan-video 是 plan-kit 的超集。scenario 的场景脚本必须由 plan-video 产出；
> 当前 OpenAPI fetch 在 scenario 分支已改发 plan-video，不能再将该路径回退到 plan-kits。

### 2.2 触发时机（谁是发起方）

| 场景 | 触发点 |
| ------ | -------- |
| Web + AI Director，detect 回调确认单一 function | `handleAiCreatorResult`：storyBoardEnable 且非 fetch 项目 → 立即 `aigcPlanVideo`，成功后 `updateToProjectType` |
| Web + AI Director，change-type 后 extract 回调 | `handleAiCreatorExtractParameterResult`：storyBoardEnable 且非 fetch 项目 → `aigcPlanVideo`，之后 `updateToPrepare` |
| **OpenAPI（fetch tracking 管控）** | 两处回调都跳过（`isUnderFetchTracking` / `isFetchPipeline` 守卫）；由 fetch 在 create-video 前补发，scenario 分支发 `aigcPlanVideo` |

- **Step 3 的 `normalized_video_script`**（visla-api `AiAgentCreateVideosBo`）只是把 DB 里的 scene script 透传给 AI（用户编辑后生效），不是新的一次 AI 编排。
- **re-normalize**：用户改 script 后点击，入参为 `script + property`，AI 返回新的 normalized_script。仅 Web 端（Step 2.1）提供。

## 3. 数据模型（API-1275）

```text
agent_project（项目）
  ├─ agent_project_scene_script     场景级脚本（Step 2 产物，N 条）
  │    ├─ speaker_id        → agent_project_kits.id (type=1 人物)
  │    ├─ ref_image_kit_id  → agent_project_kits.id（原 environment_id 已改名；场景 / Coverage kit 均可。Coverage 锚点语义不落 scene_script 独立列）
  │    ├─ audio_script    台词（TTS 输入）
  │    ├─ visual          视觉动作/走位/运镜描述
  │    ├─ scene_text      屏幕文字（json）
  │    ├─ duration        目标时长
  │    └─ sorting/status  排序与状态（0 created / 1 active / 2 inactive / 99 deleted）
  │
  ├─ agent_project_kits（既有表 + 新列）
  │    └─ spatial_rules   kit 的空间规则（varchar 512）
  │
  ├─ agent_project_kits_detail（既有表 + 新列）
  │    └─ ai_kit_id       AI 侧语义 kit id（varchar 128，演进新增；出入向双向翻译层）
  │
  ├─ agent_project_kits_relation（新表）
  │    ├─ kit_detail_id        kit detail ID
  │    ├─ source_kit_id        源 kit（coverage shot 复用来源）
  │    ├─ slot                 参考图槽位（1 起始）
  │    └─ source_kit_modified  0 normal / 1 regenerate / 2 delete
  │
  ├─ agent_project_aigc（既有表 + 新列）
  │    ├─ style_detail         ai storyboard style detail（varchar 2000）
  │    ├─ use_scene_text       是否在图内渲染精确文字（默认 1）
  │    └─ scene_script_status  scene script 状态机（1 normal / 2 processing / 3 failure / 4 editing，演进新增）
  │
  └─ agent_project_attributes（既有表）
       └─ potential_function_names  detect-function 候选（含 scenario-dialogue-video）
```

**migration**：API-1275 当前使用 `v0.3.6.7__scenario_dialogue_video.sql`（160 行幂等存储过程写法，新增 `scene_script_status` / `ai_kit_id` / `kits_relation` 等列，由 visla-api 负责）。`agent_project_scene_script` 不需要独立 `coverage_shot_id` 列：入向 Coverage 锚点映射为 `ref_image_kit_id`，锚点语义保存在 Coverage kit 的 `detail.ai_kit_id`，create-videos 出向再从该字段还原 `coverage_shot_id`。列级明细见 02 文档 §2。

## 4. 关键枚举（API-1275）

| 枚举 | 取值 | 用途 |
| ------ | ------ | ------ |
| `ProjectSpeakingModeEnum` | voiceover / monologue / **dialogue** / group / no_audio | 项目说话模式（剧本的音频形态） |
| `SpatialContinuityModeEnum` | none / coverage_locked | 空间连续性（锁定 coverage shot 保证机位一致） |
| `SceneScriptStatusEnum` | NORMAL(1) / PROCESSING(2) / FAILURE(3) / EDITING(4) | scene script 生命周期守护（create-video 前置校验，错误码 4609） |
| `KitsTypeEnum` | 新增 **COVERAGE** | coverage shot 类 kit（数据走 spatial_units 独立通道，不进 kits[]） |
| `KitModifedEnum` | normal / regenerate / delete | kits_relation 中源 kit 的修改状态 |
| `PositionLockEnum` | （见 API-1275） | 构图/走位锁定 |
| `ProjectTypeEnum` | `SCENARIO_DIALOGUE_VIDEO(12, "scenario-dialogue-video")` supportAigc=true | 项目类型 |

## 5. 与整个 Project Generation 的整合

Scenario Dialogue **不是独立 pipeline**，而是挂接在现有 project generation 全链路上：

| 阶段 | 复用点 | scenario 特有 |
| ------ | -------- | -------------- |
| 创建项目 | `addAndInitAgentProject` / OpenAPI `addBasicVideoAigc` | projectType 可为 scenario-dialogue-video |
| detect-function | `potentialFunctionNames` 候选机制 | AI 可将 scenario-dialogue-video 放入候选 |
| change-project-type / extract | 既有 CHANGE_PROJECT_TYPE/EXTRACT 阶段 | 无 |
| 前置异步 | PLAN_KITS 阶段：`needWaitForAigcPlanKits` 轮询 + fetch 补发；visual analysis | Web 端和 OpenAPI scenario fetch 均用 `aigcPlanVideo`（一次产 kits+scene script+spatial） |
| create-video | `createVideos`（AIGC 分支：storyboard → motion） | payload 附加 normalized_video_script / kits / coverage_spatial_plan / spatial_units |
| 轮询/导出 | POLL_STORYBOARD_SCENES_COMPLETION → TRIGGER_SCENE_MOTION → webhook / export-to-clip | 无 |

**含义**：

1. `supportAigc=true` 使 scenario 项目天然走 AIGC storyboard/motion 轮询，无需新 next-action。
2. vcjs 侧对「不支持的 AIGC 类型」有降级（`disableUnsupportedAigcBeforeCreateVideo`）；scenario 因 supportAigc=true **不会**被降级。
3. OpenAPI 已接入：**创建入参 + 创建时校验**（`function_name=scenario-dialogue-video` 显式指定，要求 AIGC 开启且 idea/script/pdf/webpage 至少其一，直连 GET_PARAMETER 不走 detect）、scenario fetch 的 `aigcPlanVideo` 触发、`scene-aigc-info` 查询，以及手动/自动 motion 的完整参数透传。

## 6. Web 端接口（API-1275，对照用）

`TeamspaceAgentProjectSceneScriptController`（teamspace agent project 下）：

| 端点 | 用途 |
| ------ | ------ |
| `GET /list-script` | 列出场景脚本 |
| `POST /add-script` | 新增场景脚本 |
| `PUT /{scriptUuid}/edit-script` | 编辑场景脚本（触发后续更新） |
| `DELETE /{scriptUuid}/delete-script` | 删除场景脚本 |

配套 service：`AgentProjectSceneScriptService`（408 行）、`AgentProjectKitsRelationService`（250 行）、`AgentProjectKitsService`（+145 行）。

## 7. OpenAPI 支持边界（API-1286，一期）

| 能力 | 一期 | 说明 |
| ------ | :---: | ------ |
| 自动生成 scenario script（Step2 planVideo + Step3 全自动） | ✅ | fn=scenario-dialogue-video 显式指定 + 创建时强校验（aigc 开启、idea/script/pdf/webpage 至少其一）；直连 GET_PARAMETER，不走 detect |
| AI 判断素材是否适合 scenario（detect-first） | ❌ 已废弃 | v2/v3 方案；一期用创建时强校验 + 运行期 AI 失败兜底替代 |
| 用户编辑 normalized script（re-normalize） | ❌ | Web 端 Step 2.1，OpenAPI 后续开放 |
| Kit 绑定 / 修改 / coverage shot 手动更新 | ❌ | 后续开放 |

OpenAPI 侧详细设计：[`IdeaProjects/API-1286/02-openapi-support-plan.md`](../../API-1286/02-openapi-support-plan.md)

## 8. 子 issue 地图（AI-1365）

| Issue | 内容 |
| ------- | ------ |
| AI-1362 | 新项目类型 Scenario Dialogue Video（已完成于 API-1275） |
| API-1275 | Step 2 编排 + script 编辑（Web） |
| AI-1366 / AI-1376 | Step 2.2 / 2.3 coverage shot |
| UE-1368 | Step 2.1 UI 展示与编辑 |
| AI-1367 | Step 3 create video 支持 scenario workflow |
| **API-1286** | **OpenAPI changes（本计划）** |
| WEB-990 | 第三方 App / 插件支持（canva/adobe/google generate-video-v2） |
