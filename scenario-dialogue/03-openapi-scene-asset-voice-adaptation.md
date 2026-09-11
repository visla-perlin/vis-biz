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

### 2.1 kit 与用户 asset：DB 层无关联（关键前提）

| 环节 | 事实 |
| ---- | ---- |
| 入向 | plan 请求确实携带 asset id：`assembleAssetInfo` 中 `mediaAsset.setAssetId(groupMedia.getId().toString())`（`AiAgentAssetInfoBo.assetId` = **agentProjectMedia.id**），连同 assetUrl / usage / metadata 发给 AI |
| 出向 | **AI 返回的 kit 与 agentProjectMedia 没有关系**：kit 是 AI 视觉分析后的**产出物**（新图/新实体），`AigcPlanKitsVo.KitSource{mode, assetUrl, assetId, previousAssetUrl, seed}` 携带的是 AI 侧产物引用，不会可靠地带回我们的 agentProjectMedia.id |
| 落库 | vcjs 保存 kit detail 时 `kitSource.assetId → agentProjectMediaService.getById` 的反查是**防御性死路径**（真实链路 AI 不回传我们的 id，必 miss）→ 走 `handleUrl` 用 assetUrl **新建 media**，kit.mediaId 指向 AI 产物 |

**结论（对应明确点 1）**：kit 数据全部来源于 AI 的 asset 结果，与第一步上传的 agentProjectMedia 在 DB 层**没有任何关联**。这不是「补个回传约束」能解决的：kit 的本体就是 AI 产出物，天然不是用户的 asset。要建立关联，只能**让 kit 不经过 AI——直接从用户 media 创建**。该机制 Web 端已存在：`add-uploaded-kit`（`AgentProjectKitsService.addUploadKit`，`sourceType=USER`，kit.mediaId=用户 media、voiceId 可同时指定）；但 AI plan 链路与 OpenAPI 均未使用。

### 2.2 voice 关联：有自动分配与 Web 改绑，无用户指定入口

| 环节 | 事实 |
| ---- | ---- |
| plan 请求 | **不携带 voice 候选**（plan 请求无 voice 字段）——AI 不选音色 |
| 回调分配 | vcjs `applyVoiceSelection`：按 kit gender 自动分配（`selectVoiceForKit`），无匹配 → 0。仅 CHARACTER 消耗 voice |
| Web 改绑 | kit `edit-kit` 的 `voiceUuid` / `ChangeProjectKitVoiceReqBody`：校验归属后落 `detail.voiceId` |
| OpenAPI | 已有 `GET /openapi/v1/workspace/list-voice?localeName=`（系统 + 自定义 voice）——**voice_uuid 可查，但无 kit 接口可消费它** |

**结论**：kit 是 AI 新造实体，「哪个 kit 是哪个角色」对我方不可控，即使拿到了 voice_uuid 也无处绑定。voice 指定的能力件齐全（查询 + 落库字段 + 校验），缺的是**建 kit 时带上 voice 的入口**（Web `add-uploaded-kit` 已支持，OpenAPI 无）。

### 2.3 normalize：无独立创建期接口，用户不能自定义结构化剧本

| 环节 | 事实 |
| ---- | ---- |
| 创建期 | normalize_script 是 `aigc_plan_video`（2010）网关消息的**内部第一步**，与 plan_kits / plan_spatial 一次回调返回。**没有独立 normalize 接口** |
| 编辑期 | Web 有 `POST /v3/agent-project/{uuid}/scene-script/renormalize-script`：payload = `RenormalizeScriptReqBo{extractParam, kits, spatialUnits, planVideoVo(含 edited script)}` → AI 重排 → scene scripts **硬删重建** → status NORMAL |
| 自定义 | 入参不支持直接给结构化剧本；Web 也只能「编辑 AI 产物」。但 renormalize 通道的 `edited_normalized_video_script` 事实可承载**任意用户剧本**，且其 kits 参数证明「**以既有 kits 为前提做脚本规划**」的 AI 通道存在 |

**结论（对应明确点 2）**：「character ↔ voice ↔ 剧本」的关联要成立，前提是 **kit 先于剧本、由我方从用户 asset 直接创建**（声明层），AI 只在该 kit 集合上做脚本编排。

---

## 3. 方案：三阶段（P0 声明建 kit → P1 脚本可控 → P2 直传剧本）

### P0 声明式 Character/Voice 绑定（关联在我方建立，不经过 AI）

**核心思路**：复用 Web `add-uploaded-kit` 的既有语义（kit 直接从用户 media 创建，`sourceType=USER`），把「建 kit + 绑 voice」从 AI 规划中剥离出来，由声明驱动、在 plan 编排中自动应用。

**入参**（`generate-video`，fn=scenario-dialogue-video 新增可选块；media_uuid 来自既有 `add-uploaded-media`，voice_uuid 来自既有 `list-voice`）：

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

**流程编排**：

```text
① 创建项目 + 声明持久化（visla-api：attributes JSON 或新表 agent_project_kit_binding）
② plan_video 照常触发（AI 规划 kits + scene scripts，现状不变）
③ vcjs plan 回调尾部【声明应用】（新增步骤）：
   对每条声明，在 AI 规划的 kits 中按 name 匹配：
     命中 → kit.mediaId 换绑用户 media（含 buildRef(AGENT_KIT) + 资产处理链）
            + detail.voiceId = 声明 voice_id
     未命中（AI 没规划出该角色）→ 按声明直接补建 kit（add-uploaded-kit 落库语义）
   场景脚本 speaker 引用随 kit 主键自然成立（initAgentProjectSceneScript 按 kit 映射）
```

**改动清单**：

| 仓库 | 改动 |
| ---- | ---- |
| visla-api | 入参校验（media 归属 / voice_uuid 校验复用 list-voice 同源 / name 非空）+ 声明持久化 |
| vcjs | plan 回调尾部新增「声明应用」步骤：name 匹配 → media 换绑 + voiceId 落库 / 补建 kit。两仓共享 `agent_project_kits*` 表，vcjs 已有全套 AgentProjectKitsService DAO，资产标准化处理链在 vcjs 侧本就存在 |
| AI 网关 | **零协议改动**（可选增强：plan prompt 约束角色名对齐声明，提升 name 匹配率——非阻塞，后置） |

**兼容性**：新入参可选；不声明 = 现状行为零变化；声明解析失败按字段级 ILLEGAL_PARAMS 拒绝（创建时失败优于运行期静默错绑）。

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
  - 选项 B：维持全自动，P1 退化为只读端点（脚本仅可查）。

### P2 直传完整剧本（高级，依赖 P0）

```json
"scenario_config": {
  "scene_scripts": [
    { "sorting": 1, "duration": 8, "character": "Alice", "audio_script": "...",
      "visual": "...", "scene_text": ["..."] }
  ]
}
```

- `character` 用 **P0 声明的 name** 引用，落库时解析为声明 kit 主键（speaker_id），复用 `initAgentProjectSceneScript` 的映射模式（"-1" 哨兵 = voiceover）。
- **纯本地落库，跳过 AI 重排**（用户直传的诉求就是逐字可控；剧本引用的角色必须在声明中存在，创建时校验）。AI 仅负责后续 storyboard/motion 生成。

---

## 4. 推荐落地顺序与理由

| 阶段 | 解决 | 改动量 | 外部依赖 |
| ---- | ---- | ---- | ---- |
| P0 | asset→kit 关联 + voice 指定（明确点 1、目标核心） | 中（双仓，纯我方逻辑） | **无**（可选 prompt 增强后置） |
| P1 | 剧本分镜可见可编辑（明确点 2） | 小（复用 web 语义 + 闸门） | 编排挂起决策 |
| P2 | 逐字直传剧本（目标完全体） | 小（复用 P0 声明 + P1 状态机） | 无 |

P0 先行的原因：kit 与 asset 的关联是**数据层根因**——kit 不从用户 asset 创建，voice/剧本的所有引用都是空中楼阁；且 P0 完全复用 Web 已验证的 `sourceType=USER` 建 kit 语义，不碰 AI 协议，风险最低。

## 5. 跨仓库同步清单（实施时）

- visla-api ↔ vcjs：声明持久化结构（attributes JSON 或新表，**新表则双向 Entity/Mapper + visla-api 负责 Flyway**）；声明应用逻辑在 vcjs 回调（读共享表）。
- openapi-test：新增用例——声明建 kit 全链路（media 换绑 + voice 校验）、AI 未规划出声明角色的补建路径、P1 编辑窗口 + 闸门 RETRY、P2 直传 + 角色引用校验。
- vis-biz：实施后回填本文「状态」与 02 文档协议差异。

## 6. 风险

| 风险 | 缓解 |
| ---- | ---- |
| name 匹配歧义（AI 规划的角色名与声明 name 对不上/多个相近） | 匹配策略：精确匹配优先，忽略大小写/空白；未匹配一律走「补建」而非猜测错绑；文档明确要求声明 name 使用剧中角色名 |
| AI 规划的角色数 ≠ 声明数（AI 补充了额外角色） | 语义定为「声明的 media 必须被用」，不锁死 kit 总数；AI 额外角色按现状自动分配 voice |
| 声明 media 的图片规格与 AI kit 产物不同（尺寸/风格） | 复用 `add-uploaded-kit` 的既有资产标准化处理链（media process），该链路 Web 端已验证 |
| EDITING 挂起导致 fetch 长等待 | 沿用 `OpenApiTimeoutPolicy`（30min）熔断；选项 A 下挂起不计时或独立超时 |
