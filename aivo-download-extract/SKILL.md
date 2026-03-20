---
name: aivo-download-extract
description: |
  给视频链接，自动下载视频到本地并提取文案（语音转文字）。
  触发关键词（匹配任意一个）：
  - "下载并提取文案"、"提取视频文案"、"把视频文案提取出来"、"下载视频提取字幕"
  - "提取文案"、"视频转文字"、"语音转文字"、"提取字幕"
  - "文案提取"、"ASR"、"视频文案"
  - 用户给了一个视频链接并要求"提取文案"、"提取文字"
  整合了视频下载和文案提取（ASR语音识别）功能。
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

给视频分享链接，自动下载视频到本地，并通过 ASR 语音识别提取视频中的文案（语音转文字）。

提取的文案可直接保存为文本文件，也可继续流转到文案裂变、语音合成等技能。

---

## 前提条件

- 容剪客户端已启动（Go 服务运行在 localhost:7073）
- 用户已在容剪客户端登录（提取文案接口需要认证）
- 手动调用 curl 时，需要在请求头添加 `Authorization: Bearer <token>`
  - 建议设置环境变量保存 token：
    - macOS/Linux（临时会话）: `export AIVO_TOKEN="<your_token>"`
    - Windows PowerShell（临时会话）: `$env:AIVO_TOKEN="<your_token>"`

> token 来源：容剪客户端登录后缓存于本机（仅供本机调试）；也可由前端自动携带。不要在公共环境泄露。

---

## 完整工作流

### Step 1 — 确认服务状态

```bash
curl -s --connect-timeout 3 http://localhost:7073/api/health
```

返回 `{"status":"ok"}` 表示正常。若连接失败，提示用户：**"容剪服务未启动，请先打开容剪客户端"**。

再检查视频下载服务：

```bash
curl -s --connect-timeout 3 http://localhost:7073/api/video-download/health
```

返回 `{"status":"ok"}` 表示正常。若失败，提示用户：**"视频下载服务暂时不可用，请确认容剪客户端已完全启动"**。

---

### Step 2 — 解析视频信息

发送分享链接到解析接口，服务端会提取视频信息并在远程完成下载：

```bash
curl -s -X POST http://localhost:7073/api/video-download \
  -H "Content-Type: application/json" \
  -d '{
    "text": "https://v.douyin.com/ia-nnT8Ysck/"
  }'
```

**也支持从分享文案中自动提取链接：**

```bash
curl -s -X POST http://localhost:7073/api/video-download \
  -H "Content-Type: application/json" \
  -d '{
    "text": "7.75 Njt:/ 这个宝宝太可爱了 https://v.douyin.com/ia-nnT8Ysck/"
  }'
```

**成功响应示例：**
```json
{
  "retcode": 200,
  "retdesc": "成功",
  "succ": true,
  "data": {
    "video_id": "7519781625333779770",
    "platform": "抖音",
    "title": "这个宝宝太可爱了",
    "download_url": "/downloads/20260310/douyin/7519781625333779770.mp4",
    "video_url": "https://www.douyin.com/aweme/v1/play/...",
    "cover_url": "https://p3-sign.douyinpic.com/...",
    "fallback_crawl": false
  }
}
```

**关键字段：** 从 `data.download_url` 获取文件的相对下载路径（如 `/downloads/20260310/douyin/xxx.mp4`）。

> 如果响应中没有 `download_url` 但有 `local_file_path`，可从 `local_file_path` 中提取 `/downloads/` 之后的部分作为 `download_url`。例如 `local_file_path` 为 `/www/wwwroot/video-extractor/downloads/20260312/douyin/xxx.mp4`，则 `download_url` 为 `/downloads/20260312/douyin/xxx.mp4`。

**保存以下信息：**
- `data.download_url` — 远程文件相对路径（用于保存到本地）
- `data.title` — 视频标题
- `data.platform` — 平台名称

---

### Step 3 — 下载视频到本地

使用 `save-to-local` 接口将视频下载到用户本地磁盘：

```bash
curl -s -X POST http://localhost:7073/api/video-download/save-to-local \
  -H "Content-Type: application/json" \
  -d '{
    "download_url": "/downloads/20260310/douyin/7519781625333779770.mp4",
    "save_dir": "",
    "file_name": ""
  }'
```

**参数说明：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `download_url` | string | 是 | Step 2 返回的 `data.download_url`（必须以 `/downloads/` 开头） |
| `save_dir` | string | 否 | 本地保存目录。为空时自动使用默认目录（Windows: `%USERPROFILE%/Downloads/aivo-videos`，macOS/Linux: `~/Downloads/aivo-videos`） |
| `file_name` | string | 否 | 自定义文件名（不含扩展名）。为空时使用原始文件名 |

**save_dir 规则：**
- 如果用户指定了保存目录，使用 `save_dir` 传入
- 如果用户未指定，`save_dir` 传空字符串 `""`，服务会自动创建默认目录
- 服务会自动创建不存在的目录

**成功响应示例：**
```json
{
  "retcode": 200,
  "retdesc": "文件已保存到本地",
  "succ": true,
  "data": {
    "local_path": "<用户下载目录>/aivo-videos/7519781625333779770.mp4",
    "file_size": 15728640,
    "save_dir": "<用户下载目录>/aivo-videos"
  }
}
```

**保存 `data.local_path`**，这是下载到本地的视频文件路径，后续步骤使用。

告知用户：**视频已下载到本地，开始提取文案...**

---

### Step 4 — 本地上传视频提取文案（需要 Authorization）

使用 Step 3 返回的本地视频文件路径，调用本地上传接口提取文案。

- macOS/Linux
```bash
curl -s -X POST http://localhost:7073/api/parse-task/extract-from-upload \
  -H "Authorization: Bearer $AIVO_TOKEN" \
  -F "file=@<Step 3 返回的 data.local_path>"
```

- Windows PowerShell
```powershell
curl -s -X POST http://localhost:7073/api/parse-task/extract-from-upload \
  -H "Authorization: Bearer $env:AIVO_TOKEN" \
  -F "file=@<Step 3 返回的 data.local_path>"
```

**成功响应示例：**
```json
{
  "retcode": 200,
  "retdesc": "成功",
  "succ": true,
  "data": {
    "text": "的妈呀，个位数居然让我薅到4个这么好看的陶瓷碗呢...",
    "duration_ms": 27569,
    "audio_url": "https://aivo.oss-cn-beijing.aliyuncs.com/media_convert/aivo/video/20260320/upload/c128a22f49c04675ae7231736ab3a953.aac",
    "oss_video_url": "https://aivo.oss-cn-beijing.aliyuncs.com/aivo/video/20260320/upload/c128a22f49c04675ae7231736ab3a953.mp4"
  }
}
```

**响应字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `retcode` | number | 状态码，200 表示成功 |
| `retdesc` | string | 状态描述 |
| `succ` | bool | 是否成功 |
| `data.text` | string | 提取的文案内容 |
| `data.duration_ms` | number | 音视频时长（毫秒） |
| `data.audio_url` | string | 音频文件下载地址（如果有） |
| `data.oss_video_url` | string | 视频文件 OSS 地址 |

---

### Step 5 — 保存文案并告知用户

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
| 文案保存路径 | D:/copywriting/这个宝宝太可爱了_文案.txt |
| 文案时长 | 25.6 秒 |

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

每个链接独立走完 Step 2-5 的完整流程。建议串行处理（等一个完成再处理下一个）。

---

## 接口一览

所有接口均通过容剪 Go 服务（localhost:7073）代理调用。手动 curl 时，请添加 `Authorization: Bearer <token>` 头（前端调用会自动携带）。

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/api/health` | 检查容剪服务状态 |
| GET | `/api/video-download/health` | 检查视频下载服务状态 |
| POST | `/api/video-download` | 解析视频链接并在远程下载 |
| POST | `/api/video-download/save-to-local` | 将远程视频文件下载到本地 |
| POST | `/api/parse-task/extract-from-upload` | 本地上传视频提取文案（ASR） |

---

## 与其他技能的联动

提取到的文案可以直接流转到其他技能继续使用：

```
视频链接 → 下载视频到本地 → 提取文案
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

> 重要：不要向用户展示原始的后端错误信息（如 IP 地址、端口号、内部服务名、日志路径等），应转换为用户友好的提示。

| 错误情况 | 处理方式 |
|----------|----------|
| 服务连接失败 | 提示用户：**"容剪服务未启动，请先打开容剪客户端"** |
| 认证失败（401/403） | 提示用户：**"请先在容剪客户端中登录账号"**，如仍失败请重新登录刷新 token |
| 视频下载服务不可用 | 提示用户：**"视频下载服务暂时不可用，请确认容剪客户端已完全启动"** |
| 视频解析或下载失败 | 提示用户：**"视频下载失败，请检查链接是否有效"** |
| 文案提取接口返回错误 | 提示用户：**"文案提取失败，该视频可能没有语音内容"** |
| `text` 字段为空 | 提示用户：**"视频中未检测到语音内容"** |
| curl 下载/上传失败 | 检查目标目录是否有写入权限，磁盘空间是否充足、路径/引号是否正确 |
| 其他未知错误 | 提示用户：**"操作失败，请重试"**，不要暴露内部错误堆栈 |
