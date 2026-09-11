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

### 3.3 校验与兜底规则

| 规则 | 行为 |
| ---- | ---- |
| 声明 name 重复 / voice_uuid 非法 / media 不属于本项目 | 创建时 ILLEGAL_PARAMS 拒绝（失败前置于运行期） |
| scene_scripts 引用了未声明的 character（2b） | 创建时拒绝 |
| AI 规划出同名但多个 kit（2a 匹配歧义） | 精确 name 匹配优先；歧义走「补建」不猜测错绑 + warn 日志 |
| 声明角色 AI 未规划（2a/2b-i） | 补建 kit（用户 asset 即 kit 本体） |
| 用户未声明的剧本角色 | AI 动态生成 + 自动 voice（退化为档位1行为） |
| createVideos 出向 | scene script → 反查 `ai_kit_id`（我方分配的语义 id）→ 语义 id 出向，链路自洽 |

### 3.4 风险

| 风险 | 缓解 |
| ---- | ---- |
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
