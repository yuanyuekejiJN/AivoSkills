---
name: aivo-luban
description: |
  调用容剪(AIVO)鲁班模式(ZGL)进行视频自动剪辑，批量生成多版本视频。
  触发关键词（匹配任意一个）：
  - "鲁班模式"、"ZGL模式"、"自动剪视频"、"批量生成视频版本"
  - "用鲁班剪辑"、"批量剪辑"、"混剪素材"、"自动生成多版本"
  需要用户提供素材文件夹路径、输出路径和版本数量。
  容剪客户端必须已启动（端口 7073）。
metadata:
  openclaw:
    emoji: "🎬"
    requires:
      bins:
        - curl
    os:
      - darwin
      - linux
      - win32
---

# 鲁班模式 (ZGL) 视频剪辑

> **严格约束（必须遵守）：**
> 1. **只使用本文档中明确列出的 API 路径**，绝对不要猜测或构造文档中没有的 URL
> 2. 基础地址固定为 `http://localhost:7073`
> 3. Windows 路径在 JSON 中**必须用正斜杠** `/` 替代反斜杠 `\`（例：`"E:/素材/1"` 正确，`"E:\\素材\\1"` 错误）
> 4. 所有 POST 请求必须带 `Content-Type: application/json`
> 5. **所有 curl 命令必须用 bash/sh 执行，不要用 PowerShell**
> 6. **Step 4 和 Step 5 的 `chapters` 必须包含完整的素材路径列表，这是导出成功的关键**

## 接口一览（仅限使用这些路径）

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/api/health` | 全局健康检查 |
| GET | `/api/editMode/health` | 编辑模式健康检查 |
| POST | `/api/editMode/calculateCombinationsZGL` | 计算可用组合数 |
| POST | `/api/editMode/generateTaskFileZGL` | 生成任务表文件(.acvtl) |
| POST | `/api/editMode/exportSelectedTasksZGL` | 发起导出（生成时间线+启动渲染） |
| GET | `/api/editMode/zgl-render-progress?taskId=` | 轮询FFmpeg渲染进度 |

---

## 素材文件夹结构要求

素材文件夹**必须包含子文件夹**，每个子文件夹代表一个"章节"。不能直接把视频放在根目录下。

```
素材文件夹/                    ← 用户指定的路径（workspacePath）
  ├── 1/                      ← 章节1（子文件夹名可以是数字、中文等）
  │   ├── video1.mp4
  │   ├── video2.mp4
  │   └── video3.mp4
  ├── 2/                      ← 章节2
  │   ├── videoA.mp4
  │   └── videoB.mp4
  └── BGM/                    ← 可选，名字含 bgm/BGM/背景音乐 的文件夹
      └── music.mp3
```

**关键规则：**
- 子文件夹 = 章节，每个章节内放视频素材（.mp4/.avi/.mov/.mkv/.wmv/.flv/.webm/.m4v）
- 名字含 `bgm`/`BGM`/`背景音乐`/`背景音` 的子文件夹被识别为 BGM，不作为章节
- 如果用户给的是一个扁平文件夹（视频直接在根目录，没有子文件夹），**必须提示用户整理素材结构**

---

## 输出路径规则

> **如果用户没有指定输出路径，自动使用桌面下的 `aivo_video/luban` 目录并告知用户。**

- 用户指定了路径 → 使用用户指定的路径
- 用户未指定 → 动态拼接默认路径：
  - **Windows**: `%USERPROFILE%/Desktop/aivo_video/luban`（即 `$env:USERPROFILE/Desktop/aivo_video/luban`）
  - **macOS/Linux**: `$HOME/Desktop/aivo_video/luban`（即 `~/Desktop/aivo_video/luban`）
- 路径不存在时，服务端会自动创建
- **注意：不要写死 `C:/Users/Administrator/...`，必须用环境变量动态获取用户目录**

---

## 快速示例

> **"用鲁班模式把 E:/素材 里的视频剪成10个版本"**

对应参数：
- `workspacePath` = `E:/素材`（含子文件夹的素材根目录）
- `outputPath` = `<桌面路径>/aivo_video/luban`（用户未指定时动态获取桌面路径）
- `exportCount` = `10`（生成版本数，不超过组合数上限）

---

## 完整工作流

### Step 1 — 确认服务状态

```bash
curl -s --connect-timeout 3 http://localhost:7073/api/editMode/health
```

返回 `{"status":"ok","message":"编辑模式（鲁班/哪吒）功能正常"}` 表示可用。若连接失败，告知用户：**请先启动容剪客户端**。

---

### Step 2 — 扫描素材文件夹（本地完成，不调用 API）

你需要自行扫描用户指定的素材文件夹，获取章节结构和素材信息。

#### 2.1 列出子文件夹（章节）

扫描素材文件夹的直接子目录，**过滤掉 BGM 文件夹**（名字含 bgm/BGM/背景音乐/背景音，不区分大小写）和隐藏文件夹。

```bash
# macOS/Linux
ls -d "/path/to/素材"/*/

# Windows (用 bash，不要用 PowerShell)
ls -d "E:/素材"/*/
```

如果没有子文件夹，**停止并提示用户**整理素材目录结构。

#### 2.2 统计每个章节的视频文件和时长

对每个章节子文件夹，列出其中的视频文件（.mp4/.avi/.mov/.mkv/.wmv/.flv/.webm/.m4v），并用 ffprobe 获取每个视频的时长（秒）：

```bash
ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 "E:/素材/1/video1.mp4"
```

**收集以下数据（后续所有步骤都需要用到）：**

对于每个章节，记录：
- `name` — 子文件夹名称（如 `"1"`, `"2"`, `"开场"` 等）
- `path` — 子文件夹**完整绝对路径**（如 `"E:/素材/1"`）
- `videoCount` — 视频文件数量
- `videoMaterials` — 每个视频文件的**完整绝对路径**列表
- `videoDurations` — 每个视频的时长（秒）列表，与 videoMaterials 一一对应
- `avgVideoDuration` — 视频平均时长

#### 2.3 扫描 BGM（可选）

如果存在 BGM 文件夹（名字含 bgm/BGM/背景音乐），收集其中的音频文件**完整绝对路径**列表。

---

### Step 3 — 计算可用组合数

使用 Step 2 中收集的数据构建请求。**默认使用 `no_voice_fixed`（无配音固定素材数）模式**。

```bash
curl -s -X POST http://localhost:7073/api/editMode/calculateCombinationsZGL \
  -H "Content-Type: application/json" \
  -d '{
    "chapterConfigs": [
      {
        "mode": "no_voice_fixed",
        "videoCount": 11,
        "audioCount": 0,
        "randomSelectCount": 1,
        "repeatRate": 0,
        "voiceRepeatMode": "once",
        "removeOriginalAudio": false,
        "durationAdjustMethod": "cut",
        "cutPosition": "both",
        "loopPlotMethod": "",
        "loopCount": 1,
        "subChapterVideoCounts": [],
        "videoDurations": [5.2, 8.1, 3.4, 6.7, 4.5, 9.2, 7.8, 2.1, 5.6, 8.3, 4.9],
        "audioDurations": [],
        "avgVideoDuration": 5.98,
        "minDuration": 10,
        "chapterPath": "E:/素材/1"
      },
      {
        "mode": "no_voice_fixed",
        "videoCount": 5,
        "audioCount": 0,
        "randomSelectCount": 1,
        "repeatRate": 0,
        "voiceRepeatMode": "once",
        "removeOriginalAudio": false,
        "durationAdjustMethod": "cut",
        "cutPosition": "both",
        "loopPlotMethod": "",
        "loopCount": 1,
        "subChapterVideoCounts": [],
        "videoDurations": [4.1, 7.2, 3.8, 5.5, 6.3],
        "audioDurations": [],
        "avgVideoDuration": 5.38,
        "minDuration": 10,
        "chapterPath": "E:/素材/2"
      }
    ]
  }'
```

> **⚠️ `chapterConfigs` 数组���每个章节子文件夹对应一个元素。有 N 个章节就传 N 个元素。**
> 数据中的 `videoCount`、`videoDurations`、`avgVideoDuration`、`chapterPath` 全部来自 Step 2 的扫描结果。

**返回示例：**
```json
{
  "success": true,
  "globalCombinations": 5,
  "chapterCombinations": [11, 5]
}
```

- `globalCombinations` = 所有章节组合数的**最小值**，即最终可生成的最大版本数
- 如果用户要求的版本数超过此值，自动调整为 `globalCombinations`
- 如果返回 `0`，说明某个章节没有视频素材，检查文件夹结构

---

### Step 4 — 生成任务表文件

> **⚠️ 核心要求：`chapters` 数组中每个章节必须包含完整的 `materials` 和 `videoMaterials` 字段！**
> 这两个字段必须是 Step 2 中扫描得到的视频文件完整绝对路径列表。
> 如果这些字段为空，后端会回退到扫描 `path` 目录——虽然也能工作，但最可靠的方式是**直接传入完整路径列表**。

```bash
curl -s -X POST http://localhost:7073/api/editMode/generateTaskFileZGL \
  -H "Content-Type: application/json" \
  -d '{
    "workspacePath": "E:/素材",
    "outputPath": "<动态获取的桌面路径>/aivo_video/luban",
    "outputCount": 5,
    "chapters": [
      {
        "id": 1,
        "name": "1",
        "path": "E:/素材/1",
        "videoCount": 11,
        "audioCount": 0,
        "mode": "no_voice_fixed",
        "selectCount": 1,
        "repeatRate": 0,
        "removeOriginalAudio": false,
        "durationAdjustMethod": "cut",
        "cutPosition": "both",
        "voiceRepeatMode": "once",
        "loopPlotMethod": "",
        "loopCount": 1,
        "subChapterVideoCounts": [],
        "subChapters": [],
        "minDuration": 10,
        "avgVideoDuration": 5.98,
        "videoDurations": [5.2, 8.1, 3.4, 6.7, 4.5, 9.2, 7.8, 2.1, 5.6, 8.3, 4.9],
        "audioDurations": [],
        "materials": [
          "E:/素材/1/video1.mp4",
          "E:/素材/1/video2.mp4",
          "E:/素材/1/video3.mp4",
          "E:/素材/1/video4.mp4",
          "E:/素材/1/video5.mp4",
          "E:/素材/1/video6.mp4",
          "E:/素材/1/video7.mp4",
          "E:/素材/1/video8.mp4",
          "E:/素材/1/video9.mp4",
          "E:/素材/1/video10.mp4",
          "E:/素材/1/video11.mp4"
        ],
        "videoMaterials": [
          "E:/素材/1/video1.mp4",
          "E:/素材/1/video2.mp4",
          "E:/素材/1/video3.mp4",
          "E:/素材/1/video4.mp4",
          "E:/素材/1/video5.mp4",
          "E:/素材/1/video6.mp4",
          "E:/素材/1/video7.mp4",
          "E:/素材/1/video8.mp4",
          "E:/素材/1/video9.mp4",
          "E:/素材/1/video10.mp4",
          "E:/素材/1/video11.mp4"
        ],
        "audioMaterials": []
      },
      {
        "id": 2,
        "name": "2",
        "path": "E:/素材/2",
        "videoCount": 5,
        "audioCount": 0,
        "mode": "no_voice_fixed",
        "selectCount": 1,
        "repeatRate": 0,
        "removeOriginalAudio": false,
        "durationAdjustMethod": "cut",
        "cutPosition": "both",
        "voiceRepeatMode": "once",
        "loopPlotMethod": "",
        "loopCount": 1,
        "subChapterVideoCounts": [],
        "subChapters": [],
        "minDuration": 10,
        "avgVideoDuration": 5.38,
        "videoDurations": [4.1, 7.2, 3.8, 5.5, 6.3],
        "audioDurations": [],
        "materials": [
          "E:/素材/2/videoA.mp4",
          "E:/素材/2/videoB.mp4",
          "E:/素材/2/videoC.mp4",
          "E:/素材/2/videoD.mp4",
          "E:/素材/2/videoE.mp4"
        ],
        "videoMaterials": [
          "E:/素材/2/videoA.mp4",
          "E:/素材/2/videoB.mp4",
          "E:/素材/2/videoC.mp4",
          "E:/素材/2/videoD.mp4",
          "E:/素材/2/videoE.mp4"
        ],
        "audioMaterials": []
      }
    ],
    "videoSettings": {
      "aspectRatio": "9:16",
      "width": 1080,
      "height": 1920,
      "frameRate": 60,
      "mirrorProbability": 50,
      "cropRangeMin": 100,
      "cropRangeMax": 120
    },
    "audioSettings": {
      "backgroundMusic": {
        "enabled": false,
        "path": "",
        "bgmList": [],
        "volume": 60,
        "playbackMode": "loop",
        "randomPerWork": true,
        "startFromChapter": 1
      }
    }
  }'
```

> **⚠️ 非常重要：请把这次请求中构造的完整 JSON 数据保存下来（特别是 `chapters`、`videoSettings`、`audioSettings`），Step 5 需要原样复用！**

**返回示例：**
```json
{
  "success": true,
  "filePath": "<outputPath>/鲁班_3月17日10时30分.acvtl",
  "message": "任务表生成成功"
}
```

保存返回的 `filePath`，下一步使用。

---

### Step 5 — 发起导出

> **⚠️⚠️⚠️ 最关键的一步：`chapters`、`videoSettings`、`audioSettings` 必须与 Step 4 完全一致！**
>
> 后端**不会**从 acvtl 文件中读取章节和素材数据，**完全依赖**此请求中传入的 `chapters`。
> 如果 `chapters` 中的 `materials`/`videoMaterials` 为空，后端会尝试扫描 `path` 目录获取视频。
> 如果 `path` 和 `materials` 都为空或不正确，**渲染将产出 0 个视频！**
>
> **最安全的做法：直接复用 Step 4 中构造的 chapters、videoSettings、audioSettings 原始数据。**

先生成一个唯一的 `sessionId`：格式为 `export_zgl_{毫秒时间戳}_{随机字符}`。

`selectedTaskIds` 为从 1 到 outputCount 的整数数组（即选择全部任务导出）。

```bash
curl -s -X POST http://localhost:7073/api/editMode/exportSelectedTasksZGL \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "export_zgl_1710680000000_abc123def",
    "exportMode": "direct",
    "acvtlPath": "<Step 4 返回的 filePath>",
    "selectedTaskIds": [1, 2, 3, 4, 5],
    "outputPath": "<动态获取的桌面路径>/aivo_video/luban",
    "workspacePath": "E:/素材",
    "chapters": <与 Step 4 完全相同的 chapters 数组>,
    "videoSettings": <与 Step 4 完全相同的 videoSettings>,
    "audioSettings": <与 Step 4 完全相同的 audioSettings>,
    "presetParams": {
      "resolution": "1080p",
      "codec": "H.264",
      "format": "MP4",
      "frameRate": 60,
      "renderMode": "ffmpeg"
    }
  }'
```

**说明：**
- `sessionId`: 唯一会话ID，格式 `export_zgl_{timestamp}_{random}`
- `exportMode`: 固定用 `"direct"`（FFmpeg 直接渲染）
- `acvtlPath`: Step 4 返回的 `filePath` 值
- `selectedTaskIds`: 从 `1` 到 `outputCount` 的整数数组（如导出 5 个版本就传 `[1,2,3,4,5]`）
- `outputPath`/`workspacePath`: 与 Step 4 相同
- `chapters`: **必须与 Step 4 的 chapters 完全一致，包含完整的 materials/videoMaterials 列表**
- `videoSettings`/`audioSettings`: **必须与 Step 4 完全一致**
- `presetParams`: 固定值，注意 `frameRate` 为 `60`，`renderMode` 为 `"ffmpeg"`

**返回示例：**
```json
{
  "success": true,
  "sessionId": "export_zgl_1710680000000_abc123def",
  "renderTaskId": "zgl_render_1710680005000_5",
  "outputDir": "<outputPath>/鲁班_03月17日10时30分/导出视频",
  "exportedTasks": [
    {"id": 1, "exportTime": "2026-03-17T10:30:05", "outputPath": "..."}
  ],
  "message": "已启动 5 个视频的渲染"
}
```

保存 `renderTaskId` 和 `outputDir`，用于进度轮询。

---

### Step 6 — 轮询渲染进度

导出启动后，每 **3秒** 轮询一次渲染进度：

```bash
curl -s "http://localhost:7073/api/editMode/zgl-render-progress?taskId=<renderTaskId>"
```

**进行中返回：**
```json
{
  "success": true,
  "status": "running",
  "progress": 40.5,
  "current": 2,
  "total": 5,
  "currentMsg": "已完成 2/5"
}
```

**完成返回：**
```json
{
  "success": true,
  "status": "completed",
  "progress": 100,
  "current": 5,
  "total": 5,
  "results": [
    {"taskId": 1, "success": true, "outputPath": "..."},
    {"taskId": 2, "success": true, "outputPath": "..."}
  ],
  "outputDir": "<outputPath>/鲁班_03月17日10时30分/导出视频"
}
```

**`status` 值：**
- `"pending"` — 等待中
- `"running"` / `"processing"` — 渲染中
- `"completed"` — 全部完成
- `"failed"` — 渲染失败

向用户展示进度：`▓▓▓▓░░░░░░ 40% — 已完成 2/5 个视频`

---

### Step 7 — 完成

渲染完成后告知用户：
- 输出路径：`outputDir` 的值
- 实际生成视频数量
- 可以直接打开文件夹查看生成的 MP4 文件

---

## 参数默认值速查

当用户未指定某些参数时，使用以下默认值：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `mode` | `"no_voice_fixed"` | 无配音固定素材数（最常用） |
| `selectCount` / `randomSelectCount` | `1` | 每章节选1个视频 |
| `repeatRate` | `0` | 1档（最低重复率） |
| `aspectRatio` | `"9:16"` | 竖屏；用户说"横屏"时用 `"16:9"` |
| `frameRate` | `60` | 帧率（videoSettings 和 presetParams 都用 60） |
| `mirrorProbability` | `50` | 50% 概率镜像翻转 |
| `cropRangeMin` / `cropRangeMax` | `100` / `120` | 随机裁剪范围 |
| `cutPosition` | `"both"` | 头尾各剪一半 |
| `minDuration` | `10` | 最短时长（秒） |
| `durationAdjustMethod` | `"cut"` | 裁剪方式 |
| `loopPlotMethod` | `""` | 空字符串（非循环情节模式） |
| `removeOriginalAudio` | `false` | 保留原声 |
| `exportMode` | `"direct"` | FFmpeg 直接渲染 |
| `renderMode` | `"ffmpeg"` | FFmpeg 渲染模式 |
| BGM `randomPerWork` | `true` | 每条作品随机选择BGM |
| BGM `enabled` | `false` | 默认不启用（除非检测到 BGM 文件夹） |

---

## BGM 配置

**自动启用规则：** 如果 Step 2 扫描时发现了 BGM 文件夹（名字含 bgm/BGM/背景音乐），自动启用 BGM。

**有 BGM 时的 audioSettings：**
```json
{
  "backgroundMusic": {
    "enabled": true,
    "path": "E:/素材/BGM/music1.mp3",
    "bgmList": [
      "E:/素材/BGM/music1.mp3",
      "E:/素材/BGM/music2.mp3"
    ],
    "volume": 60,
    "playbackMode": "loop",
    "randomPerWork": true,
    "startFromChapter": 1
  }
}
```

| 字段 | 说明 |
|------|------|
| `enabled` | `true` 启用 BGM |
| `path` | `bgmList` 中的第一个文件路径 |
| `bgmList` | BGM 文件**完整绝对路径**列表（Step 2 扫描得到） |
| `volume` | 音量 0-100，默认 `60` |
| `playbackMode` | `"loop"`（循环播放），默认值 |
| `randomPerWork` | `true`（每条作品随机选一个 BGM） |
| `startFromChapter` | 从第 `1` 个章节开始应用 BGM |

**没有 BGM 时的 audioSettings（默认）：**
```json
{
  "backgroundMusic": {
    "enabled": false,
    "path": "",
    "bgmList": [],
    "volume": 60,
    "playbackMode": "loop",
    "randomPerWork": true,
    "startFromChapter": 1
  }
}
```

---

## videoSettings 宽高对照表

| 用户场景 | aspectRatio | width | height |
|----------|-------------|-------|--------|
| 竖屏9:16（默认，抖音/短视频） | `"9:16"` | 1080 | 1920 |
| 竖屏3:4 | `"3:4"` | 1080 | 1440 |
| 横屏16:9 | `"16:9"` | 1920 | 1080 |
| 横屏4:3 | `"4:3"` | 1440 | 1080 |
| 方屏1:1 | `"1:1"` | 1080 | 1080 |

默认使用 `9:16`。如果用户提到"横屏"或"电脑"，使用 `16:9`。

---

## 完整流程总结

```
1. 健康检查          → GET  /api/editMode/health
2. 本地扫描素材       → 列子文件夹 + ffprobe 获取视频路径和时长
3. 计算组合数         → POST /api/editMode/calculateCombinationsZGL
4. 生成任务表         → POST /api/editMode/generateTaskFileZGL（保存完整请求数据！）
5. 发起导出           → POST /api/editMode/exportSelectedTasksZGL（复用 Step 4 的 chapters/settings！）
6. 轮询渲染进度       → GET  /api/editMode/zgl-render-progress?taskId=
7. 完成，告知用户输出路径
```

> **导出失败排查清单（如果渲染 0 个视频）：**
> 1. 检查 Step 5 的 `chapters` 是否包含完整的 `materials` 和 `videoMaterials` 列表
> 2. 检查 `chapters` 中每个章节的 `path` 是否是正确的绝对路径
> 3. 检查 `selectedTaskIds` 是否为非空的整数数组（如 `[1,2,3,4,5]`）
> 4. 检查 `videoSettings` 和 `audioSettings` 是否完整（不能为空对象）
> 5. 检查 Windows 路径是否使用了正斜杠 `/`

---

## 错误处理

> **重要：不要向用户展示原始的后端错误信息（如 IP 地址、端口号、内部服务名、日志路径等），应转换为用户友好的提示。**

| 错误情况 | 处理方式 |
|----------|----------|
| 连接失败 | 提示：**"容剪服务未启动，请先打开容剪客户端"** |
| 素材文件夹无子文件夹 | 提示：**"素材文件夹中需要子文件夹作为章节，请将视频分组到子文件夹中"** |
| `globalCombinations` 为 0 | 检查：1）是否有子文件夹；2）子文件夹中是否有视频文件；3）视频格式是否支持 |
| 版本数超过组合数 | 自动调整为 `globalCombinations` |
| 渲染完成但 0 个视频 | 检查 `chapters` 是否传入了完整的 `materials`/`videoMaterials` 和正确的 `path` |
| 导出没有 BGM | 检查 `audioSettings.backgroundMusic.enabled` 是否为 `true`，`bgmList` 是否包含有效路径 |
| `status: "failed"` | 仅展示面向用户的描述，**不暴露内部技术细节** |
| 渲染超时（> 15分钟无进度更新） | 提示视频数量较多，继续等待或减少版本数 |
