---
name: aivo-nezha
description: |
  调用容剪(AIVO)哪吒模式(SWK)进行多轨道矩阵视频剪辑。
  触发关键词（匹配任意一个）：
  - "哪吒模式"、"SWK"、"多轨道剪辑"、"矩阵组合剪辑"
  - "多轨道混剪"、"排列组合剪辑"、"矩阵剪辑"
  哪吒模式支持多个素材轨道排列组合，生成所有可能的视频版本。
  容剪客户端必须已启动（端口 7073）。
metadata:
  openclaw:
    emoji: "⚡"
    requires:
      bins:
        - curl
    os:
      - darwin
      - linux
      - win32
---

# 哪吒模式 (SWK) 多轨道矩阵剪辑

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
| POST | `/api/editMode/calculateCombinationsSWK` | 计算可用组合数 |
| POST | `/api/editMode/generateTaskFileSWK` | 生成任务表文件(.acvtl) |
| POST | `/api/editMode/exportSelectedTasksSWK` | 发起导出（生成时间线+启动渲染） |
| GET | `/api/editMode/exportProgress?sessionId=` | 轮询时间线生成进度（阶段一） |
| GET | `/api/editMode/swk-render-progress?taskId=` | 轮询FFmpeg渲染进度（阶段二） |

---

## 素材文件夹结构要求

哪吒模式与鲁班模式相同，素材文件夹**必须包含子文件夹**，每个子文件夹代表一个"章节"（轨道）。

```
素材文件夹/                    ← workspacePath
  ├── 1/                      ← 章节1/轨道1
  │   ├── video1.mp4
  │   ├── video2.mp4
  │   └── video3.mp4
  ├── 2/                      ← 章节2/轨道2
  │   ├── videoA.mp4
  │   └── videoB.mp4
  ├── 4/                      ← 章节3/轨道3
  │   └── ...
  └── BGM/                    ← 可选，背景音乐
      └── music.mp3
```

**关键规则：**
- 子文件夹 = 章节/轨道，每个内放视频素材（.mp4/.avi/.mov/.mkv/.wmv/.flv/.webm/.m4v）
- 名字含 `bgm`/`BGM`/`背景音乐`/`背景音` 的子文件夹被识别为 BGM，不作为章节
- **组合数** = 各轨道视频数之积（如 3 x 2 x 5 = 30）
- 如果用户给的是一个扁平文件夹，**必须提示用户整理素材结构**

---

## 输出路径规则

> **如果用户没有指定输出路径，自动使用桌面下的 `aivo_video/nezha` 目录并告知用户。**

- 用户指定了路径 → 使用用户指定的路径
- 用户未指定 → 动态拼接默认路径：
  - **Windows**: `%USERPROFILE%/Desktop/aivo_video/nezha`（即 `$env:USERPROFILE/Desktop/aivo_video/nezha`）
  - **macOS/Linux**: `$HOME/Desktop/aivo_video/nezha`（即 `~/Desktop/aivo_video/nezha`）
- 路径不存在时，服务端会自动创建
- **注意：不要写死 `C:/Users/Administrator/...`，必须用环境变量动态获取用户目录**

---

## 快速示例

> **"用哪吒模式把 E:/素材 里的视频剪成10个版本"**

对应参数：
- `workspacePath` = `E:/素材`
- 使用全部子文件夹作为轨道（排除 BGM 文件夹）
- `outputPath` = `<桌面路径>/aivo_video/nezha`（动态获取）
- `exportCount` = `10`（不超过组合数上限）

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

#### 2.1 列出子文件夹（轨道/章节）

扫描素材文件夹的直接子目录，**过滤掉 BGM 文件夹**（名字含 bgm/BGM/背景音乐/背景音，不区分大小写）和隐藏文件夹。

```bash
# macOS/Linux
ls -d "/path/to/素材"/*/

# Windows (用 bash，不要用 PowerShell)
ls -d "E:/素材"/*/
```

如果没有子文件夹，**停止并提示用户**整理素材目录结构。

向用户展示可用的轨道列表和每个轨道的视频数，让用户确认使用哪些轨道。

#### 2.2 统计每个选中轨道的视频文件和时长

对每个轨道子文件夹，列出其中的视频文件，并用 ffprobe 获取每个视频的时长（秒）：

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

#### 2.3 扫描 BGM（重要）

检查是否存在 BGM 文件夹（名字含 bgm/BGM/背景音乐/背景音）。如果存在，收集其中的音频文件**完整绝对路径**列表。

> **BGM 自动启用规则：如果发现了 BGM 文件夹且内有音频文件，后续步骤中 `audioSettings.backgroundMusic.enabled` 必须设为 `true`，并填入 `bgmList`。**

---

### Step 3 — 计算可用组合数

使用 Step 2 中收集的数据构建请求。**哪吒模式默认使用 `fill_screen`（为配音填充画面）模式，无配音时也用此模式。**

```bash
curl -s -X POST http://localhost:7073/api/editMode/calculateCombinationsSWK \
  -H "Content-Type: application/json" \
  -d '{
    "voiceCount": 0,
    "totalVoiceDuration": 0,
    "audioDurations": [],
    "chapters": [
      {
        "mode": "fill_screen",
        "videoCount": 3,
        "audioCount": 0,
        "randomSelectCount": 1,
        "repeatRate": 0,
        "voiceRepeatMode": "once",
        "removeOriginalAudio": false,
        "durationAdjustMethod": "cut",
        "cutPosition": "both",
        "minDuration": 10,
        "avgVideoDuration": 5.98,
        "avgAudioDuration": 0,
        "loopPlotMethod": "speed",
        "loopCount": 1,
        "subChapterVideoCounts": [],
        "videoDurations": [5.2, 8.1, 3.4],
        "audioDurations": []
      },
      {
        "mode": "fill_screen",
        "videoCount": 2,
        "audioCount": 0,
        "randomSelectCount": 1,
        "repeatRate": 0,
        "voiceRepeatMode": "once",
        "removeOriginalAudio": false,
        "durationAdjustMethod": "cut",
        "cutPosition": "both",
        "minDuration": 10,
        "avgVideoDuration": 6.65,
        "avgAudioDuration": 0,
        "loopPlotMethod": "speed",
        "loopCount": 1,
        "subChapterVideoCounts": [],
        "videoDurations": [4.1, 9.2],
        "audioDurations": []
      }
    ]
  }'
```

> **`chapters` 数组：每个选中的轨道/子文件夹对应一个元素。有 N 个轨道就传 N 个元素。**
> 数据中的 `videoCount`、`videoDurations`、`avgVideoDuration` 全部来自 Step 2 的扫描结果。

**返回示例：**
```json
{
  "success": true,
  "totalCombinations": 6,
  "chapterCombinations": [3, 2]
}
```

- `totalCombinations` = 各轨道组合数之积（3 x 2 = 6）
- 向用户展示：**共 6 个可用组合，需要生成多少个版本？**
- 如果用户要求的版本数超过此值，自动调整为 `totalCombinations`

---

### Step 4 — 生成任务表文件

> **⚠️ 核心要求：**
> 1. **`outputPath` 不能为空字符串**，否则后端会返回"输出路径为空"错误
> 2. **`chapters` 数组中每个章节必须包含完整的 `materials` 和 `videoMaterials` 字段**

```bash
curl -s -X POST http://localhost:7073/api/editMode/generateTaskFileSWK \
  -H "Content-Type: application/json" \
  -d '{
    "workspacePath": "E:/素材",
    "outputPath": "<动态获取的桌面路径>/aivo_video/nezha",
    "outputCount": 6,
    "voiceCount": 0,
    "totalVoiceDuration": 0,
    "globalDurationAdjustMethod": "cut",
    "globalCutPosition": "both",
    "chapters": [
      {
        "id": 1,
        "name": "1",
        "path": "E:/素材/1",
        "videoCount": 3,
        "audioCount": 0,
        "mode": "fill_screen",
        "selectCount": 1,
        "repeatRate": 0,
        "removeOriginalAudio": false,
        "durationAdjustMethod": "cut",
        "cutPosition": "both",
        "voiceRepeatMode": "once",
        "loopPlotMethod": "speed",
        "loopCount": 1,
        "subChapterVideoCounts": [],
        "subChapters": [],
        "minDuration": 10,
        "avgVideoDuration": 5.57,
        "videoDurations": [5.2, 8.1, 3.4],
        "audioDurations": [],
        "materials": [
          "E:/素材/1/video1.mp4",
          "E:/素材/1/video2.mp4",
          "E:/素材/1/video3.mp4"
        ],
        "videoMaterials": [
          "E:/素材/1/video1.mp4",
          "E:/素材/1/video2.mp4",
          "E:/素材/1/video3.mp4"
        ],
        "audioMaterials": []
      },
      {
        "id": 2,
        "name": "2",
        "path": "E:/素材/2",
        "videoCount": 2,
        "audioCount": 0,
        "mode": "fill_screen",
        "selectCount": 1,
        "repeatRate": 0,
        "removeOriginalAudio": false,
        "durationAdjustMethod": "cut",
        "cutPosition": "both",
        "voiceRepeatMode": "once",
        "loopPlotMethod": "speed",
        "loopCount": 1,
        "subChapterVideoCounts": [],
        "subChapters": [],
        "minDuration": 10,
        "avgVideoDuration": 6.65,
        "videoDurations": [4.1, 9.2],
        "audioDurations": [],
        "materials": [
          "E:/素材/2/videoA.mp4",
          "E:/素材/2/videoB.mp4"
        ],
        "videoMaterials": [
          "E:/素材/2/videoA.mp4",
          "E:/素材/2/videoB.mp4"
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
  "filePath": "<outputPath>/哪吒_3月17日10时30分.acvtl",
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

先生成一个唯一的 `sessionId`：格式为 `export_swk_{毫秒时间戳}_{随机字符}`。

`selectedTaskIds` 为从 1 到 outputCount 的整数数组。

```bash
curl -s -X POST http://localhost:7073/api/editMode/exportSelectedTasksSWK \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "export_swk_1710680000000_abc123def",
    "exportMode": "direct",
    "acvtlPath": "<Step 4 返回的 filePath>",
    "selectedTaskIds": [1, 2, 3, 4, 5, 6],
    "outputPath": "<动态获取的桌面路径>/aivo_video/nezha",
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
    },
    "globalDurationAdjustMethod": "cut",
    "globalCutPosition": "both"
  }'
```

**说明：**
- `sessionId`: 唯一会话ID，格式 `export_swk_{timestamp}_{random}`
- `exportMode`: 固定用 `"direct"`（FFmpeg 直接渲染）
- `acvtlPath`: Step 4 返回的 `filePath` 值
- `selectedTaskIds`: 从 `1` 到 `outputCount` 的整数数组（如导出 6 个版本就传 `[1,2,3,4,5,6]`）
- `outputPath`/`workspacePath`: 与 Step 4 相同
- `chapters`: **必须与 Step 4 的 chapters 完全一致，包含完整的 materials/videoMaterials 列表**
- `videoSettings`/`audioSettings`: **必须与 Step 4 完全一致**
- `presetParams`: 固定值，注意 `frameRate` 为 `60`，`renderMode` 为 `"ffmpeg"`
- `globalDurationAdjustMethod`: `"cut"`（哪吒模式额外需要的全局参数）
- `globalCutPosition`: `"both"`（哪吒模式额外需要的全局参数）

**返回示例：**
```json
{
  "success": true,
  "sessionId": "export_swk_1710680000000_abc123def",
  "renderTaskId": "swk_render_1710680005000_6",
  "outputDir": "<outputPath>/哪吒_03月17日10时30分/导出视频",
  "message": "已启动 6 个视频的渲染"
}
```

保存 `renderTaskId` 和 `outputDir`，用于进度轮询。

---

### Step 6 — 轮询渲染进度

导出启动后，每 **3秒** 轮询一次渲染进度：

```bash
curl -s "http://localhost:7073/api/editMode/swk-render-progress?taskId=<renderTaskId>"
```

**进行中返回：**
```json
{
  "success": true,
  "status": "running",
  "progress": 50.0,
  "current": 3,
  "total": 6,
  "currentMsg": "已完成 3/6"
}
```

**完成返回：**
```json
{
  "success": true,
  "status": "completed",
  "progress": 100,
  "current": 6,
  "total": 6,
  "results": [
    {"taskId": 1, "success": true, "outputPath": "..."},
    {"taskId": 2, "success": true, "outputPath": "..."}
  ],
  "outputDir": "<outputPath>/哪吒_03月17日10时30分/导出视频"
}
```

**`status` 值：**
- `"pending"` — 等待中
- `"running"` / `"processing"` — 渲染中
- `"completed"` — 全部完成
- `"failed"` — 渲染失败

向用户展示进度：`▓▓▓▓▓░░░░░ 50% — 已完成 3/6 个视频`

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
| `mode` | `"fill_screen"` | 为配音填充画面（哪吒默认模式） |
| `selectCount` / `randomSelectCount` | `1` | 每章节选1个视频 |
| `repeatRate` | `0` | 1档（最低重复率） |
| `aspectRatio` | `"9:16"` | 竖屏；用户说"横屏"时用 `"16:9"` |
| `frameRate` | `60` | 帧率（videoSettings 和 presetParams 都用 60） |
| `mirrorProbability` | `50` | 50% 概率镜像翻转 |
| `cropRangeMin` / `cropRangeMax` | `100` / `120` | 随机裁剪范围 |
| `cutPosition` | `"both"` | 头尾各剪一半 |
| `minDuration` | `10` | 最短时长（秒） |
| `durationAdjustMethod` | `"cut"` | 裁剪方式 |
| `loopPlotMethod` | `"speed"` | 正常速度循环 |
| `removeOriginalAudio` | `false` | 保留原声 |
| `exportMode` | `"direct"` | FFmpeg 直接渲染 |
| `renderMode` | `"ffmpeg"` | FFmpeg 渲染模式 |
| `globalDurationAdjustMethod` | `"cut"` | 全局裁剪方式 |
| `globalCutPosition` | `"both"` | 全局裁剪位置 |
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

## 哪吒模式与鲁班模式的区别

| 维度 | 哪吒(SWK) | 鲁班(ZGL) |
|------|-----------|-----------|
| 默认模式 | `"fill_screen"` | `"no_voice_fixed"` |
| 组合数计算 | 各轨道之积（3x2=6） | 各章节最小值 |
| 额外参数 | `globalDurationAdjustMethod`、`globalCutPosition` | 无 |
| 计算组合数请求 | 含 `voiceCount`、`totalVoiceDuration`、`audioDurations` | 仅 `chapterConfigs` |
| acvtl 文件名 | `哪吒_X月X日X时X分.acvtl` | `鲁班_X月X日X时X分.acvtl` |

---

## 完整流程总结

```
1. 健康检查          → GET  /api/editMode/health
2. 本地扫描素材       → 列子文件夹 + ffprobe 获取视频路径和时长 + 检查 BGM 文件夹
3. 计算组合数         → POST /api/editMode/calculateCombinationsSWK
4. 生成任务表         → POST /api/editMode/generateTaskFileSWK（保存完整请求数据！）
5. 发起导出           → POST /api/editMode/exportSelectedTasksSWK（复用 Step 4 的 chapters/settings！）
6. 轮询渲染进度       → GET  /api/editMode/swk-render-progress?taskId=
7. 完成，告知用户输出路径
```

> **导出失败排查清单（如果渲染 0 个视频）：**
> 1. 检查 Step 5 的 `chapters` 是否包含完整的 `materials` 和 `videoMaterials` 列表
> 2. 检查 `chapters` 中每个章节的 `path` 是否是正确的绝对路径
> 3. 检查 `selectedTaskIds` 是否为非空的整数数组（如 `[1,2,3,4,5,6]`）
> 4. 检查 `videoSettings` 和 `audioSettings` 是否完整（不能为空对象）
> 5. 检查 `outputPath` 是否非空
> 6. 检查 Windows 路径是否使用了正斜杠 `/`

---

## 错误处理

> **重要：不要向用户展示原始的后端错误信息（如 IP 地址、端口号、内部服务名、日志路径等），应转换为用户友好的提示。**

| 错误情况 | 处理方式 |
|----------|----------|
| 连接失败 | 提示：**"容剪服务未启动，请先打开容剪客户端"** |
| "输出路径为空" | 确保 `outputPath` 非空，使用默认路径 |
| 素材文件夹无子文件夹 | 提示：**"素材文件夹中需要子文件夹作为章节，请将视频分组到子文件夹中"** |
| `totalCombinations` 为 0 | 检查子文件夹中是否有视频 |
| 版本数超过组合数 | 自动调整为 `totalCombinations` |
| 渲染完成但 0 个视频 | 检查 `chapters` 是否传入了完整的 `materials`/`videoMaterials` 和正确的 `path` |
| 导出没有 BGM | 检查 `audioSettings.backgroundMusic.enabled` 是否为 `true`，`bgmList` 是否包含有效路径 |
| `status: "failed"` | 仅展示面向用户的描述，**不暴露内部技术细节** |
| 渲染超时（> 15分钟无进度更新） | 提示视频数量较多，继续等待或减少版本数 |
