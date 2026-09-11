# aigc — 场景级 AIGC 生成

> 业务域边界：scene 级 AIGC 生成的业务契约——prompt 接口、prompt/storyboard metadata 存储与回退、selected 生命周期、角色语音编辑分流。
> 不覆盖：AI Director 编排入口（plan-video，见 [`../scenario-dialogue/`](../scenario-dialogue/)）、语音生成链路细节（见 [`../voice/`](../voice/)）、PJSON 协议实现（见 `vme-pjson/`）。

## 覆盖内容

- prompt 增强三接口（enhance-image-prompt / enhance-video-prompt / animate-storyboard）的架构与契约
- generate-video / regenerate-video 的 prompt 与 storyboard metadata 存储规则
- selected rel 的生命周期与删除语义
- get-storyboard-metadata 的回退规则（核心契约）
- 角色语音编辑分流：换音色 / 改台词 → 换声（画面复用）vs 重生成视频
- generate-image PJSON 测试用例快照

## 文档

| 文档 | 说明 |
| ---- | ---- |
| [`aigc-prompt-interfaces.md`](./aigc-prompt-interfaces.md) | prompt 增强三接口的共同架构与逐接口契约 |
| [`aigc-prompt-metadata-rule.md`](./aigc-prompt-metadata-rule.md) | prompt 存储 / selected 生命周期 / metadata 回退规则（全域 canonical，见根 README §3.3 #4） |
| [`kit-voice-flow.md`](./kit-voice-flow.md) | 角色语音**编辑操作分流**视角（canonical 见根 README §3.3 #2） |
| [`generate-image-pjson-testcase.md`](./generate-image-pjson-testcase.md) | generate-image PJSON 测试用例快照 |

## 与其它业务域的边界

- Character Voice 语音**生成链路**（触发判定 / `project_scene_kit_voice` / RefAudioFlow）→ 见 [`../voice/scene-kit-voice-flow.md`](../voice/scene-kit-voice-flow.md)（canonical，本域 [`kit-voice-flow.md`](./kit-voice-flow.md) 只讲编辑分流，链路细节以它为准）
- AI Director 编排（plan-video / kits / spatial）与 scene script → 见 [`../scenario-dialogue/`](../scenario-dialogue/)；本域的 storyboard/motion 生成是其 Step3 下游
- scene 背景素材（footage）来源与状态机 → 见 [`../footage/`](../footage/)
- PJSON 协议与 handler 实现 → 见 `vme-pjson/` 专题仓（本域文档只写业务语义）
