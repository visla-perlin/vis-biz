# voice — 语音 / TTS

> 业务域边界：一切「声音怎么来」的业务——TTS 合成主流程、character voice（角色配音）链路、TTS ↔ PES 同步契约、上传配音 aligner 与项目翻译。
> 不覆盖：触发语音编辑的用户操作分流（见 [`../aigc/kit-voice-flow.md`](../aigc/kit-voice-flow.md)）、数字人合成（见 [`../avatar/`](../avatar/)）、PJSON handler 实现（见 `vme-pjson/`）。

## 覆盖内容

- TTS 在 project generation 中的位置、API 发起 → vcjs Job 执行 → Align 对齐的主流程与状态机
- character voice（`on_camera_speaker`）：触发判定、`project_scene_kit_voice` 数据载体、RefAudioFlow 三种策略
- video 生成接口的 voice 流程：一段式 vs 两段式、ref_audio 三层注入
- TTS ↔ PES 两条链路的同步协议、失败与部分成功语义
- TTS PJSON 接入设计（业务语义侧）
- 上传配音 aligner 与 duplicate-for-translation 项目的契约与状态机

## 文档

| 文档 | 说明 |
| ---- | ---- |
| [`tts-flow.md`](./tts-flow.md) | TTS 主流程与状态机（canonical，见根 README §3.3 #3） |
| [`scene-kit-voice-flow.md`](./scene-kit-voice-flow.md) | character voice 语音生成链路（canonical，见根 README §3.3 #1） |
| [`video-generation-voice-flows.md`](./video-generation-voice-flows.md) | video 生成接口的 voice 流程 / 一段式 vs 两段式 |
| [`tts-pes-sync-contract.md`](./tts-pes-sync-contract.md) | TTS ↔ PES 同步对接协议 |
| [`tts-pjson-design.md`](./tts-pjson-design.md) | TTS PJSON 接入设计（业务语义；实现见 `vme-pjson/`） |
| [`upload-aligner-and-translation.md`](./upload-aligner-and-translation.md) | 上传配音 aligner + 项目翻译 API 契约与状态机 |

## 与其它业务域的边界

- 角色语音**编辑操作分流**（换音色/改台词 → 换声 vs 重生成）→ 见 [`../aigc/kit-voice-flow.md`](../aigc/kit-voice-flow.md)（canonical，见根 README §3.3 #2；本域只讲语音生成与执行策略）
- 数字人 lip-sync 的音频来源引用本域 TTS；Avatar 体系本身 → 见 [`../avatar/`](../avatar/)
- 配音类快速编辑操作的接口契约与状态流转 → 见 [`../clip/`](../clip/)（见根 README §3.3 #8：操作契约归 clip，音频语义归本域）
- 多角色情景对话的 speaker voice 绑定 → 见 [`../scenario-dialogue/`](../scenario-dialogue/)
