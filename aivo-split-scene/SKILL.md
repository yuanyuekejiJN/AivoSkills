---
name: aivo-split-scene
description: |
  调用容剪(AIVO)按场景拆解视频功能，自动检测场景切换点并将视频拆分为多个镜头片段导出。
  触发关键词（匹配任意一个）：
  - "按场景拆解"、"拆分视频镜头"、"场景切割"、"把视频按场景分开"
  - "场景检测"、"拆解视频"、"分割场景"、"视频拆分"、"场景拆分"
  - "按场景分段"、"镜头拆分"、"拆分镜头"、"split scene"
  - 用户给了一个本地视频文件路径并要求"拆解"、"拆分"、"切分"
  支持单视频检测和批量多视频检测，可导出为独立片段或生成 DaVinci Resolve Lua 脚本。
  容剪客户端必须已启动（端口 7073）。
  注意：API 路径前缀是 /api/split-scene/（不是 /api/tools/split-scene/）。
metadata:
  openclaw:
    emoji: "🎞️"
    requires:
      bins:
        - curl
    os:
      - darwin
      - linux
      - win32
---

# 按场景拆解视频

> **严格约束（必须遵守）：**
> 1. **只使用本文档中明确列出的 API 路径**，绝对不要猜测或构造文档中没有的 URL
> 2. 基础地址固定为 `http://localhost:7073`
> 3. 路径前缀是 `/api/split-scene/`，**不是** `/api/tools/split-scene/`，**不是** `/api/scene/`
> 4. `/api/health` 返回 `{"status":"ok"}` 即代表服务可用（含场景拆解功能），不要再告诉用户"功能暂不支持"或"API 返回 404"
> 5. Windows 路径在 JSON 中必须用正斜杠 `/`（如 `D:/vlog.mp4`）
> 6. **如果用户给了本地视频路径并要求拆解/拆分，直接使用本技能，不要用 FFmpeg 命令行替代**

## 接口一览（仅限使用这些路径）

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/api/health` | 全局健康检查 |
| GET | `/api/split-scene/health` | 场景拆解模块健康检查 |
| POST | `/api/split-scene/video-info` | 获取视频基本信息 |
| POST | `/api/split-scene/detect` | 发起场景检测（异步） |
| GET | `/api/split-scene/detect/progress?taskId=` | 轮询检测进度 |
| POST | `/api/split-scene/detect-batch` | 批量检测多个视频 |
| POST | `/api/split-scene/export-shots` | 导出镜头片段 |
| GET | `/api/split-scene/export-shots/progress?taskId=` | 轮询导出进度 |
| POST | `/api/split-scene/generate-script` | 生成 DaVinci Resolve 脚本 |

自动用 FFmpeg 检测视频中的场景切换点，将整段视频拆分成多个独立镜头片段。

对应容剪界面：工具栏 → 按场景拆解视频

---

## 输出路径规则

> **如果用户没有指定输出路径，自动使用桌面下的 `aivo_video/split_scene` 目录并告知用户。**

- 用户指定了路径 → 使用用户指定的路径
- 用户未指定 → 动态拼接默认路径：
  - **Windows**: `%USERPROFILE%/Desktop/aivo_video/split_scene`（即 `$env:USERPROFILE/Desktop/aivo_video/split_scene`）
  - **macOS/Linux**: `$HOME/Desktop/aivo_video/split_scene`（即 `~/Desktop/aivo_video/split_scene`）
- 路径不存在时需要先用 `mkdir -p`（macOS/Linux）或 PowerShell `New-Item -ItemType Directory`（Windows）创建
- **注意：不要写死 `C:/Users/Administrator/...`，必须用环境变量动态获取用户目录**

---

## 快速示例

> **"把 D:/vlog.mp4 按场景拆解"**

对应参数：
- `videoPath` = `D:/vlog.mp4`
- `outputDir` = `<桌面路径>/aivo_video/split_scene`（动态获取，自动创建）
- `threshold` = `10`（UI 灵敏度值，范围 1-30，默认 10）

---

## 完整工作流

### Step 1 — 确认服务状态

```bash
curl -s --connect-timeout 3 http://localhost:7073/api/health
```

返回 `{"status":"ok"}` 表示正常。若连接失败，告知用户：**请先启动容剪客户端**。

---

### Step 2 — 获取视频基本信息（可选）

```bash
curl -s -X POST http://localhost:7073/api/split-scene/video-info   -H "Content-Type: application/json"   -d '{"videoPath": "D:/vlog.mp4"}'
```

**返回示例：**
```json
{
  "success": true,
  "data": {
    "path": "D:/vlog.mp4",
    "duration": 120.5,
    "fps": 29.97,
    "width": 1920,
    "height": 1080,
    "bitrate": 8000000
  }
}
```

告知用户视频时长、分辨率等基本信息。

---

### Step 3 — 发起场景检测（异步）

```bash
curl -s -X POST http://localhost:7073/api/split-scene/detect   -H "Content-Type: application/json"   -d '{
    "videoPath": "D:/vlog.mp4",
    "threshold": 10
  }'
```

**threshold 说明：**
- 范围：1-30（UI 值，服务端自动换算为 FFmpeg 阈值 0.03-0.60）
- 值越小 → 检测越灵敏 → 切点越多
- 值越大 → 只检测明显场景变化 → 切点越少
- **推荐默认值：10**

**返回示例：**
```json
{
  "success": true,
  "taskId": "split_scene_1741234567_abc"
}
```

保存 `taskId`，用于下一步轮询。

---

### Step 4 — 轮询检测进度

每 **2秒** 执行一次：

```bash
curl -s "http://localhost:7073/api/split-scene/detect/progress?taskId=split_scene_1741234567_abc"
```

**进行中返回：**
```json
{
  "success": true,
  "taskId": "split_scene_1741234567_abc",
  "status": "processing",
  "progress": 60.0,
  "message": "正在检测场景..."
}
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

`scenes` 是场景切点时间戳数组（秒），相邻两个值之间为一个镜头片段：
- 镜头1：0 → 8.34 秒
- 镜头2：8.34 → 21.07 秒
- 镜头3：21.07 → 35.60 秒
- ...

向用户展示：**检测到 N 个场景，共 M 个镜头片段**，列出每个镜头的时间范围和时长。

---

### Step 5 — 确认并发起导出

用户确认后，构建 `shots` 数组（每个镜头一个对象）并发起导出：

```bash
curl -s -X POST http://localhost:7073/api/split-scene/export-shots   -H "Content-Type: application/json"   -d '{
    "shots": [
      {
        "sourcePath": "D:/vlog.mp4",
        "outputDir": "D:/shots",
        "outputName": "shot_001",
        "startTime": 0,
        "endTime": 8.34,
        "fps": 29.97,
        "width": 1920,
        "height": 1080
      },
      {
        "sourcePath": "D:/vlog.mp4",
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
| `sourcePath` | string | 源视频文件路径 |
| `outputDir` | string | 输出目录 |
| `outputName` | string | 输出文件名（不含扩展名，如 `shot_001`） |
| `startTime` | float64 | 起始时间（秒，取自 `scenes` 数组） |
| `endTime` | float64 | 结束时间（秒，取自 `scenes` 数组） |
| `fps` | float64 | 帧率（从 `videoInfo` 取） |
| `width` | int | 视频宽度（从 `videoInfo` 取） |
| `height` | int | 视频高度（从 `videoInfo` 取） |

**返回示例：**
```json
{
  "success": true,
  "taskId": "export_shot_1741234599_xyz"
}
```

---

### Step 6 — 轮询导出进度

每 **2秒** 执行一次：

```bash
curl -s "http://localhost:7073/api/split-scene/export-shots/progress?taskId=export_shot_1741234599_xyz"
```

**进行中返回：**
```json
{
  "success": true,
  "status": "running",
  "total": 7,
  "current": 3,
  "message": "正在导出第 3/7 个片段..."
}
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

### Step 7 — 完成

导出完成后告知用户：
- 输出目录：`D:/shots`
- 成功导出片段数：`successCount`
- 列出各片段文件名

---

## 批量检测多个视频

若用户有多个视频需要同时检测：

```bash
curl -s -X POST http://localhost:7073/api/split-scene/detect-batch   -H "Content-Type: application/json"   -d '{
    "videos": [
      {"videoPath": "D:/videos/clip1.mp4", "materialId": "clip1", "threshold": 10},
      {"videoPath": "D:/videos/clip2.mp4", "materialId": "clip2", "threshold": 10}
    ]
  }'
```

**返回：**
```json
{
  "success": true,
  "taskIds": {
    "clip1": "split_scene_xxx_1",
    "clip2": "split_scene_xxx_2"
  }
}
```

对每个 `taskId` 分别轮询进度（同 Step 4）。

---

## 生成 DaVinci Resolve 脚本（可选）

若用户需要在 DaVinci Resolve 中剪辑而非直接导出片段：

```bash
curl -s -X POST http://localhost:7073/api/split-scene/generate-script   -H "Content-Type: application/json"   -d '{
    "scriptDir": "D:/shots",
    "exportFormat": "MP4",
    "exportCodec": "H264",
    "shots": [
      {
        "sourcePath": "D:/vlog.mp4",
        "outputDir": "D:/shots",
        "outputName": "shot_001",
        "startTime": 0,
        "endTime": 8.34,
        "fps": 29.97,
        "width": 1920,
        "height": 1080
      }
    ]
  }'
```

**返回：**
```json
{
  "success": true,
  "scriptPath": "D:/shots/split_scene.lua",
  "dofileCmd": "dofile('D:/shots/split_scene.lua')"
}
```

告知用户在 DaVinci Resolve 控制台执行 `dofileCmd` 命令即可自动导入并剪辑。

---

## 错误处理

> **重要：不要向用户展示原始的后端错误信息（如 IP 地址、端口号、内部服务名、日志路径等），应转换为用户友好的提示。**

| 错误情况 | 处理方式 |
|----------|----------|
| 连接失败 | 提示用户：**"容剪服务未启动，请先打开容剪客户端"** |
| `videoPath` 不存在 | 检查文件路径是否正确 |
| `status: "failed"` | 提示用户：**"场景检测失败，请检查视频文件是否有效"**，不要直接展示 `error` 中的系统错误信息 |
| `scenes` 为空数组 | 降低 `threshold` 值（尝试 5 或 3）再重新检测 |
| 检测场景过多（> 100）| 提高 `threshold` 值（尝试 20）减少切点 |
| 导出某片段失败 | `results` 中 `success:false` 的条目，记录并告知用户 |
| 其他未知错误 | 提示用户：**"操作失败，请重试"**，不要暴露内部错误堆栈 |
