# Scenario Dialogue × OpenAPI：kit 分档方案与 script 自动匹配

> 状态：**方案分档（未实施）**。基于 visla-api-v2-pjson / vcjs-v2-pjson 代码核实（2026-09）。
> 上游：[README.md](./README.md)（业务全景）、[02-api1275-data-and-protocol.md](./02-api1275-data-and-protocol.md)（数据与协议）。
> 本文只回答：OpenAPI 创建 scenario 项目时，kit 从哪来（两档）、scene script 如何自动匹配 kit + voice（分支与兜底）。

---

## 0. 关键前提（代码事实）

1. **kit 与用户上传的 agentProjectMedia 无关联**：kit 全部来源于 AI 的 asset 结果（plan-video 产物），AI 不回传我们的 media id；`kitSource.assetId → agentProjectMediaService.getById` 反查是防御性死路径。用户 asset 想成为 kit，只能**我方直接建 kit**（Web `add-uploaded-kit`，`sourceType=USER`，mediaId=用户 media、voiceId 可同时落）。
2. **renormalize 通道 = 「给定 kits + 给定 script → AI 重排」**：payload 为 `{extractParam, kits[], spatialUnits[], planVideoVo(edited script)}`，返回 normalized script（speaker_kit_id 引用入参 kits 语义 id）→ `initAgentProjectSceneScript` 硬删重建、映射 kit 主键。该通道可承载任意用户剧本。
3. **voice 现状**：plan 请求不带 voice 候选；vcjs `applyVoiceSelection` 按 kit gender 自动分配；Web 有改绑接口；OpenAPI 有 `GET /workspace/list-voice` 可查 voice_uuid 但无处消费。
4. **状态机与闸门**：`scene_script_status`（NORMAL/PROCESSING/FAILURE/EDITING）；create-video 前置 ≠NORMAL → 4609；OpenAPI fetch 闸门 EDITING/PROCESSING → RETRY + 30min 熔断。

---

## 1. 分档总览（决策树）

```text
OpenAPI 创建 fn=scenario-dialogue-video
│
├── 档位 1：未传 kit 声明（默认）──────── AI 生成 kit（现状，零改动）
│     script 中的角色由 AI 在 plan-video 中规划为 kit，
│     speaker 引用 AI 自产自配（天然一致），voice 按 gender 自动分配。
│
└── 档位 2：传入 kit 声明 ──────────────── 用户上传 kit（新增）
      scenario_config: {
        characters:   [ {name, asset(media_uuid|assetUrl), voice_uuid} ],
        environments: [ {name, asset} ],
        scene_scripts: [ ... ]          // 可选：完整剧本直传
      }
      │
      ├── 分支 2a：未传 scene_scripts（kit + 剧本素材 idea/script/doc）
      │     plan-video → 按声明替换/补建 kit（voice 落库）
      │     → renormalize：AI 按 kit【name】把剧本角色绑定到 kit（AI 辅助匹配）
      │     → 匹配不上的剧本角色 → AI 补建 kit + 自动 voice（兜底）
      │
      └── 分支 2b：传 scene_scripts（kit + 完整剧本）
            scene 角色引用 declaration name → 我方代码【确定性解析】为 kit 主键，零 AI 参与
            ├── 2b-i script_locked=true（推荐默认）：跳过 renormalize，剧本逐字本地落库
            │     plan-video 仍执行（spatial/coverage 规划仍需 AI）
            └── 2b-ii ai_align=true：走 renormalize，允许 AI 微调对齐 kits/spatial
```

---

### 1.1 参数场景路由（四场景）

按创建请求参数组合路由（asset = 独立上传素材；kit = 用户 kit 声明；scene_script = 用户直传分镜）：

| # | 参数组合 | kit 来源 | script 来源 | AI 调用 | 新增逻辑 | 结论 |
| - | ------ | ------ | ------ | ------ | ------ | ------ |
| 1 | asset + script | AI（现状） | AI | 1（plan-video） | 无 | ✅ 现状；kit 图是 AI 参考重生成，非 asset 本体（预期管理） |
| 2 | script + scene_script（±asset） | AI | 用户（renormalize 重排） | 2 | 无（纯复用通道） | ✅ AI kit 源自同一剧本，name 对齐质量天然高；voice 仅自动分配 |
| 3 | kit + scene_script（asset 忽略） | 用户 | 用户（覆盖 + renormalize） | 2 | 声明应用（建 kit+voice） | ✅ 确定性最高；可选 script_locked 退化本地直落 |
| 4 | kit + AI script + 用户 script | 用户 | 合并（覆盖式：用户 scene 按 sorting 占位覆盖，AI 补全） | 2 | 声明应用 + 合并规则 | ⚠️ 可行，合并语义待产品确认，可后置 |

路由判定（创建时）：`kit 声明?` → 3/4 vs 1/2；`scene_script?` + 覆盖语义 → 2 vs 3 vs 4。场景 2/3/4 的 characters 区分强弱引用（见 §3.1）；`scene_script` 非空时**豁免「idea/script/doc/webpage 至少其一」校验**。asset 与 kit 声明并存时：asset 仍作为素材进 plan-video（AI 可作环境/风格参考），kit 本体只认声明。

---

## 2. 档位 1：AI 生成 kit（现状档）

| 项 | 说明 |
| -- | ---- |
| 入参 | 现状不变：idea / script / doc / webpage 至少其一 + `aigc_config.enable_ai_director=true` |
| kit 来源 | plan-video 一次产出 kits + normalized script + spatial |
| script ↔ kit 匹配 | AI 自产自配：normalize 时角色描述 → kit，script 的 speaker_kit_id 引用同批产出，天然一致 |
| voice | vcjs `applyVoiceSelection` 按 kit gender 自动分配（仅 CHARACTER 消耗 voice） |
| 改动 | **无**。本档即现状（API-1286 一期行为） |

---

## 3. 档位 2：用户上传 kit

### 3.1 script 自动匹配 kit + voice 的机制（核心）

匹配拆成两段，确定性不同：

```text
scene_script ──(匹配A: 角色→kit)──> kit ──(匹配B: voice)──> TTS
```

| 匹配段 | 分支 2a（kit + 剧本素材） | 分支 2b（kit + 直传剧本） |
| ------ | ------ | ------ |
| **匹配A：script 角色 → kit** | **AI 辅助匹配**：renormalize 的 kits[] 携带（语义 id、name、图、visualAppearance），AI 按角色名+形象把剧本角色绑定到 kit，返回结构化 speaker_kit_id | **我方确定性解析**：scene_scripts[].characters 传声明 name，服务端按 name → 声明 kit 主键（speaker_id），零 AI 参与、可预校验 |
| **匹配B：kit → voice** | **确定性**：声明 voice_uuid 创建时落 `kit.voiceId`，不经过 AI；scene script 引用 kit 即间接锁定 voice | 同左 |

**voice 的确定性来源**：voice 是 kit 的属性（`agent_project_kits_detail.voiceId`），我方建 kit 时落库；TTS/口型链路按 kit.voiceId 取音。script 是否被 AI 改写不影响 voice 归属。

### 3.2 流程时序（两分支共用骨架）

```text
① 创建项目：剧本素材 + scenario_config 声明持久化（visla-api）
② plan-video 照常执行（计费；AI 产 kits + script + spatial，落库）
③ 声明应用（vcjs plan 回调后，新增步骤）：
   对每条声明：
     - AI 已规划同名 kit → 替换：kit.mediaId 换绑用户 media（buildRef + 资产标准化链）
                           + detail.voiceId = 声明 voice + 我方分配 ai_kit_id（语义 id）
     - AI 未规划         → 补建 kit（add-uploaded-kit 落库语义，sourceType=USER）
   AI 规划的额外角色（用户未声明）→ 保留，voice 走自动分配
④ 分支 2a：renormalize（重建 kits[] + 剧本素材 → AI 重排 → scene scripts 硬删重建）
   分支 2b：scene_scripts 本地直落（复用 initAgentProjectSceneScript 映射，"-1"=voiceover）
⑤ scene_script_status → NORMAL，闸门放行，create-video 现状不变
```

### 3.3 renormalize 请求 payload 清单（代码核实：`assembleRenormalizeScriptReqExtraBo`）

网关消息 `AIGC_RENORMALIZE_VIDEO_SCRIPT`；通用头（entityId/entityType/trackingId/userId/feePlan/msgType）外，业务载荷整体序列化为 S3 jsonFile，内容：

| payload 字段 | 组装来源 | 内容与要点 |
| ------ | ------ | ------ |
| `target_video_desc` | `aiAssetService.getTargetVideoDesc(agentProject)` | 用户目标视频描述（idea/剧本素材） |
| `extract_param`（AiAgentAttributeBo） | `getAiAgentAttributeBo(attributes, aigc)` | 项目属性全集：contentGuide / videoDurationInSeconds(+Hard) / aspectRatio / videoPace / videoType / audioLanguage(+Region) / textOverlayLanguage / emoji / userDefinedVisualizations / compositionMode / voiceOption / recommendKenBurns / speakerNotesVerbatim / styleFamily / targetAudience / videoPurpose / avatarModelPolicy |
| `plan_video`（PlanVideoVo，edited script） | `assembleNormalizedVideoScript`：**从 DB scene scripts 组装**（按 sorting） | title/summary（groupScript 或项目名/描述）；scenes[] = `toPlanVideoScriptVo`：scene_index、duration、coverage_shot_id / environment_kit_id/index（`ref_image_kit_id` → kit detail `ai_kit_id` 翻译，COVERAGE 型 → coverage_shot_id）、visual、speaker_kit_id/index、audio_script、scene_text |
| `kits`（KitsBo[]） | `queryKitsByAgentProjectId` | index、kitId（语义 id）、type、metadata、role、name、visualAppearance、creativeNotes、voiceProfile、kitSource、stillPrompt、style、speechCapability、gender、**voiceId**、visualReference —— kit 全量属性含 voice |
| `spatial_units`（SpatialCoverageUnitBo[]） | `querySpatialUnitsByAgentProjectId` | spatialUnitId、environmentKitIndex、cameraAxisRule、actorSlots、masterCoverageShot |

**对 OpenAPI 方案的两条直接推论**：
1. renormalize 的**全部输入都从 DB 组装**（scene scripts / kits / spatial / attributes），没有直接透传请求体的字段 → OpenAPI 走该通道 = 「用户数据先落库（声明建 kit、scene_script 写入），再触发 renormalize」，方案与通道天然契合，无需改 AI 协议。
2. 我方重建 kit 时只要落好 `ai_kit_id`，出向的 kit 语义 id 翻译（`resolveKitDetail`）自动工作；kit 的 name / visualAppearance / 图（kitSource）/ voiceId 随 payload 全量给 AI，具备按声明对齐的信息基础。

### 3.4 校验与兜底规则

| 规则 | 行为 |
| ---- | ---- |
| 声明 name 重复 / voice_uuid 非法 / media 不属于本项目 | 创建时 ILLEGAL_PARAMS 拒绝（失败前置于运行期） |
| scene_scripts 引用了未声明的 character（2b） | 创建时拒绝 |
| AI 规划出同名但多个 kit（2a 匹配歧义） | 精确 name 匹配优先；歧义走「补建」不猜测错绑 + warn 日志 |
| 声明角色 AI 未规划（2a/2b-i） | 补建 kit（用户 asset 即 kit 本体） |
| 用户未声明的剧本角色 | AI 动态生成 + 自动 voice（退化为档位1行为） |
| createVideos 出向 | scene script → 反查 `ai_kit_id`（我方分配的语义 id）→ 语义 id 出向，链路自洽 |

### 3.5 scene_script 用户入参与字段填充矩阵（DB 全字段 × 数据来源）

用户入参无法也不需要覆盖表全部字段。入参模型（每条 scene）：

```json
{ "sorting": 1, "text": "...", "visual": "...", "scene_text": ["..."],
  "duration": 8, "characters": ["Alice"], "environment": "Office" }
```

DB 全字段来源矩阵（落库实现在 vcjs `buildSceneScript`，哨兵规则以此为准）：

| DB 字段 | 约束 | 用户入参（场景 3/4） | AI 回填（renormalize 返回） | 系统 | 映射规则 |
| ------ | ---- | ------ | ------ | ------ | ------ |
| `sorting` | NOT NULL | **必传**（或由数组顺序推导） | scene_index | | `scene_index + 1`，缺省 1 |
| `audio_script` | varchar 2000 | **text：核心字段**（与 visual 至少其一非空） | audio_script | | 直传，2000 截断 |
| `visual` | varchar 2000 | 可选 | visual | | 直传，2000 截断 |
| `scene_text` | json | 可选（`List<String>`） | scene_text | | List → JSON，空则 null |
| `duration` | double 可空 | 可选 | duration | | 直传；未传由 AI 规划，仍空则生成端自定 |
| `speaker_id` | kit 主键 | characters：声明 name（**强引用**，创建时校验存在） | speaker_kit_id | | kitsVoMap 映射；未命中 → **-1**（voiceover 哨兵）；speaker 空且 text 空 → **null**（纯视觉场） |
| `ref_image_kit_id` | kit 主键 | environment：声明 name（强引用） | coverage_shot_id / environment_kit_id | | kitsVoMap 映射，未命中 → **-1**；AI spatial 锚定会改写此字段 |
| `status` | 默认 0 | | | **ACTIVE** | 系统固定 |
| `id` / `agent_project_id` / `created_at` / `modified_at` | | | | **系统生成** | |
| （`visible_kit_indexes`） | 表无对应列 | 不传 | AI 返回但不落库 | 出向时按 kit 关系组装 | 仅 createVideos 出向协议字段 |

**用户字段与 AI 回填的关系**（关键）：走 renormalize 的分支（2a/3/4-ai_align）不存在「用户字段 + AI 字段合并」——用户 scene_script 是 renormalize 的**输入**，落库的是 AI 返回的**整条重排结果**（text 可能被微调、ref_image_kit_id 会被 spatial 锚定改写）。要逐字保留走 `script_locked` 本地直落：用户字段直接落库、AI 不回填，duration/visual 留空由生成端自定。分支 2a 的 characters 是**弱引用**（自由角色名，AI 在 renormalize 时对齐到 AI kit），不做创建时校验。

### 3.6 风险

| 风险 | 缓解 |
| ---- | ---- |
| 场景 3/4 的 scene_script 仅传 text+sorting 时，renormalize 后 visual/duration 全由 AI 补，用户预期可能偏差 | 文档明示：用户字段是重排输入而非最终值；逐字/字段级控制诉求走 script_locked 本地直落 |
| 2a 的 AI name 匹配质量（角色名泛化/别名） | 文档要求声明 name 使用剧中角色名；结构化返回可审计；错配走补建不静默 |
| 用户 environment 图与 AI coverage/spatial 规划的整合 | renormalize payload 含 spatialUnits；联调验证覆盖质量，不行则 environment 场景降级 `spatial_mode=none` |
| 声明 media 图片规格与 AI 产物差异 | 复用 `add-uploaded-kit` 已验证的资产标准化处理链 |
| 2b-i 跳过 renormalize 时 spatial 仍来自 plan-video（基于 AI 图） | 可选 `spatial_mode=none` 联合跳过 plan-video（省一次 AI 调用）——需 AI 侧确认空 spatial 的 createVideos 可接受，标注待确认 |

---

## 4. 待确认决策点

| # | 决策点 | 倾向 |
| - | ------ | ---- |
| 1 | 2a/2b 中 renormalize 是第二次 AI 调用（成本）：随 plan-video 打包计费 vs 独立计费点 | 打包计费（用户视角一次创建） |
| 2 | 2b 默认 script_locked（逐字保留）还是 ai_align（允许微调） | script_locked 默认，ai_align 显式开启 |
| 3 | 2b-i + `spatial_mode=none` 跳过 plan-video：AI 侧空 spatial 的 createVideos 兼容性 | 与 AI 团队确认后开放 |
