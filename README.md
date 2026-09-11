# vis-biz — 跨项目业务文档库

> 跨 visla-api / vcjs（以及 admin-api 等周边）共用的**业务规则 / 契约**文档库。
> 回答「业务是什么、为什么这样设计」；「某个仓库怎么实现的」不在这里。

---

## 1. 定位与三库边界

| 仓库 / 目录 | 放什么 | 不放什么 |
| ------ | ------ | ------ |
| **vis-biz**（本库） | 业务规则、状态机、接口/协议契约、数据模型语义、跨仓库链路 | 实现笔记、BUG 修复过程、代码索引 |
| `visla-api/ai-notes/` 等项目内目录 | 单仓库实现笔记（BUG / 设计 / 计划 / 代码索引） | 跨仓库共用的业务规则 |
| `vme-pjson/` | PJSON（PES/VME）架构专题：协议、handler、改造进度 | PJSON 之外的业务规则（业务语义引 vis-biz） |

判定口诀：**这段话如果同时约束两个以上仓库的行为 → vis-biz；只约束一个仓库的代码怎么写 → 该仓库自己的笔记；只属于 PJSON 改造 → vme-pjson。**

---

## 2. 内容结构规划

### 2.1 目录布局

```text
vis-biz/
├── README.md                  # 总导航 + 结构规划 + 重叠引用规划（本文件）
└── {biz}/                     # 业务域目录，kebab-case，一个目录一个业务域
    ├── README.md              # 【必备】域内导航，固定四节（见 2.3）
    ├── {topic}.md             # L1 核心业务规则文档（不编号）
    ├── {NN}-{topic}.md        # L2 协议明细 / 系列补编（NN- 编号，如 02-api1275-data-and-protocol）
    └── {topic}-testcase.md    # L3 测试用例快照 / 验证记录
```

### 2.2 文档分层

| 层 | 内容 | 命名 | 示例 |
| -- | ---- | ---- | ---- |
| **L0 域 README** | 域边界、覆盖内容、文档清单、域间边界 | 固定 `README.md` | [`clip/README.md`](./clip/README.md) |
| **L1 业务规则** | 业务是什么、用户操作 → 系统动作、状态机、契约 | `{topic}.md`，kebab-case | [`voice/tts-flow.md`](./voice/tts-flow.md) |
| **L2 协议/明细** | 按 Linear issue 的增量协议、字段级变更明细、对外对接说明 | `{NN}-{topic}.md`，编号表示补编顺序 | [`scenario-dialogue/02-api1275-data-and-protocol.md`](./scenario-dialogue/02-api1275-data-and-protocol.md) |
| **L3 测试/验证** | PJSON 测试用例、端到端验证快照 | `{topic}-testcase.md` | [`aigc/generate-image-pjson-testcase.md`](./aigc/generate-image-pjson-testcase.md) |

命名规则：

1. 文件名一律 kebab-case 英文，禁止中文/空格/下划线混用。
2. 主文档用业务名而非 issue 号（`tts-flow.md` 而非 `api-1001.md`）；issue 归属写在文档头与 L2 编号里。
3. 一篇文档只讲一个主题；超过 ~400 行且含两个独立主题时，拆分并在域 README 登记。

### 2.3 域 README 固定四节

每个业务域目录**必须**有 README.md，固定四节结构（现有 clip / footage / avatar 已是此风格）：

1. **业务域边界**：一句话说清本域管什么、不管什么。
2. **覆盖内容**：本域文档覆盖的知识点清单。
3. **文档清单**：`| 文档 | 说明 |` 表格。
4. **与其它业务域的边界**：列出「本域不展开、去哪里看」的相对链接。

### 2.4 单篇文档结构约定

推荐骨架（不强制的八股，但开头三行必须有）：

```markdown
# {主题}

> 范围：本文讲什么、不讲什么
> 视角：业务 / 协议 / 对接（多视角文档要声明以哪个为准）
> 关联：canonical 文档或被引用文档的相对链接

## 1. 业务定位        # 是什么、解决什么问题
## 2. 核心概念        # 术语表
## 3. 数据载体        # 表 / 字段 / 枚举（语义，不是建表语句）
## 4. 流程与状态机    # 链路图、状态流转、幂等/失败语义
## 5. 边界与互引      # 与其它域的重叠点及链接
```

### 2.5 文档生命周期

- **新增**：先判断归属（§3.1 判定法），再建目录/文档，并在域 README 与本文件 §4 登记。
- **演进**：修改 canonical 文档时，必须 grep 引用方（`rg "\.\./{biz}/{doc}"`），同步检查引用表述是否仍成立。
- **废弃**：不删文件，文档头标注 `> ⚠️ Deprecated：被 {link} 取代`，域 README 清单同步标记。

---

## 3. 业务重叠与引用规划（核心）

跨域业务（如 kit voice 同时涉及 voice / aigc / avatar / scenario-dialogue）**必然产生文档重叠**。重叠不可怕，可怕的是两处各自演进、互相矛盾。规则如下。

### 3.1 归属判定：一段规则只有一个 canonical

一段业务规则的**唯一权威版本（canonical / 主责文档）**用三问判定：

1. **改动联动问**：这条规则变化时，哪个域必须第一个改文档？→ 那是 owner。
2. **删除损失问**：删掉哪篇会造成不可恢复的信息损失？删掉另一篇只是丢一个链接？→ 损失大的是 owner。
3. **真相源问**：这条规则锚定在表 / 状态机 / 协议上，它首次在哪篇文档被完整定义？→ 那是 owner。

判定结果必须登记进 §3.3 重叠矩阵；未登记的重叠视为技术债。

### 3.2 引用规范

| 规则 | 说明 |
| ---- | ---- |
| **相对链接 + 节定位** | `见 [voice/scene-kit-voice-flow.md §4](../voice/scene-kit-voice-flow.md)`，写清去看**什么问题**，不许只丢一个裸链接 |
| **摘要上限** | 引用方允许 ≤3 句概述 + 术语对齐；**禁止**复制状态机表、字段表、流程图——复制即失联 |
| **视角分工声明** | 重叠双方必须在文档头「范围 / 视角」声明分工（如：本文讲编辑操作分流，语音生成链路见 xxx），并互相链接 |
| **双向登记** | owner 域 README 第 4 节登记「被谁引用」；引用方登记「本域不含 X，见 …」 |
| **漂移处理** | 发现引用与 canonical 不一致：canonical 对 → 修引用方；canonical 过时 → 先修 canonical，再同步引用方，**不允许**只在引用方「就地改对」 |

### 3.3 已知重叠矩阵

| # | 重叠主题 | Canonical（唯一权威） | 引用方（允许的视角） |
| -- | ------ | ------ | ------ |
| 1 | Character Voice 语音生成链路（触发判定 / `project_scene_kit_voice` / RefAudioFlow 状态机） | [`voice/scene-kit-voice-flow.md`](./voice/scene-kit-voice-flow.md) | [`aigc/kit-voice-flow.md`](./aigc/kit-voice-flow.md)（只讲编辑分流，链路细节引用）；`scenario-dialogue/`（speaker voice 绑定）；`avatar/`（lip-sync 音频来源） |
| 2 | 角色语音编辑分流（换音色/改台词 → 换声 vs 重生成视频） | [`aigc/kit-voice-flow.md`](./aigc/kit-voice-flow.md) | `voice/scene-kit-voice-flow.md`（用户触发入口互链） |
| 3 | TTS 主流程与状态机（API 发起 → vcjs Job → Align） | [`voice/tts-flow.md`](./voice/tts-flow.md) | `avatar/`（音频来源）；`clip/`（voice_over 只写操作契约，音频语义引 voice） |
| 4 | AIGC prompt 存储 / selected 生命周期 / storyboard metadata 回退 | [`aigc/aigc-prompt-metadata-rule.md`](./aigc/aigc-prompt-metadata-rule.md) | `scenario-dialogue/`（candidate 固态链）；`voice/`（metadata 契约引用） |
| 5 | AI Director 编排（plan-video / normalize_script / kits / spatial） | [`scenario-dialogue/README.md`](./scenario-dialogue/README.md) | `aigc/`（scene 级 storyboard/motion 是其 Step3 下游） |
| 6 | PJSON 协议与 handler 实现 | `vme-pjson/` 专题仓（本库之外） | 全域：vis-biz 只写业务语义与契约，实现细节链 vme-pjson |
| 7 | scene 背景素材（footage）来源分类与 `process_status` 状态机 | [`footage/footage-source-and-processing.md`](./footage/footage-source-and-processing.md) | `clip/`（clip 作为 footage 来源）；`aigc/`（AIGC 生成 footage 的引用） |
| 8 | 配音类快速编辑（voice_over / aligner） | 按「操作契约 vs 音频语义」切分：接口与状态流转归 `clip/`，音频/TTS 处理语义归 `voice/` | 双向互链 |

> 新增重叠时：先走 §3.1 三问定 canonical → 加入本矩阵 → 在两个域 README 互登边界。

---

## 4. 当前业务域地图

| 域 | 一句话边界 | 主要引用关系 |
| -- | ------ | ------ |
| [`aigc/`](./aigc/) | scene 级 AIGC 生成：prompt 接口、metadata/selected 契约、角色语音编辑分流 | 引用 voice(#1)、footage(#7)；被 scenario-dialogue(#5) 引用 |
| [`avatar/`](./avatar/) | 数字人 / AI 主播：Avatar / Look / 景别 / lip-sync | 引用 voice(#3)；与 scenario 的 kit speaker 是两套体系，勿混 |
| [`clip/`](./clip/) | clip 素材媒体预处理 action 体系与快速编辑契约 | 引用 voice(#3/#8)、footage(#7)、aigc(#7) |
| [`footage/`](./footage/) | scene 背景 B-Roll：来源分类与处理状态机 | 被 clip / aigc 引用（#7） |
| [`scenario-dialogue/`](./scenario-dialogue/) | 多角色情景对话视频（AI Director Mode）全链路 | 引用 aigc(#4/#5)、voice(#1) |
| [`voice/`](./voice/) | 语音 / TTS：主流程、character voice、PES 同步、aligner/翻译 | 被全域引用（#1/#2/#3/#8） |

> 域内文档清单见各自 README；本表只回答「该去哪个域」。

---

## 5. 规划中的业务域（按需新增，勿提前铺目录）

| 候选域 | 触发条件（出现以下需求时再建） |
| ------ | ------ |
| `credit-billing/` | 预扣 / 确认 / 退款三路径的跨仓库规则需要独立成文时 |
| `access-control/` | ACL / Workspace / Teamspace 权限模型需要跨项目对齐时 |
| `webhook-delivery/` | 对外交付契约（幂等 / failCount / deliveryStatus）需要独立域时 |
| `project-timeline/` | scene / timeline 结构与层级规则被 ≥2 个域反复引用时 |
| `openapi-contract/` | OpenAPI 对外契约的「业务语义」文档多到需要独立域时（当前散在 scenario-dialogue 等） |

新增域步骤：建目录 + 域 README（§2.3 四节）→ 更新 §4 地图 → 有重叠则登记 §3.3。

---

## 6. Agent / 开发者路由表

| 任务涉及 | 先读 |
| ------ | ------ |
| AIGC scene 生成、prompt、storyboard metadata、selected | [`aigc/`](./aigc/) |
| TTS、配音、character voice、翻译 | [`voice/`](./voice/) |
| 数字人 / Avatar / lip-sync | [`avatar/`](./avatar/) |
| clip 预处理、快速编辑（cut / transform / voice_over 操作） | [`clip/`](./clip/) |
| scene 背景素材 / B-Roll | [`footage/`](./footage/) |
| 多角色情景对话（scenario-dialogue-video / plan-video / scene script） | [`scenario-dialogue/`](./scenario-dialogue/) 及其引用链 |
| PJSON / PES 改造 | `vme-pjson/` + 本库对应域的业务语义 |

跨仓库任务（visla-api ↔ vcjs 同步）必读本库对应域文档后再动代码；发现文档与实现不一致，先修文档或登记 issue，再改代码。

---

## 7. 维护与 Git 规范

- 分支：直接在 `main` 上编辑，推送前按 IdeaProjects 总规范展示 commit 并确认；结构性调整建议开分支评审。
- Commit：`docs: {english desc}`；涉及具体 Linear issue 时用 `docs[{issueId}]: {desc}`。
- 每次改动自查：域 README 清单是否同步？§4 地图是否需要更新？是否产生了未登记的重叠（§3.3）？
