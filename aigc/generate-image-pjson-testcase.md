# generate-image PJSON Test Case

> 接口：`POST /v3/project/{projectUuid}/scene/generate-image`
> 基于 snapshot 环境真实请求编写，验证 PJSON（PES）项目的图像生成流程。
> 关联：[pjson-impl.md](../../vme-pjson/pjson-impl.md) §8.1、[aigc-prompt-interfaces.md](./aigc-prompt-interfaces.md)

---

## 1. 被测接口

| 项 | 值 |
|----|-----|
| 端点 | `POST /v3/project/{projectUuid}/scene/generate-image` |
| Controller | `ProjectEditingV3Controller.generateAiImage` |
| Service | `ProjectSceneService.generateAIImage`（`@PjsonProjectOp("generateImage")`） |
| PES handler | `PjsonProjectEditService.generateImage` |
| 锁 | `@ProjectOperateLockRequired(PROJECT_OP_GROUP, needEditLock=true)` |

---

## 2. 前置条件

1. **目标 project 必须是 PJSON 项目**：`project.storage_type == 1`（PJSON）。否则走 NORMAL 路径（scene 从 DB 取），不触发本次改造逻辑。
   - 验证 SQL：`SELECT id, storage_type FROM project WHERE id = 1526998718761522090;`
2. **scene 已存在于 PES**：`sceneUuid=1527001102619361280` 能从 PES JSON 读到（`project_pjson` 有有效 `s3_link`）。
3. **该 scene 无未完成的 AIGC 记录**：`ai_generated_content_rel` 中该 scene 没有 `process_status` 非 FINISHED 的记录。
4. 当前用户对该 project 有 `PROJECT_CAN_UPLOAD` 权限。
5. 工作空间 credit / feePlan 支持图像生成。

> ⚠️ **部署阻塞**：当前 `v2-pjson` 分支有 3 个既有编译错误（`PjsonProjectEditService` animate handler 的 voice 类型 `SyntheticVoiceMetadata → RefAudio`），**整个分支无法 BUILD SUCCESS / 部署**。需先修复（见文末「遗留」）才能在 snapshot 实测。

---

## 3. 测试请求（snapshot）

```bash
curl -X POST 'https://api-snapshot.dev01.vislus.cn/v3/project/1526998718761522090/scene/generate-image' \
  -H 'Authorization: Bearer <TOKEN>' \
  -H 'Content-Type: application/json' \
  -d '{
    "sceneUuid": "1527001102619361280",
    "prompt": "Filmic documentary, respectful historical realism: wide archival-feeling establishing shot of the 1963 National Mall packed with a massive crowd under soft motivated daylight and subtle film grain, with clean sky negative space reserved for the three-line title overlay; cooler, slightly desaturated opening tension and restrained, dignified mood.",
    "refImages": [
      {
        "entityUuid": "1527001103139454976",
        "imageLink": "https://cn-northwest-1-dev-01.s3.cn-northwest-1.amazonaws.com.cn/m6/196/9/1527001102648721408/d8ae510b09452eea29330feab4f73851_std.png"
      }
    ],
    "opType": "edit"
  }'
```

---

## 4. PJSON 路径预期流程

请求进入后，`PjsonAspect` 拦截 `generateAIImage`：

```
1. extractProject → project 1526998718761522090
2. storageType == PJSON → 查找 handler("generateImage") → SHORT_CIRCUIT
3. PjsonProjectEditService.generateImage(project, reqBody):
   a. aclService.checkPermission(PROJECT_CAN_UPLOAD)
   b. user 校验
   c. sceneId = 1527001102619361280
   d. ★ pesScene = projectPjsonService.loadSceneDataFromPES(projectId, sceneId)   ← PES 读取（不走 DB project_scene）
   e. scene = buildPseudoScene(project, pesScene)                                  ← 构造伪 ProjectScene（含 duration）
   f. workspace = workspaceService.getById(getWorkspaceIdByDomain(project))
   g. aspectRatio 默认值兜底（reqBody 未传时按 videoFormat 推导）
   h. 校验无未完成 rel（aiGeneratedContentService.listRelsBySceneId）
   i. ★ 新建 asset：mediaAssembleService.addAigcVislaMedia(IMAGE) + projectAssetService.saveAigcAsset
   j. refImages → projectAssetService.checkAndGetRefImageAsset → DB project_asset
   k. ★ saveAndAssembleAigcData(AigcParamBo.from(reqBody), user, voice=null, ...)   ← voice=null（图像不需要配音）
   l. ★ afterCommit: createImageFromAigc(workspace.feePlan, assembleBo, null)       ← 调 AI + 状态同步
4. Controller 用返回的 assembleBo.asset.id 调 fetchAssetInfo → 返回 ProjectAssetVo
```

> 与 NORMAL 路径的唯一差异：**步骤 d**（scene 从 PES 取，而非 DB `findAndAssertScene`）。其余 asset/aigc/rel/AI 调用链完全一致。

---

## 5. 验证点

### 5.1 响应
- HTTP 200，返回 `Result<ProjectAssetVo>`，`code=0`。
- `data.assetUuid` 为新生成的 asset id（非入参 refImage 的 entityUuid）。

### 5.2 DB 落库（PJSON 专属验证：不应写 project_scene/project_timeline）
| 表 | 期望 |
|----|------|
| `project_asset` | 新增 1 条，`media_type=IMAGE`，`asset_type=AI_GENERATE`，`process_status` 由 `UPLOADING`→`ACTIVE`/`FAILED` |
| `ai_generated_content` | 新增 1 条，`media_type=IMAGE`，`status` 同步迁移 |
| `ai_generated_content_rel` | 新增 1 条，关联 project/scene/asset/aigc，`process_status` 同步 |
| `project_scene` | **无变更**（PJSON 下 scene 在 PES，不回写 DB） |
| `project_timeline` | **无变更** |

### 5.3 PES / AI 侧
- `aiGeneratedContentService.createImageFromAigc` 被调用（afterCommit）。
- 失败时：asset/aigc 标 `FAILED`；成功时：标 `UPLOADING`，rel 标 `UPLOADING`。

### 5.4 日志关键词（排障）
- `[PjsonAspect] SHORT_CIRCUIT op='generateImage'`
- `loadSceneDataFromPES: parsed N scenes` / `getAllScenes`
- `createImageFromAigc`

---

## 6. 异常 / 边界用例

| # | 场景 | 构造方式 | 预期 |
|---|------|---------|------|
| 1 | 非 PJSON 项目 | `project.storage_type != 1` | 走 NORMAL `generateAIImage`（scene 从 DB），不进 handler |
| 2 | scene 不在 PES | sceneUuid 在 PES JSON 中不存在 | `ApiException(PROJECT_SCENES_NOT_FOUND)` |
| 3 | sceneUuid 非法 | 传非数字 / 空 | `ApiException(ILLEGAL_PARAMS)` |
| 4 | 存在未完成 AIGC rel | 该 scene 有 process_status 非 FINISHED 的 rel | `ApiException(ILLEGAL_PARAMS, "There is not finished rel for this scene")` |
| 5 | 无 PROJECT_CAN_UPLOAD 权限 | 普通查看者调用 | ACL 拒绝 |
| 6 | refImages 无效 | entityUuid 不存在 / 非 IMAGE | `checkAndGetRefImageAsset` 抛错 |
| 7 | AI 调用失败 | mock createImageFromAigc 返回 false | asset/aigc 标 FAILED，rel 不更新为 UPLOADING |
| 8 | aspectRatio 缺省 | reqBody 不传 aspectRatio | 按 `project.videoFormat` 兜底推导 |

---

## 7. 遗留：voice 类型错误阻塞部署

`PjsonProjectEditService` 的 animate handler（行 533/578/602）调用 `saveAndAssembleAigcData` / `assembleAigcData` 时传 `SyntheticVoiceMetadata`，但方法签名要 `RefAudio`，导致整个分支编译失败。

**修复方向**（已确认 `RefAudio.from(SyntheticVoiceMetadata, voiceOver)` 存在）：
```java
// buildGlobalVoiceMap 返回 SyntheticVoiceMetadata，调用处转换为 RefAudio：
RefAudio refAudio = voiceMetadata != null
    ? RefAudio.from(voiceMetadata, <voiceOver>) : null;
```
其中 `voiceOver` 来自 scene transcript 发音矫正（参考 `ProjectSceneService.buildAiCharacterVoiceBo`）。修好后 animate + 本接口都能 BUILD SUCCESS 并部署 snapshot 实测。
