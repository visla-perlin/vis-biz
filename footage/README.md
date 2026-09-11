# footage — 场景背景素材（B-Roll）

> 业务域边界：scene 背景视频/图片素材（`project_timeline.category = FOOTAGE`）的来源分类、`project_asset` 处理状态机与 add/replace 接口契约

## 覆盖内容

- footage 的来源分类（本地上传 / free stock / premium stock / clip / 跨项目素材）
- `process_status` 状态机与各来源的初始状态差异
- uploaded_from_local 与 free/premium stock 的不同预处理链路
- `scene/add-footage` 与 `scene/replace-footage` 两个接口的契约与就绪校验差异

## 文档

| 文档 | 说明 |
| ---- | ---- |
| [`footage-source-and-processing.md`](./footage-source-and-processing.md) | 来源分类、处理流程、状态机、add/replace 接口契约与校验差异 |

## 与其它业务域的边界

- 语音/配音类素材（`SYNTHETIC_VOICE_OVER` / `RECORD_VOICE_OVER`）→ 见 [`../voice/`](../voice/)
- AIGC 生成的 footage（AI 分镜/动效视频）→ 见 [`../aigc/`](../aigc/)
- 本文聚焦「非 AIGC 的素材来源导入与 footage 轨绑定」
