# Project Footage 来源与处理流程

> 范围：visla-api 编辑态「添加 footage」与「替换 footage」两个接口涉及的素材来源分类、`project_asset` 处理状态机与预处理链路
> 视角：业务来源 + 数据载体 + 状态机 + 接口契约
> 背景：footage（B-Roll）是 scene 的背景视频/图片素材，对应 `project_timeline.category = FOOTAGE`；其载体是 `project_asset`。不同来源的 footage，初始处理状态与是否需要预处理完全不同，这直接影响 add/replace 接口对素材的就绪校验逻辑

---

## 1. footage 在 scene 中的位置（背景）

```
project
  └─ scene（project_scene）
       ├─ footage timeline（category=FOOTAGE）   ← 本文焦点（背景视频/图片）
       │     └─ asset_id → project_asset
       ├─ voice over timeline（SYNTHETIC_VOICE_OVER / RECORD_VOICE_OVER）
       └─ bgm / transition / subtitle ...
```

footage timeline 通过 `asset_id` 指向一条 `project_asset` 记录。add/replace footage 接口的核心工作就是：**把指定来源的素材落成（或复用）一条 project_asset，再绑定到 scene 的 footage timeline 上**。

---

## 2. 涉及的数据载体

| 表 / 字段 | 作用 | footage 链路职责 |
| ----------- | ------ | ------------- |
| `project_asset.asset_type` | 素材来源类型（`ProjectAssetTypeEnum`） | 区分 uploaded / stock / clip / self_recording 等 |
| `project_asset.process_status` | 处理状态机（`AssetProcessStatusEnum`） | 标识素材是否就绪（INIT/DOWNLOADING/PROCESSING/COMPLETED/FAILED） |
| `project_asset.category_type` | 业务分类（`ProjectAssetCategoryEnum`） | footage 链路固定为 `FOOTAGE` |
| `project_asset.media_type` | 媒体类型（VIDEO/IMAGE/AUDIO） | 决定 duration/timeline 计算分支 |
| `project_asset.duration` | 素材时长（ms） | 预处理完成后才填充；uploaded/local 必须依赖它 |
| `project_asset.source_clip_id` + `provider_id` | stock 来源标识 | 用于 stock 素材去重（同 project 内不重复导入） |
| `project_asset.asset_link` / `original_link` | 可播放/原始文件地址 | premium stock 指向对方 preview/source link；free stock 预处理后指向自有 S3 |
| `project_timeline`（category=FOOTAGE） | 场景背景素材轨 | add 新建 / replace 替换其 asset_id |
| `stock_media` / `stock_audio` | stock 字典（源信息） | 导入时读取 duration / tags / link 做校正 |

---

## 3. footage 来源分类

### 3.1 统一入口：`importAssetOfAnyType`

add/replace footage 都通过同一个 facade 导入素材（`ProjectImportService.importAssetOfAnyType`，`ProjectImportService.java:55`），按 `EntityTypeEnum` 分发：

| entityType（请求参数） | 分发方法 | 业务含义 |
| ---------------------- | -------- | -------- |
| `PROJECT_ASSET` | `importAsset` | 已有素材（本地上传 / 录制 / AI 生成 / 跨项目库素材） |
| `CLIP` | `importClip` | 从 Clip（My Library / Recording 等）导入 |
| `STOCK_MEDIA` / `STOCK_MEDIA_PEXELS` / `STOCK_MEDIA_STORYBLOCKS` / `STOCK_MEDIA_GETTYIMAGES` | `importExistsStockMedia` | stock 视频/图片 |
| `STOCK_AUDIO` | `importStockAudio` | stock 音频（非 footage 主路径，见 §6.4） |

> 去重规则：stock 按 `(project_id, source_clip_id, provider_id)` 查已存在的 `IMPORTED_FROM_STOCK` 资产，命中则直接复用 assetId，不重复导入。

### 3.2 stock 的「付费 / 免费」划分

由 `StockProviderCodeEnum`（`StockProviderCodeEnum.java`）决定，`isPremium()` 只认两类：

| provider | id | 类别 | 是否需下载到自有 S3 |
| -------- | -- | ---- | ------------------ |
| `GETTYIMAGES`（Getty Images） | 5 | **premium** | 否（用对方 link） |
| `STORYBLOCKS`（Storyblocks） | 4 | **premium** | 否（用对方 link） |
| `PEXELS`（Pexels） | 3 | free | 是（需下载 + 预处理） |
| `PIXABAY`（Pixabay） | 1 | free | 是（需下载 + 预处理） |

> 关键业务差异：premium stock 受对方协议约束，**不下载到自有 S3**，入库即用对方 preview/source link，因此初始状态直接 `COMPLETED`；free stock 需要把源文件拉到自有 S3 再做标准化预处理，因此初始状态为「未完成」。

---

## 4. `process_status` 状态机

`AssetProcessStatusEnum`（`AssetProcessStatusEnum.java`）：

```
INIT(0) ─┐
DOWNLOADING(1) ─┐
UPLOADING(2) ───┤
PROCESSING(3) ──┼──→ COMPLETED(9)   ← 唯一「就绪可用」终态
PREPARING_REFAUDIO(4) ┤
PUBLISHING(6) ──┘
                 └──→ FAILED(12)    ← 处理失败终态
```

- `isFinished(status)` = `COMPLETED || FAILED`
- 对 footage 业务而言：**只有 `COMPLETED` 表示素材 duration/metadata 已就绪**；其余均为「处理中，未就绪」

---

## 5. 各来源的预处理流程

### 5.1 uploaded_from_local（本地上传）

```
用户上传文件到 S3（预签名）
  → ProjectAssetsV3Controller → ProjectAssetService.addUploadedAsset
     → addAssetForLocalUpload（ProjectAssetService.java:1512）
        asset_type = UPLOADED_FROM_LOCAL
        process_status = INIT                       ← 初始未就绪
        → preprocessUploadOneProjectAssetAfterTransactionCommit（事务提交后触发）
           → SQS → vcjs job 处理（转码/抽帧/提取 duration/metadata）
              → process_status = COMPLETED
```

**特征**：素材的 `duration` 只有预处理完成后才准确。因此业务上强制要求 **uploaded_from_local 必须 `COMPLETED` 才允许加入/替换 footage**，否则会因 duration 为 null 引发 NPE 或时间计算错误。

### 5.2 free stock（Pexels / Pixabay 视频·图片）

```
importAssetOfAnyType（entityType=STOCK_MEDIA_PEXELS/STOCK_MEDIA）
  → importExistsStockMedia（ProjectImportService.java:146）
     → PartnerStockService.importExistsStockMedia（PartnerStockService.java:825）
        VIDEO + free  → importExistsStockMediaFromEs（:534）
        IMAGE（任意）  → importExistsStockMediaFromEs（:534）
           asset_type = IMPORTED_FROM_STOCK
           process_status = DOWNLOADING              ← 初始未就绪
           → StockAsyncService.processImportStockMediaFromEsToProject（@Async）
              1. downloadStockMedia        从源下载到自有 S3
              2. updateProjectAssetAfterDownLoadStockMedia  回填 asset_link/duration
              3. preprocessImportStockToProjectAsset2
                   → sqsService.sendSqsToJobProcess（→ vcjs 标准化）
              4. updateProjectModifyAt
              5. sendImportStockNotification
                 → process_status = COMPLETED
```

**特征**：free stock 入库时 `duration` 可从 `stock_media` 字典获取（用于 add 时的时长校正），但 asset 本身的完整处理（下载、转码、标准化）是异步的。因此**允许在 `DOWNLOADING/PROCESSING` 状态下加入/替换 footage**——timeline 的 duration 可由请求参数或 stock 字典兜底，不强制依赖 asset 的预处理完成。

### 5.3 premium stock（Getty Images / Storyblocks 视频）

```
importExistsStockMedia
  VIDEO + premium → importStockMediaWithRestricted（PartnerStockService.java:637）
     → ProjectAssetService.addAssetForImportRestrictedStock（ProjectAssetService.java:899）
        asset_type = IMPORTED_FROM_STOCK
        asset_link / original_link = stockMedia.sourceLink   ← 直接用对方 link，不下载
        process_status = COMPLETED                            ← 入库即就绪
        duration / width / height 直接取自 stock_media
```

**特征**：受版权协议约束不下载到自有 S3，使用对方 link，**入库即 `COMPLETED`**，无需预处理，可立即用于 footage。

> 注意 IMAGE 的差异：即使是 premium（如 Getty 图片），在 `importExistsStockMedia` 路径下仍走 `importExistsStockMediaFromEs`（`DOWNLOADING` + 异步下载），与 premium video 的「入库即完成」不同。视频的 premium 特判只发生在 `mediaType == VIDEO` 分支。

### 5.4 stock audio（补充，非 footage 主路径）

`entityType=STOCK_AUDIO` 走 `importStockAudio`（`PartnerStockService.java:948`）：

- 默认 `process_status = INIT`
- storyblocks audio 在 `newItvReady` 开启时直接 `COMPLETED`（无需预处理）
- 未完成的走 `preprocessImportAssetFromStock`（事务提交后 SQS → vcjs）

> audio 一般作为 BGM / 旁白，不属于 `FOOTAGE` category，列出仅为说明同一 facade 的完整状态机。

---

## 6. add-footage 接口契约

### 6.1 入口

```
PUT /v3/{projectUuid}/scene/add-footage
  → ProjectEditingV3Controller.addFootage（ProjectEditingV3Controller.java:577）
     注解：@ProjectEditFlagAnnotation + @ProjectOperateLockRequired（编辑锁）
  → ProjectSceneService.addSceneFootage（ProjectSceneService.java:2590）
```

### 6.2 请求体（`ProjectSceneAddFootageReqBody`）

| 字段 | 说明 |
| ---- | ---- |
| `sceneUuid` | 目标场景 |
| `bRoll.entityType` | 素材来源（PROJECT_ASSET / STOCK_MEDIA* / CLIP） |
| `bRoll.entityUuid` | 素材/stock id |
| `bRoll.providerId` | stock provider（3=pexels, 5=getty…） |
| `bRoll.startTime` / `endTime` | 片段起止（ms） |
| `bRoll.timeOffsetInProject` | 在项目时间轴的偏移；为空则取 scene 偏移 |
| `signedId` / `sign` / `assetSignedId` / `assetSign` | 预生成 id 防并发（签名校验） |

### 6.3 核心流程

1. 权限校验（`PROJECT_CAN_EDIT`）+ 定位 scene + 过滤 intro/outro
2. 重叠检测：新 footage 与现有 footage timeline 不可重叠（`avoidOverlapSceneTimeline`）
3. **duration 校正**：
   - stock：查 `stock_media` 表，premium video 不能超过素材时长；其余按 md5 查 `media_source` 校正
   - `PROJECT_ASSET` 且 `endTime==null`：用 `asset.duration` 兜底，此时才检查处理状态（见 §6.4）
4. `importAssetOfAnyType` 落库/复用 asset
5. 新建 footage timeline，计算 scale / effectiveDuration，insert
6. text-2-video（无 voice over）联动 scene 时长；blank_scene 转 default
7. self_recording 特判：清掉合成/录制 voice over、音量置 1
8. 更新项目缩略图、调整图层、重新 assemble scene 返回

### 6.4 处理状态校验（关键，且与 replace 不一致）

```java
// 仅当 endTime==null 且 entityType==PROJECT_ASSET 时才进入
if (bRollAsset.getProcessStatus() == AssetProcessStatusEnum.PROCESSING
        && bRollAsset.getAssetType() == ProjectAssetTypeEnum.UPLOADED_FROM_LOCAL) {
    throw new ApiException(ApiMessage.PROJECT_HAS_NOT_YET_FINISHED_PROCESSING);
}
```

| 条件 | 行为 |
| ---- | ---- |
| stock（任意状态） | **不校验** process_status，允许未完成直接加入 |
| UPLOADED_FROM_LOCAL + `PROCESSING` | 拒绝（未完成处理） |
| UPLOADED_FROM_LOCAL + `INIT`/`UPLOADING` | **当前不拦截**（只拦了 PROCESSING） |
| 任意 asset + `FAILED` | **当前不拦截** |

---

## 7. replace-footage 接口契约

### 7.1 入口

```
POST /v3/{projectUuid}/scene/replace-footage
  → ProjectEditingV3Controller.replaceFootage（ProjectEditingV3Controller.java:625）
     注解：@ProjectEditFlagAnnotation + @ProjectOperateLockRequired（编辑锁）
  → ProjectSceneService.replaceFootage（ProjectSceneService.java:3207）
```

### 7.2 请求体（`ProjectSceneReplaceFootageReqBody`）

| 字段 | 说明 |
| ---- | ---- |
| `sceneUuid` | 目标场景 |
| `timelineUuid` | 可选；指定替换哪条 footage timeline，为空则取该 scene 唯一 footage timeline |
| `entityType` / `entityUuid` / `providerId` | 新素材来源（同 add） |
| `startTime` / `duration` | 可选；覆盖 timeline 起止 |

### 7.3 核心流程

1. 权限校验（`PROJECT_CAN_EDIT`）+ 定位 scene + 解析/兜底目标 timeline
2. `importAssetOfAnyType` 落库/复用新 asset
3. **处理状态校验**（见 §7.4）
4. 缩略图联动：旧 asset 曾作为缩略图来源时，用新 asset 更新
5. 按媒体类型重算 timeline 时间：
   - IMAGE：时长对齐 scene（voice 模板下对齐旧时长）
   - VIDEO：时长取 `min(asset.duration, reqBody.duration, scene.duration)`，并校验 ≥ 50ms
6. 重算 scale / coordOffset，speed 强制重置为 1.0
7. self_recording 特判：音量置 1、清合成/录制 voice over、联动 scene 时长
8. effectiveDuration 按「scene 内可用空间」封顶（position-aware cap）
9. blank_scene 转 default；AI 生成内容关联（`selectRelByScene`）；清旧 crop/animation
10. 重新 assemble scene 返回

### 7.4 处理状态校验（关键，且与 add 不一致）

```java
// 1) 任意来源，处理失败一律拒绝
if (projectAsset.getProcessStatus() == AssetProcessStatusEnum.FAILED) {
    throw new ApiException(ApiMessage.PROJECT_HAS_FAILED_PROCESSING);
}
// 2) uploaded_from_local 必须处理完成，其余状态一律拒绝
if (projectAsset.getProcessStatus() != AssetProcessStatusEnum.COMPLETED
    && projectAsset.getAssetType() == ProjectAssetTypeEnum.UPLOADED_FROM_LOCAL) {
    throw new ApiException(ApiMessage.PROJECT_HAS_NOT_YET_FINISHED_PROCESSING);
}
```

| 条件 | 行为 |
| ---- | ---- |
| 任意 asset + `FAILED` | 拒绝（处理失败） |
| UPLOADED_FROM_LOCAL + 非 `COMPLETED` | 拒绝（未完成处理） |
| UPLOADED_FROM_LOCAL + `COMPLETED` | 允许 |
| stock（IMPORTED_FROM_STOCK，任意状态） | **允许**（不强制 COMPLETED，允许 DOWNLOADING/PROCESSING 时替换） |
| self_recording / AI 生成等 | 不在此两条件内，按各自状态推进（self_recording 见 §7.3 步骤 7） |

---

## 8. add 与 replace 的校验差异与风险点

### 8.1 校验严格度对比

| 场景 | add-footage | replace-footage |
| ---- | ----------- | --------------- |
| FAILED asset | 不拦截 | **拦截** |
| UPLOADED_FROM_LOCAL 未完成 | 仅拦 `PROCESSING` | 拦所有非 `COMPLETED` |
| 校验触发前提 | 仅 `endTime==null && PROJECT_ASSET` 才检查 | 导入后无条件检查 |
| stock 未完成 | 允许 | 允许 |

### 8.2 业务根因

- **stock 允许未完成就绪**：stock 的 duration 可从 `stock_media` 字典或请求参数获取，timeline 时间计算不依赖 asset 预处理完成；且 premium stock 本就入库即完成，free stock 异步处理不影响加入场景。
- **uploaded_from_local 强制完成**：其 duration 仅在预处理后准确，未完成时为 null，会直接导致 timeline 时间计算 NPE 或错误。

### 8.3 风险点（建议关注）

1. **add 与 replace 校验不一致**：同一份未完成 uploaded asset，add 可能放行（INIT/UPLOADING 不拦）、replace 会拒绝。前端交互上可能出现「能 add 却不能 replace」的困惑，建议统一为 replace 的严格策略。
2. **add 不拦 FAILED**：add 对 FAILED 素材完全不拦截（且只在特定分支才检查），可能把失败素材加入场景。
3. **replace 存在潜在 NPE**：`replaceFootage` 中 `projectAssetService.getById(assetId, ACTIVE)` 若返回 null，后续 `projectAsset.getProcessStatus()` 会 NPE（add 用 `getActiveById` 同样存在）。正常路径下 import 后 asset 必然存在，属于防御性缺陷。
4. **premium IMAGE 与 premium VIDEO 处理不同**：premium video 入库即完成，premium image 仍走异步下载，若前端误以为「premium 都立即可用」可能对 image 产生时长/链接预期偏差。

---

## 9. 代码索引

| 关注点 | 位置 |
| ------ | ---- |
| 导入 facade | `ProjectImportService.importAssetOfAnyType`（`ProjectImportService.java:55`） |
| stock 导入分发 | `PartnerStockService.importExistsStockMedia`（`PartnerStockService.java:825`） |
| free stock 异步链路 | `StockAsyncService.processImportStockMediaFromEsToProject`（`StockAsyncService.java:74`） |
| stock 预处理发 SQS | `PreprocessService.preprocessImportStockToProjectAsset`（`PreprocessService.java:1812`） |
| 本地上传建资产 | `ProjectAssetService.addAssetForLocalUpload`（`ProjectAssetService.java:1512`） |
| premium stock 建资产 | `ProjectAssetService.addAssetForImportRestrictedStock`（`ProjectAssetService.java:899`） |
| add-footage 服务 | `ProjectSceneService.addSceneFootage`（`ProjectSceneService.java:2590`） |
| replace-footage 服务 | `ProjectSceneService.replaceFootage`（`ProjectSceneService.java:3207`） |
| 状态枚举 | `AssetProcessStatusEnum` / `ProjectAssetTypeEnum` / `StockProviderCodeEnum` |

> 本文档仅描述 visla-api 侧的来源分类、状态机与接口契约；vcjs 侧的标准化 job 处理（收到 SQS 后如何把 `process_status` 推进到 `COMPLETED`）属于 vcjs 实现细节，不在本文范围。
