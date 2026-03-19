---
name: aivo-download-split
description: |
  先下载视频，再自动按场景拆解为多个镜头片段。
  触发关键词（匹配任意一个）：
  - "下载并拆解"、"下载视频然后按场景拆分"、"从链接拆解视频"
  - "下载并分割场景"、"下载这个视频然后拆分"
  - 用户给了一个视频链接（抖音/快手/B站等）并要求"拆解"、"拆分"
  整合了视频下载和场景拆解两个功能。
  需要容剪客户端已启动（端口 7073）。
metadata:
  openclaw:
    emoji: "🔗"
    requires:
      bins:
        - curl
    os:
      - darwin
      - linux
      - win32
---

# 视频下载 + 场景拆解

> **严格约束（必须遵守）：**
> 1. **只使用本文档中明确列出的 API 路径**，绝对不要猜测或构造文档中没有的 URL
> 2. 基础地址固定为 `http://localhost:7073`
> 3. `/api/health` 返回 `{"status":"ok"}` 即代表服务可用
> 4. 视频下载路径是 `/api/video-download`（连字符），场景拆解路径是 `/api/split-scene/`
> 5. Windows 路径在 JSON 中必须用正斜杠 `/`
> 6. 健康检查：`/api/video-download/health`（视频下载），`/api/split-scene/health`（场景拆解）

给视频分享链接，自动下载到本地，然后用 FFmpeg 检测场景切换点，将视频拆分为多个独立镜头片段导出。

---

## 输出路径规则

> **如果用户没有指定输出路径，自动使用桌面下的 `aivo_video/split_scene` 目录并告知用户。**

- 用户指定了路径 → 使用用户指定的路径
- 用户未指定 → 动态拼接默认路径：
  - **Windows**: `%USERPROFILE%/Desktop/aivo_video/split_scene`（即 `$env:USERPROFILE/Desktop/aivo_video/split_scene`）
  - **macOS/Linux**: `$HOME/Desktop/aivo_video/split_scene`（即 `~/Desktop/aivo_video/split_scene`）
- 路径不存在时需要先创建
- **注意：不要写死 `C:/Users/Administrator/...`，必须用环境变量动态获取用户目录**

---

## 快速示例

> **"下载 https://v.douyin.com/ia-nnT8Ysck/ 并按场景拆解"**

对应流程：
1. 下载视频到本地
2. 检测场景切换点
3. 导出各镜头片段到 `<桌面路径>/aivo_video/split_scene`（动态获取，自动创建）

---

## 完整工作流

### Step 1 — 确认服务状态

同时检查三个端点：

```bash
curl -s --connect-timeout 3 http://localhost:7073/api/health
```

```bash
curl -s --connect-timeout 3 http://localhost:7073/api/video-download/health
```

```bash
curl -s --connect-timeout 3 http://localhost:7073/api/split-scene/health
```

三个都返回正常才能继续。若服务连接失败，提示**"容剪服务未启动，请先打开容剪客户端"**；若视频下载服务不可用，提示**"视频下载服务暂时不可用，请确认容剪客户端已完全启动"**；若场景拆解不可用，提示**"场景拆解服务不可用"**。

---

### Step 2 — 解析视频

```bash
curl -s -X POST http://localhost:7073/api/video-download \
  -H "Content-Type: application/json" \
  -d '{
    "text": "https://v.douyin.com/ia-nnT8Ysck/"
  }'
```

**成功响应：**
```json
{
  "retcode": 200,
  "succ": true,
  "data": {
    "video_id": "7519781625333779770",
    "platform": "抖音",
    "title": "这个宝宝太可爱了",
    "download_url": "/downloads/20260310/douyin/7519781625333779770.mp4",
    "fallback_crawl": false
  }
}
```

保存 `data.download_url`（若返回字段为 `local_file_path`，从中提取 `/downloads/` 部分）。

---

### Step 2b — 下载视频到本地

```bash
curl -s -X POST http://localhost:7073/api/video-download/save-to-local \
  -H "Content-Type: application/json" \
  -d '{
    "download_url": "/downloads/20260310/douyin/7519781625333779770.mp4"
  }'
```

**成功响应：**
```json
{
  "retcode": 200,
  "succ": true,
  "data": {
    "local_path": "<用户下载目录>/aivo-videos/7519781625333779770.mp4",
    "file_size": 15728640,
    "save_dir": "<用户下载目录>/aivo-videos"
  }
}
```

保存 `data.local_path`，这是下载到本地的视频文件路径，后续步骤使用。

如果下载失败，停止流程并告知用户。

告知用户：**视频已下载到本地，开始场景拆解...**

---

### Step 3 — 获取视频信息

```bash
curl -s -X POST http://localhost:7073/api/split-scene/video-info \
  -H "Content-Type: application/json" \
  -d '{"videoPath": "<Step 2b 返回的 data.local_path>"}'
```

**返回示例：**
```json
{
  "success": true,
  "data": {
    "path": "<用户下载目录>/aivo-videos/7519781625333779770.mp4",
    "duration": 120.5,
    "fps": 29.97,
    "width": 1920,
    "height": 1080,
    "bitrate": 8000000
  }
}
```

向用户展示视频基本信息。

---

### Step 4 — 发起场景检测（异步）

```bash
curl -s -X POST http://localhost:7073/api/split-scene/detect \
  -H "Content-Type: application/json" \
  -d '{
    "videoPath": "<Step 2b 返回的 data.local_path>",
    "threshold": 10
  }'
```

**threshold 说明：**
- 范围：1-30（UI 值，服务端自动换算为 FFmpeg 阈值 0.03-0.60）
- 值越小 → 检测越灵敏 → 切点越多
- 值越大 → 只检测明显场景变化 → 切点越少
- **推荐默认值：10**

**返回：**
```json
{
  "success": true,
  "taskId": "split_scene_1741234567_abc"
}
```

---

### Step 5 — 轮询检测进度

每 **2秒** 执行一次：

```bash
curl -s "http://localhost:7073/api/split-scene/detect/progress?taskId=split_scene_1741234567_abc"
```

**完成返回：**
```json
{
  "success": true,
  "status": "completed",
  "progress": 100,
  "scenes": [0, 8.34, 21.07, 35.60, 52.13, 78.90, 104.22, 120.5],
  "videoInfo": {
    "duration": 120.5,
    "fps": 29.97,
    "width": 1920,
    "height": 1080
  }
}
```

向用户展示检测结果：**检测到 N 个场景，共 M 个镜头片段**，列出每个镜头的时间范围和时长。

---

### Step 6 — 确认并发起导出

用户确认后（或如果用户之前已明确指示自动导出，则无需确认），构建 `shots` 数组并发起导出。

输出目录优先使用用户指定的路径，若未指定，默认使用下载视频所在目录下的 `shots` 子目录。

```bash
curl -s -X POST http://localhost:7073/api/split-scene/export-shots \
  -H "Content-Type: application/json" \
  -d '{
    "shots": [
      {
        "sourcePath": "<Step 2b 返回的 data.local_path>",
        "outputDir": "D:/shots",
        "outputName": "shot_001",
        "startTime": 0,
        "endTime": 8.34,
        "fps": 29.97,
        "width": 1920,
        "height": 1080
      },
      {
        "sourcePath": "<Step 2b 返回的 data.local_path>",
        "outputDir": "D:/shots",
        "outputName": "shot_002",
        "startTime": 8.34,
        "endTime": 21.07,
        "fps": 29.97,
        "width": 1920,
        "height": 1080
      }
    ]
  }'
```

**shots 字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `sourcePath` | string | 源视频文件路径（Step 2b 下载得到的 `data.local_path`） |
| `outputDir` | string | 输出目录 |
| `outputName` | string | 输出文件名（不含扩展名，如 `shot_001`） |
| `startTime` | float64 | 起始时间（秒，取自 `scenes` 数组） |
| `endTime` | float64 | 结束时间（秒，取自 `scenes` 数组） |
| `fps` | float64 | 帧率（从 `videoInfo` 取） |
| `width` | int | 视频宽度（从 `videoInfo` 取） |
| `height` | int | 视频高度（从 `videoInfo` 取） |

**返回：**
```json
{
  "success": true,
  "taskId": "export_shot_1741234599_xyz"
}
```

---

### Step 7 — 轮询导出进度

每 **2秒** 执行一次：

```bash
curl -s "http://localhost:7073/api/split-scene/export-shots/progress?taskId=export_shot_1741234599_xyz"
```

**完成返回：**
```json
{
  "success": true,
  "status": "completed",
  "total": 7,
  "current": 7,
  "successCount": 7,
  "results": [
    {"outputPath": "D:/shots/shot_001.mp4", "success": true},
    {"outputPath": "D:/shots/shot_002.mp4", "success": true}
  ]
}
```

向用户展示进度：`▓▓▓▓▓░░░░░ 3/7 — 正在导出第3个片段...`

---

### Step 8 — 完成

导出完成后告知用户：
- 原视频下载路径
- 视频标题和平台
- 输出目录
- 成功导出片段数
- 列出各片段文件名

---

## 错误处理

> **重要：不要向用户展示原始的后端错误信息（如 IP 地址、端口号、内部服务名、日志路径等），应转换为用户友好的提示。**

| 错误情况 | 处理方式 |
|----------|----------|
| 服务连接失败 | 提示用户：**"容剪服务未启动，请先打开容剪客户端"** |
| 视频下载服务不可用 | 提示用户：**"视频下载服务暂时不可用，请确认容剪客户端已完全启动"** |
| 视频解析或下载失败 | 停止流程，告知用户链接可能无效或平台不支持 |
| 场景检测 `status: "failed"` | 提示用户：**"场景检测失败，请检查视频文件是否有效"**，不要直接展示 `error` 中的系统错误信息 |
| `scenes` 为空数组 | 建议降低 `threshold` 值（尝试 5 或 3）重新检测 |
| 检测场景过多（> 100）| 建议提高 `threshold` 值（尝试 20） |
| 导出某片段失败 | `results` 中 `success:false` 的条目，记录并告知用户 |
| 其他未知错误 | 提示用户：**"操作失败，请重试"**，不要暴露内部错误堆栈 |
