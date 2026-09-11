# Scenario Dialogue × OpenAPI：素材/voice 声明与剧本直传适配方案

> 状态：**方案设计（未实施）**。基于 visla-api-v2-pjson / vcjs-v2-pjson 分支代码核实（2026-09）。
> 上游文档：[README.md](./README.md)（业务全景）、[02-api1275-data-and-protocol.md](./02-api1275-data-and-protocol.md)（数据与协议）。
> 本文回答三个问题：① 用户上传的 asset 如何与 kit/voice 保持关联；② 剧本 normalize 有无独立接口、能否用户自定义；③ 「上传 scene/asset/voice + 剧本 → 直接生成对话视频」的可行路径。

---

## 1. 目标链路（用户诉求）

```text
用户准备：
  character 参考图（asset） × N   → 期望成为 kit(character)，并【指定 voice】
  environment/scene 参考图 × M    → 期望成为 kit(environment)
  剧本文本（script）              → 期望按剧本分场景生成对话视频
现状 OpenAPI（API-1286 一期）：
  仅支持「素材喂给 AI + AI 全自动编排」，用户无法指定 asset→kit 归属与 voice。
```

---

## 2. 现状事实（代码核实）

### 2.1 asset → kit 关联：链路存在，契约不强制

| 环节 | 事实 |
| ---- | ---- |
| 入向 | plan 请求携带 `assets: List<AiAgentAssetInfoBo>`（`AigcPlanVideoReqBo`）：`assetId / assetType(IMAGE/VIDEO/…)/ assetUrl / usage / containsSpeech / metadata`。**用户上传的 project media 全量发给 AI，但没有「这是角色参考/场景参考」的声明位** |
| 出向 | AI 回调 kit 携带 `AigcPlanKitsVo.KitSource{mode, assetUrl, assetId, previousAssetUrl, seed}` —— **协议上存在 assetId 回传通道** |
| 落库 | vcjs `AgentProjectKitsService` 保存 kit detail：`sourceType == USER` 时读 `kitSource.assetId` → `agentProjectMediaService.getById(assetId)`：**命中 → `buildRef(mediaId, AGENT_KIT)` 并复用原 media（关联保留）**；未命中 → 走 `handleUrl` 用 assetUrl 新建 media（**与原 asset 失联**） |

**结论（对应明确点 1）**：「AI 分析后 kit 与 asset 失联」不是表结构缺失（`agent_project_kits_detail.mediaId` 一直在），而是两个契约缺口：

1. AI 对 USER 素材**未被强制**必须回传 `asset_id`（回传 asset_url 即失联）；
2. 用户**没有声明位**表达「这个 media 是 character / environment，绑定 voice X」——AI 只能靠猜，猜完命名/数量与用户意图无约束关系。

### 2.2 voice 关联：有自动分配与 web 改绑，无用户指定入口

| 环节 | 事实 |
| ---- | ---- |
| plan 请求 | **不携带 voice 候选**（`AigcPlanKitsReqBo` / `AigcPlanVideoReqBo` 无 voice 字段）——AI 不选音色 |
| 回调分配 | vcjs `applyVoiceSelection`：kit 自带 `voiceId` 且归属当前用户、未被占用 → 采纳；否则按 kit gender 自动分配（`selectVoiceForKit`）；无匹配 → 0。仅 CHARACTER 消耗 voice |
| web 改绑 | kit `edit-kit` 请求体 `voiceUuid` / `ChangeProjectKitVoiceReqBody`：`voiceCloneService.getSyntheticVoiceMetadata` 校验归属后落 `detail.voiceId` |
| OpenAPI | 已有 `GET /openapi/v1/workspace/list-voice?localeName=`（`OpenApiVoiceService`，系统 + 自定义 voice，返回 `OpenVoiceListVo`）——**voice_uuid 可查，但无 kit 接口可消费它** |

**结论（对应明确点 2 前半）**：voice 指定的能力件齐全（查询 + 落库字段 + 校验逻辑），缺的只是 OpenAPI 入口与「声明 → 回调应用」的桥。

### 2.3 normalize：无独立创建期接口，用户不能自定义结构化剧本

| 环节 | 事实 |
| ---- | ---- |
| 创建期 | normalize_script 是 `aigc_plan_video`（2010）网关消息的**内部第一步**，与 plan_kits / plan_spatial 一次回调返回。**没有独立 normalize 接口** |
| 编辑期 | Web 有 `POST /v3/agent-project/{uuid}/scene-script/renormalize-script`：payload = `RenormalizeScriptReqBo{extractParam, kits, spatialUnits, planVideoVo(含 edited script)}` → AI 重排 → scene scripts **硬删重建** → status NORMAL |
| 自定义 | 入参不支持直接给结构化剧本；Web 也只能「编辑 AI 产物」。但 renormalize 通道的 `edited_normalized_video_script` 事实可承载**任意用户剧本**（AI 按其重排对齐 kits） |

**结论（对应明确点 2 后半）**：「character ↔ voice ↔ 剧本」的关联要落地，需要显式的**声明层**（谁是谁、用什么声音）+ **回调应用层**（plan 完成时按声明强绑定），而不是依赖 AI 自由发挥。

---

## 3. 方案：三阶段（P0 声明绑定 → P1 脚本可控 → P2 直传剧本）

### P0 声明式 Character/Voice 绑定（解决失联根因，改动面最小）

**入参**（`generate-video`，fn=scenario-dialogue-video 新增可选块；media_uuid 来自既有 `add-uploaded-media`）：

```json
"scenario_config": {
  "characters": [
    { "media_uuid": "m-1", "name": "Alice", "voice_uuid": "v-123" }
  ],
  "environments": [
    { "media_uuid": "m-2", "name": "Office" }
  ]
}
```

**改动清单**：

| 仓库 | 改动 |
| ---- | ---- |
| visla-api | ① 入参校验：media 归属校验、voice_uuid 校验（复用 `list-voice` 同源校验）、name 非空；② 声明持久化到 `agent_project_attributes`（JSON，复用现有 JSON 列模式）或新轻量表 `agent_project_kit_binding(agent_project_id, media_id, intended_type, name, voice_id, status)`；③ assets 发 AI 时对被声明 media 追加语义标注（如 `usage=character_reference` + declaration id） |
| AI 网关（外部依赖） | 协议约束：USER 素材派生的 kit **必须回传 asset_id**；kit.name 与声明 name 对齐。⚠️ 需与 AI 团队确认排期，是本方案唯一外部依赖 |
| vcjs | plan 回调落 kit 时：`kitSource.assetId` → 查声明表 → 命中则强校验（type/name）并 `detail.voiceId = 声明 voice_id`（替代 gender 自动分配）；**兜底链**：assetId 未回传 → 按 name 匹配声明 → 仍不中则按现状自动分配 + warn 日志（不阻塞主流程） |

**兜底与兼容**：所有新入参可选；不声明 = 现状行为零变化；声明解析失败按字段级 ILLEGAL_PARAMS 拒绝（创建时失败优于运行期静默错绑）。

### P1 脚本可见可控（OpenAPI 编辑窗口）

复用 Web 语义与既有闸门，新增三个 OpenAPI 端点：

| 端点 | 语义 | 复用 |
| ---- | ---- | ---- |
| `GET /openapi/v1/project/{uuid}/scene-scripts` | 列表 + `scene_script_status` | web `list-script` VO |
| `PUT /openapi/v1/project/{uuid}/scene-scripts/{scriptUuid}` | 编辑 visual/audio_script/scene_text/speaker/ref_image → status=EDITING | web `edit-script` |
| `POST .../scene-scripts/renormalize`（可选） | EDITING → PROCESSING → AI 重排 → NORMAL | web `renormalize-script` |

- 闸门 **零改动**：vcjs `checkSceneScriptReady` 已实现 EDITING → RETRY（等待编辑完成）、30min 超时熔断。
- **编排决策点**（需产品确认）：一期 fetch pipeline 在 plan 完成后自动推进 create-video，用户没有编辑窗口。两个选项：
  - **选项 A（推荐）**：请求带 `hold_after_plan=true`（或 scenario 默认）→ plan 完成后 pipeline 挂起，用户编辑完显式调 `generate-video` 推进；
  - 选项 B：维持全自动，编辑视为「下一版」能力（脚本仅可读），P1 退化为只读端点。

### P2 直传完整剧本（高级，依赖 P0/P1）

```json
"scenario_config": {
  "scene_scripts": [
    { "sorting": 1, "duration": 8, "character": "Alice", "audio_script": "...",
      "visual": "...", "scene_text": ["..."] }
  ]
}
```

- `character` 用 **P0 声明的 name** 引用，落库时解析为 kit 主键（speaker_id），复用 `initAgentProjectSceneScript` 的映射模式（"-1" 哨兵 = voiceover）。
- 实现二选一：① 走 renormalize 通道（plan 完成 → 用户剧本作为 `edited_normalized_video_script` 发 AI 对齐 kits，脚本被 AI 微调）；② 纯本地落库跳过 AI 重排（脚本逐字保留，AI 仅负责后续 storyboard/motion）。**建议 ②**：用户直传的诉求就是「逐字可控」。

---

## 4. 推荐落地顺序与理由

| 阶段 | 解决 | 改动量 | 依赖 |
| ---- | ---- | ---- | ---- |
| P0 | asset→kit 失联 + voice 指定（明确点 1、目标核心） | 中（双仓 + AI 网关协议） | AI 团队确认 asset_id 强约束 |
| P1 | 剧本分镜可见可编辑（明确点 2） | 小（复用 web 语义 + 闸门） | 编排挂起决策 |
| P2 | 逐字直传剧本（目标完全体） | 小（复用 P0 声明 + P1 状态机） | P0 |

P0 先行的原因：voice/asset 失联是**数据层根因**，P1/P2 的脚本引用（speaker→kit）全部建立在 kit 正确之上；且 P0 不改任何状态机，风险最低。

## 5. 跨仓库同步清单（实施时）

- visla-api ↔ vcjs：声明持久化结构（attributes JSON 或新表）、声明应用逻辑（vcjs 回调）；新表则**双向 Entity/Mapper + visla-api 负责 Flyway**。
- openapi-test：新增用例——声明绑定全链路（asset→kit→voice 校验）、失联兜底路径、P1 编辑窗口 + 闸门 RETRY、P2 直传。
- vis-biz：实施后回填本文「状态」与 02 文档协议差异。

## 6. 风险

| 风险 | 缓解 |
| ---- | ---- |
| AI 网关不保证回传 asset_id（外部依赖） | name 匹配兜底 + 自动分配降级 + warn 可观测；与 AI 团队明确 SLA |
| 声明与 AI 规划数量不一致（用户声明 3 角色，AI 只规划 2） | 语义是「声明的 media 必须被用」，不锁死 kit 总数；AI 可补充角色（无声明→自动分配） |
| EDITING 挂起导致 fetch 长等待 | 沿用 `OpenApiTimeoutPolicy`（30min）熔断；选项 A 下挂起不计时或独立超时 |
| 直传剧本与 kits 不匹配（引用了未声明角色） | P2 创建时校验：script 引用的 character 必须在声明中存在 |
