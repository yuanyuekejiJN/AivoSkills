---
name: aivo-download-extract
description: |
  给视频链接，自动解析视频并提取文案（语音转文字），同时下载视频和音频到本地。
  触发关键词（匹配任意一个）：
  - "下载并提取文案"、"提取视频文案"、"把视频文案提取出来"、"下载视频提取字幕"
  - "提取文案"、"视频转文字"、"语音转文字"、"提取字幕"
  - "文案提取"、"ASR"、"视频文案"
  - 用户给了一个视频链接并要求"提取文案"、"提取文字"
  整合了视频解析、视频/音频下载和文案提取（ASR语音识别）功能。
  需要容剪客户端已启动且用户已登录（端口 7073）。
metadata:
  openclaw:
    emoji: "📋"
    requires:
      bins:
        - curl
    os:
      - darwin
      - linux
      - win32
---

# 视频下载 + 文案提取

给视频分享链接，自动解析视频、下载视频和音频到本地，并通过 ASR 语音识别提取视频中的文案（语音转文字）。

提取的文案可直接保存为文本文件，也可继续流转到文案裂变、语音合成等技能。

---

## 快速示例

> **"下载 https://v.douyin.com/ia-nnT8Ysck/ 并提取文案"**

> **"帮我提取这个视频的文案 https://v.douyin.com/ia-nnT8Ysck/ ，保存到 D:/copywriting"**

---

## 前提条件

- 容剪客户端已启动（Go 服务运行在 localhost:7073）
- **用户已在容剪客户端登录**（文案提取接口需要认证，Go 服务自动使用缓存的用户 token）

---

## 完整工作流

### Step 1 — 确认服务状态

```bash
curl -s --connect-timeout 3 http://localhost:7073/api/health
```

返回 `{"status":"ok"}` 表示正常。若连接失败，提示 **"容剪服务未启动，请先打开容剪客户端"**。

---

### Step 2 — 创建视频解析任务

将视频链接提交到解析接口，yyac-server 会自动下载视频、提取音频：

```bash
curl -s -X POST http://localhost:7073/api/parse-task/extract-video \
  -H "Content-Type: application/json" \
  -d '{
    "sourceUrl": "https://v.douyin.com/ia-nnT8Ysck/"
  }'
```

**也支持从分享文案中提取链接：**

```bash
curl -s -X POST http://localhost:7073/api/parse-task/extract-video \
  -H "Content-Type: application/json" \
  -d '{
    "sourceUrl": "7.75 Njt:/ 这个宝宝太可爱了 https://v.douyin.com/ia-nnT8Ysck/"
  }'
```

**注意：** 不需要手动传 `Authorization` 头。Go 服务会自动使用当前登录用户的缓存 token 进行认证（前提是用户已在容剪客户端登录）。如果返回认证错误，提示用户先在容剪客户端登录账号。

**成功响应示例：**
```json
{
  "code": 200,
  "data": {
    "id": 12345,
    "taskNo": "PT20260314001",
    "title": "这个宝宝太可爱了",
    "sourceUrl": "https://v.douyin.com/ia-nnT8Ysck/",
    "platform": "抖音",
    "taskStatus": "processing",
    "createdAt": "2026-03-14T10:30:00",
    "video": {
      "ossUrl": "https://oss.example.com/video/xxx.mp4",
      "downloadUrl": "https://cdn.example.com/video/xxx.mp4",
      "fileFormat": "mp4"
    },
    "audio": {
      "downloadUrl": "https://cdn.example.com/audio/xxx.aac",
      "fileFormat": "aac",
      "convertStatus": 0
    }
  }
}
```

**保存以下信息：**
- `data.id` — 任务 ID（后续所有步骤使用）
- `data.title` — 视频标题
- `data.platform` — 平台名称
- `data.video.ossUrl` 或 `data.video.downloadUrl` — 视频下载地址（优先用 `ossUrl`）
- `data.video.fileFormat` — 视频格式（默认 mp4）
- `data.audio.downloadUrl` — 音频下载地址
- `data.audio.fileFormat` — 音频格式（默认 aac）

告知用户：**视频解析任务已创建，正在处理...**

---

### Step 3 — 下载视频和音频到本地

使用 Step 2 返回的 OSS/CDN 地址，将视频和音频文件下载到用户本地。

**保存目录规则：**
- 如果用户指定了保存目录，使用用户指定的路径
- 如果未指定，默认使用：Windows `%USERPROFILE%/Downloads/aivo-videos`，macOS/Linux `~/Downloads/aivo-videos`

**文件名规则：** 使用视频标题（去除文件名非法字符），截断到 100 字符。

#### 3a. 下载视频

```bash
curl -s -L -o "D:/copywriting/这个宝宝太可爱了.mp4" "https://oss.example.com/video/xxx.mp4"
```

> 使用 `data.video.ossUrl`（优先）或 `data.video.downloadUrl`。如果两个地址都为空，跳过视频下载并告知用户"视频暂未就绪"。

#### 3b. 下载音频

```bash
curl -s -L -o "D:/copywriting/这个宝宝太可爱了.aac" "https://cdn.example.com/audio/xxx.aac"
```

> 使用 `data.audio.downloadUrl`。如果音频地址为空或 `convertStatus` 不为 2，音频可能还在转码中——可以跳过音频下载，后续从任务列表重新获取。

告知用户视频和音频下载情况。

> **注意：** `video` 和 `audio` 字段在任务刚创建时可能暂时没有下载地址（正在处理中）。如果地址为空，可以在 Step 4 轮询时重新检查，从任务列表中获取最新的下载地址后再下载。

---

### Step 4 — 触发文案提取

使用 Step 2 返回的 `data.id` 触发文案提取（ASR 语音识别）：

```bash
curl -s -X POST http://localhost:7073/api/parse-task/extract-transcript \
  -H "Content-Type: application/json" \
  -d '{
    "taskId": 12345
  }'
```

**注意：** 同样不需要手动传 `Authorization` 头，Go 服务自动使用缓存 token。

**成功响应示例：**
```json
{
  "code": 200,
  "message": "文案提取任务已提交",
  "data": {
    "taskId": 12345,
    "extractStatus": 1
  }
}
```

`extractStatus` 状态码：
- `0` — 未提取
- `1` — 提取中
- `2` — 提取完成

---

### Step 5 — 轮询任务状态直到文案提取完成

每 **3秒** 查询一次任务列表，检查目标任务的文案提取状态：

```bash
curl -s "http://localhost:7073/api/parse-task/list?pageNo=1&pageSize=20"
```

**响应示例：**
```json
{
  "code": 200,
  "data": {
    "rows": [
      {
        "id": 12345,
        "title": "这个宝宝太可爱了",
        "platform": "抖音",
        "taskStatus": "success",
        "video": {
          "ossUrl": "https://oss.example.com/video/xxx.mp4",
          "downloadUrl": "https://cdn.example.com/video/xxx.mp4",
          "fileFormat": "mp4"
        },
        "audio": {
          "downloadUrl": "https://cdn.example.com/audio/xxx.aac",
          "fileFormat": "aac",
          "convertStatus": 2
        },
        "transcript": {
          "extractStatus": 2,
          "content": "大家好，今天给大家推荐一款超好用的面膜，补水效果特别好..."
        }
      }
    ]
  }
}
```

在 `data.rows` 中找到 `id` 匹配的任务，检查 `transcript.extractStatus`：

- `0` — 未提取，说明 Step 4 可能未成功触发，重新调用 Step 4
- `1` — 提取中，继续轮询（向用户展示进度提示）
- `2` — 提取完成，从 `transcript.content` 获取文案内容

**补充下载：** 如果 Step 3 因为地址为空跳过了视频或音频下载，在轮询到 `taskStatus: "success"` 后可以重新用列表中最新的 `video` 和 `audio` 地址进行下载。

**超时处理：** 如果轮询超过 **120秒**（40次）仍未完成，提示用户：**"文案提取时间较长，请稍后在容剪客户端中查看结果"**。

---

### Step 6 — 保存文案并告知用户

文案提取完成后：

1. **保存文案为文本文件**：
   - 如果用户指定了保存路径，保存到指定路径
   - 如果未指定，保存到与视频相同的目录，文件名格式：`{视频标题}_文案.txt`

2. **展示完整结果**给用户：

```
视频下载 + 文案提取完成！

| 信息 | 详情 |
|------|------|
| 视频标题 | 这个宝宝太可爱了 |
| 平台 | 抖音 |
| 视频保存路径 | D:/copywriting/这个宝宝太可爱了.mp4 |
| 音频保存路径 | D:/copywriting/这个宝宝太可爱了.aac |
| 文案保存路径 | D:/copywriting/这个宝宝太可爱了_文案.txt |

提取的文案内容：
---
大家好，今天给大家推荐一款超好用的面膜，补水效果特别好...
---

提示：可以将文案内容用于文案裂变或语音合成。
```

---

## 批量处理

支持一次传入多个链接，依次执行完整流程：

```
用户：帮我下载并提取文案：
1. https://v.douyin.com/ia-nnT8Ysck/
2. https://v.kuaishou.com/abc123
```

每个链接独立走完 Step 2-6 的完整流程。建议串行处理（等一个完成再处理下一个），因为文案提取需要等待 ASR 结果。

---

## 接口一览

所有接口均通过容剪 Go 服务（localhost:7073）代理调用，**不需要手动传 Authorization 头**（Go 服务自动使用缓存的用户 token）：

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/api/health` | 检查容剪服务状态 |
| POST | `/api/parse-task/extract-video` | 创建视频解析任务（传入视频链接） |
| POST | `/api/parse-task/extract-transcript` | 触发文案提取（ASR 语音识别） |
| GET | `/api/parse-task/list` | 查询任务列表（含视频/音频/文案信息） |

---

## 响应字段说明

### 解析任务接口 (`/api/parse-task/extract-video`) ���应

| 字段 | 类型 | 说明 |
|------|------|------|
| `data.id` | int64 | 任务 ID（后续步骤使用） |
| `data.taskNo` | string | 任务编号 |
| `data.title` | string | 视频标题 |
| `data.platform` | string | 平台名称 |
| `data.taskStatus` | string | 任务状态 |
| `data.video.ossUrl` | string | 视频 OSS 地址（优先用于下载） |
| `data.video.downloadUrl` | string | 视频 CDN 下载地址（备选） |
| `data.video.fileFormat` | string | 视频格式（mp4） |
| `data.audio.downloadUrl` | string | 音频下载地址 |
| `data.audio.fileFormat` | string | 音频格式（aac） |
| `data.audio.convertStatus` | int | 音频转码状态：0=待转码，1=转码中，2=成功，3=失败 |

### 任务列表接口 (`/api/parse-task/list`) 响应

| 字段 | 类型 | 说明 |
|------|------|------|
| `data.rows[].id` | int64 | 任务 ID |
| `data.rows[].title` | string | 视频标题 |
| `data.rows[].platform` | string | 平台名称 |
| `data.rows[].taskStatus` | string | 任务状态（processing / success / failure） |
| `data.rows[].video.ossUrl` | string | 视频 OSS 地址 |
| `data.rows[].video.downloadUrl` | string | 视频 CDN 下载地址 |
| `data.rows[].audio.downloadUrl` | string | 音频下载地址 |
| `data.rows[].audio.convertStatus` | int | 音频转码状态 |
| `data.rows[].transcript.extractStatus` | int | 文案提取状态：0=未提取，1=提取中，2=已完成 |
| `data.rows[].transcript.content` | string | 提取的文案内容（extractStatus=2 时有值） |

---

## 与其他技能的联动

提取到的文案可以直接流转到其他技能继续使用：

```
视频链接 → 解析+下载视频/音频 → 提取文案
                                    ↓
                              文案裂变（生成多版本改写）
                                    ↓
                              语音合成（文案转语音配音）
```

**联动方式：**
- 将提取的文案传递给 `aivo-copywriting-fission` 技能进行裂变
- 将文案直接传递给 `aivo-tts` 技能进行语音合成

---

## 错误处理

> **重要：不要向用户展示原始的后端错误信息（如 IP 地址、端口号、内部服务名、日志路径等），应转换为用户友好的提示。**

| 错误情况 | 处理方式 |
|----------|----------|
| 服务连接失败 | 提示用户：**"容剪服务未启动，请先打开容剪客户端"** |
| 认证失败（401/403） | 提示用户：**"请先在容剪客户端中登录账号"** |
| `extract-video` 返回错误 | 提示用户：**"视频解析失败，请检查链接是否有效"** |
| 视频/音频下载地址为空 | 任务可能还在处理中，在轮询时重新获取 |
| `extract-transcript` 返回错误 | 提示用户：**"文案提取失败，该视频可能没有语音内容"** |
| 轮询超时（120秒） | 提示用户：**"文案提取时间较长，请稍后在容剪客户端中查看结果"** |
| `transcript.content` 为空 | 提示用户：**"视频中未检测到语音内容"** |
| curl 下载文件失败 | 检查目标目录是否有写入权限，磁盘空间是否充足 |
| 其他未知错误 | 提示用户：**"操作失败，请重试"**，不要暴露内部错误堆栈 |
