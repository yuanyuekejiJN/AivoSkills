---
name: aivo-video-download
description: |
  调用容剪(AIVO)视频下载功能，通过分享链接提取并下载视频到本地。
  触发关键词（匹配任意一个）：
  - "下载视频"、"提取视频"、"保存这个视频"、"把视频下载下来"
  - "下载这个链接"、"帮我下这个视频"、"视频下载"、"去水印下载"
  - 用户给了一个视频平台的分享链接（抖音、快手、B站、小红书等）
  支持抖音、快手、小红书、哔哩哔哩、好看视频、微视、梨视频、皮皮搞笑等平台。
  需要容剪客户端已启动（端口 7073）。
metadata:
  openclaw:
    emoji: "📥"
    requires:
      bins:
        - curl
    os:
      - darwin
      - linux
      - win32
---

# 视频下载提取

> **严格约束（必须遵守）：**
> 1. **只使用本文档中明确列出的 API 路径**，绝对不要猜测或构造文档中没有的 URL
> 2. 基础地址固定为 `http://localhost:7073`
> 3. `/api/health` 返回 `{"status":"ok"}` 即代表服务可用
> 4. 视频下载路径是 `/api/video-download`（连字符），**不是** `/api/video/download`（斜杠）
> 5. **Step 2 只是远程下载，必须执行 Step 3 save-to-local 才能把视频保存到用户本地**

通过分享链接解析并下载视频到**本地磁盘**，支持多个主流视频平台。

所有请求通过容剪本地服务代理处理，视频文件最终保存到用户本地目录。

---

## 支持的平台

| 平台 | 示例链接格式 |
|------|-------------|
| 抖音 | `https://v.douyin.com/xxx/` |
| 快手 | `https://v.kuaishou.com/xxx` |
| 小红书 | `https://www.xiaohongshu.com/explore/xxx` |
| 哔哩哔哩 | `https://www.bilibili.com/video/BVxxx` |
| 好看视频 | `https://haokan.baidu.com/v?vid=xxx` |
| 微视 | `https://isee.weishi.qq.com/ws/app-pages/share/...` |
| 梨视频 | `https://www.pearvideo.com/video_xxx` |
| 皮皮搞笑 | `https://h5.pipigx.com/pp/post/xxx` |

---

## 输出路径规则

> **如果用户没有指定保存目录，`save_dir` 传空字符串 `""`，服务会自动使用默认目录并告知用户。**

- 用户指定了路径 → 使用 `save_dir` 传入
- 用户未指定 → `save_dir` 传 `""`，服务自动使用默认下载目录（Windows: `%USERPROFILE%/Downloads/aivo-videos`，macOS/Linux: `~/Downloads/aivo-videos`）
- 服务会自动创建不存在的目录

---

## 快速示例

> **"帮我下载这个视频 https://v.douyin.com/ia-nnT8Ysck/"**

> **"下载这个视频到 D:/我的视频 https://v.douyin.com/ia-nnT8Ysck/"**

---

## 完整工作流

### Step 1 — 确认服务状态

先检查容剪服务：

```bash
curl -s --connect-timeout 3 http://localhost:7073/api/health
```

返回 `{"status":"ok"}` 表示正常。若连接失败，告知用户：**"容剪服务未启动，请先打开容剪客户端"**。

再检查视频下载服务：

```bash
curl -s --connect-timeout 3 http://localhost:7073/api/video-download/health
```

返回 `{"status":"ok"}` 表示正常。若失败，告知用户：**"视频下载服务暂时不可用，请确认容剪客户端已完全启动"**。

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
    "text": "7.75 Njt:/ M@f.Sv 02/27 这个宝宝太可爱了# 萌宝 # 可爱  https://v.douyin.com/ia-nnT8Ysck/"
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

> **注意：** 如果响应中没有 `download_url` 字段但有 `local_file_path`，则从 `local_file_path` 中提取 `/downloads/` 之后的部分作为 `download_url`。例如 `local_file_path` 为 `/www/wwwroot/video-extractor/downloads/20260312/douyin/xxx.mp4`，则 `download_url` 为 `/downloads/20260312/douyin/xxx.mp4`。

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

---

### Step 4 — 告知用户结果

下载成功后，向用户展示：
- 视频标题：Step 2 的 `data.title`
- 平台来源：Step 2 的 `data.platform`
- **本地保存路径**：Step 3 的 `data.local_path`
- **保存目录**：Step 3 的 `data.save_dir`
- 文件大小：Step 3 的 `data.file_size`（转为 MB 显示，格式如 `15.0 MB`）

**示例展示格式：**
```
✅ 视频下载成功！

| 信息 | 详情 |
|------|------|
| 标题 | 这个宝宝太可爱了 |
| 平台 | 抖音 |
| 保存路径 | <实际返回的 local_path> |
| 文件大小 | 15.0 MB |
```

---

## 接口一览

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/api/video-download/health` | 检查视频下载服务状态 |
| POST | `/api/video-download` | 解析视频链接并在远程下载 |
| POST | `/api/video-download/save-to-local` | 将远程视频文件下载到本地 |

---

## 响应字段说明

### 解析接口 (`/api/video-download`) 响应

| 字段 | 类型 | 说明 |
|------|------|------|
| `data.video_id` | string | 视频 ID |
| `data.platform` | string | 平台名称 |
| `data.title` | string | 视频标题 |
| `data.download_url` | string | 远程文件相对路径（用于 save-to-local 接口） |
| `data.video_url` | string | 视频直链（兜底方案时可能为空） |
| `data.cover_url` | string | 封面图 URL |
| `data.fallback_crawl` | bool | 是否通过兜底方案下载 |

### 保存接口 (`/api/video-download/save-to-local`) 响应

| 字段 | 类型 | 说明 |
|------|------|------|
| `data.local_path` | string | 本地文件完整路径 |
| `data.file_size` | int64 | 文件大小（字节） |
| `data.save_dir` | string | 实际使用的保存目录 |

---

## 错误处理

> **重要：不要向用户展示原始的后端错误信息（如 IP 地址、端口号、内部服务名、日志路径等），应转换为用户友好的提示。**

| 错误情况 | 处理方式 |
|----------|----------|
| 服务连接失败 | 提示用户：**"容剪服务未启动，请先打开容剪客户端"** |
| 视频下载服务不可用 | 提示用户：**"视频下载服务暂时不可用，请确认容剪客户端已完全启动"** |
| `text` 参数为空 | 提示用户提供视频分享链接 |
| 平台不支持 | 显示当前支持的平台列表 |
| 解析成功但无 `download_url` | 检查 `local_file_path` 字段提取路径，若都没有则告知用户解析失败 |
| 保存到本地失败 | 提示用户检查本地磁盘空间和目录权限 |
| `succ` 为 false | 仅展示 `retdesc` 中面向用户的描述，**不要展示包含 IP、端口、服务名等技术细节的原始错误** |
| 其他未知错误 | 提示用户：**"视频下载失败，请检查链接是否正确并重试"**，不要暴露内部错误堆栈 |
