# Agnes 视频生成本地 WebUI 需求文档

版本：v0.1 需求梳理与演示版范围
日期：2026-06-30
参考文档：

- Agnes Overview: https://wiki.agnes-ai.com/en/docs/overview.md
- Agnes Video V2.0: https://wiki.agnes-ai.com/en/docs/agnes-video-v20.md
- Agnes Common Error Codes: https://wiki.agnes-ai.com/en/docs/code.md

## 1. 项目目标

开发一款本地运行的 Agnes 视频生成 WebUI，让用户能够同时配置多个 Agnes API Key，并把同一个或多个提示词自动分发到不同 Key 上并行生成视频。系统需要自动创建异步视频任务、轮询任务状态、拉取最终视频 URL，并在单文件演示版中展示完成结果和打开视频。

第一阶段先交付一个单文件 HTML 演示版，用于验证界面布局、任务分组、并发/串行调度、队列状态、完成结果展示等交互是否符合预期。

## 2. 官方 API 摘要

Agnes Video V2.0 是异步任务模型：

- 创建任务：`POST https://apihub.agnes-ai.com/v1/videos`
- 推荐查询：`GET https://apihub.agnes-ai.com/agnesapi?video_id=<VIDEO_ID>`
- 兼容查询：`GET https://apihub.agnes-ai.com/v1/videos/<TASK_ID>`
- 认证方式：`Authorization: Bearer YOUR_API_KEY`
- 模型名：`agnes-video-v2.0`
- 任务状态：`queued`、`in_progress`、`completed`、`failed`
- 完成结果字段：`remixed_from_video_id` 为最终视频 URL

核心创建参数：

- `model`：必填，固定为 `agnes-video-v2.0`
- `prompt`：必填，视频提示词
- `image`：可选，单图生视频 URL
- `mode`：可选，生成模式，例如 `ti2vid` 或 `keyframes`
- `extra_body.image`：可选，多图或关键帧图 URL 数组
- `extra_body.mode`：可选，关键帧模式为 `keyframes`
- `width`、`height`：可选，系统会标准化到支持分辨率
- `num_frames`：可选，必须小于等于 `441`，且符合 `8n + 1`
- `frame_rate`：可选，范围 `1-60`
- `num_inference_steps`：可选，推理步数
- `seed`、`negative_prompt`：可选

参数标准化：

- 官方支持标准分辨率档位：`480p`、`720p`、`1080p`。
- 官方列出的宽高比：`16:9`、`9:16`、`1:1`、`4:3`、`3:4`。
- 当提交的 `width`、`height` 或宽高比与模型支持规格不完全匹配时，Agnes 会映射到最接近的标准输出尺寸。
- 展示任务信息时以响应中的 `size`、`seconds` 为准。

## 3. 用户场景

1. 用户只有 1 个 API Key，需要一次生成多个视频。系统应自动串行执行，直到全部完成。
2. 用户有多个 API Key，需要同一个提示词生成多条视频。系统应把任务分发到不同 Key，独立并发执行。
3. 用户希望不同 Key 的任务分组清楚，可同时观察每个 Key 的队列、运行、完成和失败状态。
4. 用户需要自动轮询并拉取 Agnes 返回的视频 URL，不想反复手动刷新任务结果。
5. 用户希望完成的视频进入完成列表，能直接打开视频查看。

## 4. 功能需求

### 4.1 API Key 管理

- 支持新增、编辑、删除 API Key。
- 每个 Key 支持自定义名称、颜色、启用/停用状态。
- Key 显示时默认脱敏，只显示首尾少量字符。
- 每个 Key 维护独立队列、运行状态、历史任务、错误信息。
- 多 Key 场景默认每个 Key 同时只跑 1 个 Agnes 视频任务，避免单 Key 被 429 限流。
- 后续可增加每个 Key 的并发上限、备注、可用额度、失败次数统计。

### 4.2 提示词与任务配置

- 支持文本生视频。
- 支持图生视频、多图生视频、关键帧模式。
- 支持负面提示词。
- 支持官方分辨率档位、官方宽高比、帧数、帧率、Seed、推理步数、生成数量等参数。
- 分辨率档位仅提供 `480p`、`720p`、`1080p`。
- 宽高比仅提供 `16:9`、`9:16`、`1:1`、`4:3`、`3:4`。
- 对 `num_frames` 执行前端校验：小于等于 `441` 且符合 `8n + 1`。
- 对 `frame_rate` 执行前端校验：`1-60`。
- 对图生视频 URL 做基础 URL 格式检查；图生视频至少 1 个 URL，多图和关键帧模式至少 2 个 URL。

### 4.3 批量生成与调度

- 当启用 Key 数量为 0 时，禁止创建任务并提示用户配置 Key。
- 当启用 Key 数量为 1，生成数量大于 1 时，自动串行执行。
- 当启用 Key 数量大于 1 时，按 Key 分组并行执行。
- 分配策略第一版采用轮询分配；后续可升级为最少排队优先。
- 每个任务生命周期：
  - 本地排队
  - 创建 Agnes 任务
  - 远端排队
  - 生成中
  - 已完成
  - 失败
- 支持暂停队列、重试失败任务、清空已完成任务。

### 4.4 自动轮询

- 创建任务成功后保存 `task_id`、`video_id`。
- 优先使用 `video_id` 查询结果。
- 轮询间隔建议：
  - 初始：5 秒
  - 连续排队：10-15 秒
  - 429/503/504：指数退避，最高 60 秒
- 遇到 `completed` 后读取 `remixed_from_video_id` 并进入完成视频列表。
- 遇到 `failed` 或错误对象时记录失败原因。

### 4.5 完成结果管理

- Agnes 任务完成后进入“完成视频”列表。
- 完成卡片展示任务编号、Key 分组、服务端返回尺寸、帧数、帧率、时长和视频标识。
- 有视频 URL 时提供“打开视频”按钮，仅负责打开 Agnes 返回的远程视频地址。
- 演示版不提供任何下载入口，不触发浏览器下载，不选择或展示本地保存位置。
- 已完成任务保留在历史记录中，可通过“清理完成”移除。

浏览器安全限制说明：

- 纯浏览器单文件 HTML 不能静默写入任意本地路径。
- 当前演示版不使用 File System Access API，不尝试读写本地视频文件。
- 纯单文件 WebUI 不提供网页外助手，不调用 localhost 服务，不依赖 Python/Node 常驻进程。

### 4.6 本地存储

- 演示版使用 `localStorage` 保存非敏感 UI 配置和模拟任务。
- 正式浏览器版可使用 IndexedDB 保存任务历史。
- 正式桌面版建议：
  - API Key 存入系统 Keychain/Windows Credential Manager 或加密配置文件。
  - 任务历史存本地 SQLite/JSON。
  - 保存位置、命名模板、调度设置持久化保存。

### 4.7 错误处理

需覆盖 Agnes 常见错误：

- `400`：参数错误，提示检查 JSON、视频参数、图像 URL。
- `401`：Key 无效或过期，标记对应 Key 需处理。
- `402`：额度不足，暂停该 Key 队列。
- `403`：权限不足，暂停该 Key 队列。
- `404`：任务或视频不存在，允许切换 legacy 查询重试。
- `408`、`499`、`504`、`524`：超时，按退避策略重试。
- `409`：疑似重复提交，避免自动重复创建。
- `422`：参数值不合法，展示具体字段。
- `429`：限流，降低并发并退避。
- `500`、`502`、`503`、`520`、`522`：服务或网络问题，重试后仍失败则保留任务状态。

## 5. 非功能需求

- 本地优先：除 Agnes API 与视频 URL 外不依赖外部服务。
- 易打开：演示版为单 HTML 文件；正式版提供 macOS 与 Windows 一键启动包。
- 响应式：适配常见桌面窗口和移动窄屏。
- 可观察：每个 Key 的排队、运行、完成、失败状态一眼可见。
- 可靠：页面刷新后应能恢复未完成任务并继续轮询。
- 安全：避免在日志、截图、导出文件中明文泄露 API Key。
- 可扩展：后续可接入更多 Agnes 视频模型或其他 OpenAI 风格视频接口。

## 6. 数据模型草案

### ApiKeyProfile

```json
{
  "id": "key_local_id",
  "name": "Key A",
  "apiKeyEncrypted": "...",
  "maskedKey": "sk-...abcd",
  "color": "#0f766e",
  "enabled": true,
  "concurrency": 1,
  "createdAt": 1780457477,
  "lastError": null
}
```

### VideoJob

```json
{
  "id": "job_local_id",
  "batchId": "batch_local_id",
  "keyId": "key_local_id",
  "prompt": "A cinematic shot...",
  "params": {
    "model": "agnes-video-v2.0",
    "width": 1280,
    "height": 720,
    "num_frames": 121,
    "frame_rate": 24,
    "num_inference_steps": null,
    "seed": null,
    "negative_prompt": ""
  },
  "status": "queued",
  "progress": 0,
  "taskId": null,
  "videoId": null,
  "videoUrl": null,
  "error": null,
  "createdAt": 1780457477,
  "updatedAt": 1780457477
}
```

### Batch

```json
{
  "id": "batch_local_id",
  "prompt": "A cinematic shot...",
  "total": 6,
  "createdAt": 1780457477,
  "scheduler": "round_robin",
  "status": "running"
}
```

## 7. 调度规则

1. 读取所有 `enabled=true` 的 Key。
2. 生成 N 个本地任务。
3. 按 round-robin 分配：
   - Key A：任务 1、4、7
   - Key B：任务 2、5、8
   - Key C：任务 3、6、9
4. 每个 Key 内按队列顺序串行执行。
5. 若某 Key 出现 401/402/403，可暂停该 Key，并把未开始任务转移到其他可用 Key。
6. 若所有 Key 都暂停或失败，批次进入 `blocked` 状态，等待用户处理。

## 8. 正式版技术方案建议

### 方案 A：Tauri 桌面应用，推荐

- 前端：HTML/CSS/TypeScript WebUI。
- 后端：Rust/Tauri 命令负责 API 请求、轮询、系统凭据存储和可选文件能力。
- 优点：安装包小，跨 macOS/Windows，能使用系统凭据存储。
- 缺点：需要打包流程。

### 方案 B：Electron 桌面应用

- 前端：WebUI。
- 后端：Node 主进程负责 API、任务恢复和可选文件能力。
- 优点：生态成熟，开发快。
- 缺点：安装包较大。

### 方案 C：单 HTML 浏览器版

- 优点：最轻量，双击即可打开。
- 缺点：CORS、API Key 安全、本地目录写入受浏览器限制。
- 适合作为演示版或轻量模式；若未来需要稳定本地保存，建议另做桌面壳能力。

## 9. 分阶段交付

### v0.1 演示版

- 单 HTML 文件。
- 可新增/删除模拟 Key。
- Key 分组明显。
- 同提示词批量创建模拟任务。
- 多 Key 并发，单 Key 串行。
- 模拟进度与完成状态。
- 完成视频列表展示结果，并支持打开视频链接。
- 不调用真实 API。

### v0.2 API 联调版

- 接入真实 `POST /v1/videos`。
- 接入 `GET /agnesapi?video_id=...`。
- 支持创建、轮询、完成状态展示。
- 不提供下载入口，仅展示 Agnes 返回的视频 URL 并支持打开查看。

### v0.3 可靠性增强版

- 增强真实 API 正向 UI 验证。
- 增加多 Key 真实并发压测。
- 增加任务恢复、失败重试和轮询退避配置。

### v1.0 正式本地桌面版

- Tauri/Electron 打包。
- API Key 安全存储。
- 任务恢复、失败重试、Key 暂停与转移。
- macOS 与 Windows 一键打开。

## 10. 演示版验收标准

- 双击 HTML 可打开。
- 没有任何真实 Key 也可以查看和操作模拟流程。
- 新增 1 个 Key，生成 3 个视频时表现为串行。
- 新增 3 个 Key，生成 6 个视频时表现为三路并行。
- 每个 Key 的任务列表独立显示。
- 刷新页面后保留 Key 和近期任务状态。
- 完成任务进入完成视频列表，并提供打开视频入口。

## 11. 正式版验收标准

- 输入真实 Agnes API Key 后可以成功创建视频任务。
- 同一提示词可以按数量拆分为多个任务。
- 多 Key 可以并行生成，单 Key 可以自动串行。
- 任务完成后自动获取视频 URL。
- 应用重启后能记住历史任务。
- 遇到 429/503/504 等错误可自动退避重试。
- 任一 Key 失效不会影响其他 Key 队列。

## 12. 真实 API 验证补充

日期：2026-07-02

使用用户提供的 Agnes API Key 完成 1 次最小真实任务验证。密钥仅通过无回显 stdin 传入测试脚本，未写入代码、报告或命令参数。

验证结果：

- `POST https://apihub.agnes-ai.com/v1/videos` 创建任务成功，HTTP `200`。
- 创建响应返回 `task_id` 与 `video_id`。
- `GET https://apihub.agnes-ai.com/agnesapi?video_id=...&model_name=agnes-video-v2.0` 轮询成功。
- 任务约 1 分 50 秒后从 `in_progress` 变为 `completed`。
- 完成响应返回 `remixed_from_video_id`。
- 最终视频 URL 可 `HEAD` 与 `GET`，文件类型为 `video/mp4`。
- Agnes API 对 `/v1/videos` 的浏览器预检返回 HTTP `204`，`Access-Control-Allow-Origin: *`，单 HTML 直接 `fetch` 从 CORS 角度可行。

对方案的影响：

- 现有演示版的任务调度、Key 分组、轮询和完成结果展示设计成立。
- 查询接口不在 `/v1` 下，代码需单独配置 `https://apihub.agnes-ai.com/agnesapi`，避免拼成 `/v1/agnesapi`。
- API 会标准化分辨率，本次请求 `1152x768`，返回实际 `1088x832`，UI 必须显示服务端返回的 `size`。
- 进度可能长时间停留在同一值，本次 `progress=30` 持续多轮后直接到 `100`，轮询逻辑不能因进度不动误判失败。
- 浏览器 CORS 可行不代表自定义目录静默保存可行，因此当前演示版已移除下载链路。

详细验证报告见：`outputs/agnes-api-validation-report.md`。

## 13. 可靠性验证补充

日期：2026-07-03

完成单文件 WebUI 的 P0 真实联调能力改进，并进行了浏览器与真实 API 组合验证。

本轮新增：

- “模拟模式 / 真实 API”切换。
- 真实 `POST /v1/videos` 创建任务。
- 真实 `GET /agnesapi?video_id=...&model_name=agnes-video-v2.0` 轮询。
- API Key 仅保存在页面内存，`localStorage` 只保存脱敏标识。
- 请求超时、网络失败、429/5xx 等可重试错误处理。
- 真实模式完成后展示 Agnes 返回的视频 URL，并提供打开视频入口。

验证结果：

- 模拟模式浏览器真机通过：6 个模拟任务全部完成并进入完成视频列表。
- 真实 API 浏览器负向路径通过：假 Key 返回 401，UI 正确失败，不保存 Key 原文。
- 真实 API 第二轮正向探针通过：创建、轮询、视频 URL 获取成功。
- 第二轮真实轮询中出现 1 次临时网络错误后恢复，已将浏览器 `status=0` 网络失败纳入重试范围。

详细记录见：`outputs/agnes-reliability-validation-20260703.md`。
