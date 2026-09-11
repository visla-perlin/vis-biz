# clip — Clip 素材与快速编辑

> 业务域边界：`clip` 素材（视频 / 音频 / 图片）的媒体预处理 action 体系与快速编辑（quick edit）契约。
> 覆盖：clip 预处理链路（visla-api → transcoder → vcjs）、action 契约与状态机、快速编辑操作（cut / extract / trim / merge / voice_over / reformat / transform 等）的接口契约。

## 覆盖内容

- clip 预处理 action 体系全景：三方链路（visla-api 发起 → transcoder 执行 → vcjs 回调处理）
- `PreprocessActionEnum` 动作清单与请求 / 结果 payload 契约
- `media_process_result` 状态机（init / todo / doing / success / failed）
- 快速编辑操作的状态流转与后继动作触发规则
- rotation（旋转）的业务契约（API-1281）

## 文档

| 文档 | 说明 |
| ---- | ---- |
| [`preprocess-actions.md`](./preprocess-actions.md) | 预处理 action 体系：三方链路、契约、状态机、action 清单 |
| [`transform.md`](./transform.md) | clip 变换 rotate/flip（API-1281）：需求、契约、跨仓库改动清单 |
| [`transform-integration.md`](./transform-integration.md) | API-1281 对接文档：前端传参/状态感知、VME SQS 契约、vcjs 回调链路 |

## 与其它业务域的边界

- 场景背景素材（`project_timeline.category = FOOTAGE`）的导入与替换 → 见 [`../footage/`](../footage/)
- AIGC 生成的视频 / 图片 → 见 [`../aigc/`](../aigc/)
- 语音 / TTS → 见 [`../voice/`](../voice/)
- 本文聚焦「clip 素材本身的媒体处理与快速编辑」，不涉及 project / scene 内的编辑
