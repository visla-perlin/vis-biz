# Clip 变换（Rotation / Flip）— API-1281

> 需求：客户（Eventage / Danny）常导入方向错误的素材，需要在**非场景编辑器**的 Clip 菜单中直接旋转/翻转。
> 父 issue：VIS-3050（Add "Rotation" option to Clip Menu）；产品补充：**clip 包含 image，也要支持**。
> 本文：业务契约 + 跨仓库改动清单。链路背景见 [`preprocess-actions.md`](./preprocess-actions.md)。

---

## 1. 业务规则

- 前端 Clip 菜单提供 4 个操作：Flip Horizontal / Flip Vertical / Rotate Left / Rotate Right，**confirm 模式**：前端本地维护累积状态（angle 对 360 取模、flip 布尔翻转），点 confirm 才发一次全量请求。
- 一次请求 = 一次复合变换（rotate + flip 任意组合），参数是相对**原始素材**的最终状态，从原媒体一次转码，无画质累积损失。
- 支持媒体：**video + image**（reformat 仅 video，transform 是 image 第一个编辑类动作）。
- 结果语义：破坏性编辑，与 reformat 一致 —— 深拷贝媒体后在新媒体上执行，clip 置 `PROCESSING`，完成前禁止其他编辑。
- 90°/270° 旋转宽高互换，由后继 METADATA 动作重取宽高回写 clip / visla_media。

## 2. 变换语义（顺序契约，写死）

**结果 = flip(rotate(source))，先旋转、后翻转。**

- `rotate`：0 / 90 / 180 / 270，**顺时针为正**（VME 实现须与此一致，正角度旋向是与 VME 的对齐点）。
- `flip.horizontal`：水平翻转（左右镜像）；`flip.vertical`：垂直翻转（上下镜像）。
- rotate 与 flip **不可交换**（flipH∘rot90 ≠ rot90∘flipH），复合顺序必须固定；前端预览渲染顺序与此一致（先 rotation 再 flip）。
- 前端按钮映射：Rotate Right → rotate=90；Rotate Left → rotate=270；Flip H/V → 对应 flip 位。
- **rotate=0 且 flip 全 false = 无变换，API 直接拒绝（400），不发送 SQS**。

## 3. 接口契约（visla-api）

```
POST /v3/clip/{clipUuid}/transform
```

请求体（`TransformClipReqBody`）：

| 字段 | 类型 | 必填 | 校验 |
| ---- | ---- | ---- | ---- |
| rotate | Integer | 是 | 0/90/180/270，默认 0 |
| flip | Flip | 否 | `{horizontal: bool, vertical: bool}`，默认全 false |
| duplicate | boolean | 否 | **Keep a Copy of Original**，与 cut 请求的 `duplicate` 一致，**默认 false**，前端按 checkbox 状态显式传（见下） |

**Keep a Copy of Original**：变换前先浅拷贝一份原 clip（标题加 " copy"，引用旧 media，保留原始内容），变换作用于原 clip id。与 reformat 的 `duplicate` 语义一致：

- new media clip：`vislaClipService.shallowDuplicateV3`
- 旧 clip：`clipService.doDuplicateClipV3`
- 原为 personal stock（COMPLETED/COMPLETED_FOR_REVIEW）时发 `CLIP_STOCK_REPLACE` 事件把 stock 转给副本，原 clip 置 `NOT_PRIVATE_STOCK`

响应：`ClipEditVo { clipUuid, clipStatus(PROCESSING), clipLink, capabilities }`（与 reformat 一致）。

前置校验（service 层）：

- clip 存在；`clipType ∈ {VIDEO, IMAGE}`
- **composite video clip 直接拒绝（`ACTION_DENIED`），不支持 transform**（其快速编辑走 compositeVideoJson 体系，与 transform 链路不兼容）
- `clipStatus ∉ {PROCESSING, PUBLISHING, FAILED}`
- ACL：`CLIP_CAN_USE_QUICK_EDIT`
- 至少一项变换生效（rotate ≠ 0 或任一 flip true），否则 `ILLEGAL_PARAMS("no transform to apply")`

注解组合：`@SecureUrl(tokenValidate)` + `@WorkspaceStorageStatsRequired` + `@DomainOperateLockRequired(CLIP_EDIT_GROUP, CLIP_TRANSFORM, clipUuid, autoRelease, lockTime=10)` + `@EntityParentRecordAnnotation(TRANSFORM, CLIP, CLIP)`。

## 4. Action 契约（transcoder）

- action 名：`transform`，id **49**（`PreprocessActionEnum.TRANSFORM`，visla-api 与 vcjs 双侧同步）。
- 请求参数块（`PreprocessEntityReqBody.Transform`，`data[].transform`）：

| 字段 | 类型 | 说明 |
| ---- | ---- | ---- |
| clip_link | String | 产物目标地址（API 预生成，深拷贝后的新媒体 standard link） |
| rotate | int | 0/90/180/270 顺时针，0 = 不旋转 |
| flip | Flip | `{horizontal, vertical}`，全 false 时仅 rotate 生效；flip-only 时 rotate=0 照传 |

`media_type` 传 `video` 或 `image`。

- 结果块（`PreprocessEntityBo.Transform`，`data[].transform`）：`{ code }`，0 = 成功。
- 执行语义（VME）：**flip(rotate(source))**，视频旋转保留音轨；图片同理。

## 5. 处理流程

```
API: transformClip
  ├─ 校验(类型/状态/ACL/参数有效性；composite 直接拒绝) + DomainOperateLock
  ├─ deepCopyVislaMediaForReformat(深拷贝 visla_media 及派生物，指向新 S3 key)
  ├─ clip 清空派生 link、置 PROCESSING
  ├─ SQS: actions=["transform"], data[].transform={clip_link, rotate, flip}
  └─ media_process_result.transform = DOING
transcoder: ffmpeg transpose+hflip/vflip → 产物上传 clip_link → 回发结果
vcjs: TranscodingProcessHandler（transform 属编辑类 action，路由 handleClipEditingResult）
  └─ handleVislaClipEditingResult → ClipVideo / ClipImageProcessHandler.handlerEditingTranscodingResult
      ├─ code=0: transform=SUCCESS；nextActions=METADATA+THUMBNAIL(+video: PLAYBACK)
      ├─ code≠0: transform=FAILED，通知失败
      └─ hasMoreTranscoding → 强制 PROCESSING 并再发后继预处理
          └─ 后继完成 → 宽高回写(90/270 互换)、缩略图/m3u8 重生成 → COMPLETED + 通知 + ES
```

状态机（`getVislaProcessStatusEnum` / `getEditProcessStatus`）：transform 列纳入编辑维度，DOING/TODO 阻塞 COMPLETED，INIT 不阻塞（保护存量上传链路）。

## 6. 跨仓库改动清单

### visla-api（API-1281，worktree `visla-api-API-1281`）

| 位置 | 改动 |
| ---- | ---- |
| `enums/PreprocessActionEnum` | `TRANSFORM(49, "transform")` |
| `request/PreprocessEntityReqBody` | 内部类 `Transform{clipLink, rotate, flip}` + `Flip{horizontal, vertical}` + `data[].transform` |
| `enums/UserOperateTypeEnum` | `CLIP_TRANSFORM(2027, "clip_transform")` |
| `enums/EntityOperationEnum` | `TRANSFORM("transform", 8)` |
| `controller/v3/TeamspaceClipController` | `POST /{clipUuid}/transform`（quick edit 注解组合） |
| `service/ClipQuickEditService` | `transformClip(...)`：校验 + duplicate 副本（Keep a Copy of Original）+ 深拷贝 + PROCESSING + 发 SQS + 统计 + quota 事件 |
| `request/TransformClipReqBody`（新增） | `rotate` + `Flip` + `duplicate`（默认 true），含 `hasAnyTransform()` 校验辅助 |
| `service/VislaMediaProcessService` | `preprocessTransformClip[AfterTransactionCommit]`：组装参数块、transform 置 DOING |
| `service/MediaProcessResultService` | 构造方法补 `transform = INIT` |
| `domain/MediaProcessResult` + Mapper XML | 加 `transform` 列（resultMap / Base_Column_List / insert / batchInsert / insertSelective / updateById / updateByIdSelective / updateByEntityIdSelective） |
| `db/migration/v0.3.6.5` | `media_process_result` 加 `transform`；`rpt_quick_edit` 加 `transform_times` |
| `enums/QuickEditEnum` + `RptQuickEdit` + Mapper | `TRANSFORM` → `transformTimes` 记次 |

### vcjs（API-1281，worktree `vcjs-API-1281`）

| 位置 | 改动 |
| ---- | ---- |
| `enums/PreprocessActionEnum` | `TRANSFORM(49, "transform")`（同步） |
| `process/bo/PreprocessEntityBo` | 内部类 `Transform{code}` + `data[].transform` |
| `store/entity/MediaProcessResult` + Mapper XML | 加 `transform` 列（同步共享表） |
| `service/MediaProcessResultService` | 构造方法补 `transform = INIT` |
| `process/prepare/TranscodingProcessHandler` | 编辑类 action 路由判断加入 `TRANSFORM`（走 handleClipEditingResult） |
| `process/handler/ClipVideoProcessHandler` | `handlerEditingTranscodingResult` 加 `case TRANSFORM`（SUCCESS + METADATA/THUMBNAIL/PLAYBACK 后继）；`getEditProcessStatus` / `getVislaProcessStatusEnum` 纳入 transform |
| `process/handler/ClipImageProcessHandler` | `handlerEditingTranscodingResult` 从 null 实现 `case TRANSFORM`（SUCCESS + METADATA/THUMBNAIL 后继）；`getEditProcessStatus` 实现；`getVislaProcessStatusEnum` 纳入 transform |

### transcoder / VME（外部依赖，非本仓库）

实现 `transform` 动作：`flip(rotate(source))` 固定顺序、顺时针为正、视频保留音轨，产物上传 `clip_link`，按结果契约回发 `{code}`。**未实现前链路卡 DOING，需对齐排期。**

## 7. 风险与边界

- **VME 排期与旋向对齐**（顺时针为正）是最大外部依赖；上线顺序需 transcoder 先行或同步。
- image clip 是第一个编辑类动作：`handlerEditingTranscodingResult` 此前返回 null（会 NPE），本次已实现；回归覆盖 image transform 成功/失败两条路径。
- 老数据 `transform` 列默认 0（INIT），不阻塞存量 clip 的 COMPLETED 判定。
- composite video clip 直接拒绝（`ACTION_DENIED`）；如未来需要支持，需走 compositeVideoJson 体系单独设计，不复用本期链路。
- rpt 统计记 `transform_times` 单列（flip/rotate 复合算一次），与 action 一一对应。
