---
name: aivo-tts
description: |
  调用容剪(AIVO)语音合成功能，将文本转换为语音音频。
  触发关键词（匹配任意一个）：
  - "语音合成"、"文字转语音"、"TTS合成"、"合成语音"
  - "用xx音色读这段话"、"配音"、"生成语音"、"朗读"
  - "转成语音"、"文字转音频"、"tts"、"合成音频"
  支持导入文案裂变 JSON 文件或直接输入文本，支持选择多种音色。
  需要容剪客户端已启动（端口 7073）。
metadata:
  openclaw:
    emoji: "🔊"
    requires:
      bins:
        - curl
    os:
      - darwin
      - linux
      - win32
---

# 语音合成 (TTS)

> **严格约束（必须遵守）：**
> 1. **只使用本文档中明确列出的 API 路径**，绝对不要猜测或构造文档中没有的 URL
> 2. 基础地址固定为 `http://localhost:7073`
> 3. `/api/health` 返回 `{"status":"ok"}` 即代表服务可用，不要再告诉用户"服务不可用"
> 4. **创建任务的 `shots` 数组中，文本字段名是 `content`（不是 `text`、不是 `items`）**
> 5. **轮询状态返回的 `data.id` 是数字类型，用于 save-to-folder；`taskNo` 是字符串，用于轮询状态**
> 6. **save-to-folder 的 `id` 参数必须是数字类型（如 `12345`），不是字符串**
> 7. 不需要手动传 Authorization 头，Go 服务自动使用缓存的用户 token

## 接口一览（仅限使用这些路径）

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/api/health` | 健康检查 |
| GET | `/api/tts/voices/categories` | 获取音色分类 |
| GET | `/api/tts/voices?category=&language=&keyword=` | 获取音色列表 |
| GET | `/api/tts/voices/preview/{voiceId}` | 音色试听（返回 MP3 流） |
| POST | `/api/tts/member/create` | 创建合成任务 |
| GET | `/api/tts/task/status/{taskNo}` | 查询任务状态 |
| POST | `/api/tts/member/save-to-folder` | 下载音频到本地文件夹 |

将文本转换为语音音频，支持多种音色选择。可以导入文案裂变的 JSON 结果文件，也可以直接输入文本进行合成。

对应容剪界面：工具栏 → 语音合成

---

## 快速示例

> **"用甜美女声帮我合成这段话：大家好，今天给大家推荐一款超好用的面膜"**

> **"导入 D:/fission_result.json 进行语音合成"**

---

## 完整工作流

### Step 1 — 确认服务状态

```bash
curl -s --connect-timeout 3 http://localhost:7073/api/health
```

返回 `{"status":"ok"}` 表示正常。若连接失败，告知用户：**请先启动容剪客户端**。

---

### Step 2 — 获取可用音色列表

#### 2a. 获取音色分类

```bash
curl -s http://localhost:7073/api/tts/voices/categories
```

**返回示例：**
```json
{
  "code": 200,
  "data": {
    "categories": ["通用", "小说", "客服", "直播", "有声书"],
    "languages": ["中文", "英文", "日文", "韩文"]
  }
}
```

#### 2b. 获取音色列表（可按分类/语言/关键词过滤）

```bash
curl -s "http://localhost:7073/api/tts/voices?category=通用&language=中文"
```

**也可以搜索关键词：**
```bash
curl -s "http://localhost:7073/api/tts/voices?keyword=甜美"
```

**返回示例：**
```json
{
  "code": 200,
  "data": {
    "list": [
      {
        "voiceId": "zh_female_sweet",
        "name": "甜美女声",
        "gender": "female",
        "language": "中文",
        "category": "通用",
        "description": "温柔甜美的女性声音，适合情感类内容"
      },
      {
        "voiceId": "zh_male_broadcast",
        "name": "新闻播报",
        "gender": "male",
        "language": "中文",
        "category": "通用",
        "description": "标准男性播音腔"
      }
    ]
  }
}
```

**向用户展示可用音色列表，让用户选择想要使用的音色。** 展示时包含音色名称、性别、类别和描述。

---

### Step 3 — 准备合成文本

文本来源有两种方式：

#### 方式 A：直接输入文本

用户直接提供文本内容，构建合成请求。

#### 方式 B：导入文案裂变 JSON 文件

读取文案裂变 skill 生成的 JSON 文件：

```bash
cat D:/fission_result.json
```

JSON 文件格式：
```json
{
  "title": "超好用面膜推荐",
  "shots": [
    {"id": 1, "name": "开场招呼", "content": "..."},
    {"id": 2, "name": "产品卖点", "content": "..."}
  ],
  "matrix": [
    {
      "id": 1,
      "name": "开场招呼",
      "versions": [
        {"version": 1, "content": "嗨各位宝子们，今天必须给你们安利这款绝绝子面膜"},
        {"version": 2, "content": "家人们看过来！今天的主角是一款我自用超久的面膜"}
      ]
    }
  ]
}
```

从 JSON 中提取需要合成的文本内容。如果有 `matrix`（裂变结果），让用户选择要合成哪些版本。

---

### Step 4 — 用户选择音色

**必须询问用户想要使用哪个音色进行合成。**

展示方式：
```
请选择合成音色：
1. 甜美女声 (zh_female_sweet) - 温柔甜美的女性声音
2. 新闻播报 (zh_male_broadcast) - 标准男性播音腔
3. 活力少女 (zh_female_lively) - 活泼年轻的女性声音
...

请输入编号或音色名称：
```

保存用户选择的 `voiceId`。

---

### Step 5 — 创建合成任务

> **⚠️ 请求格式非常重要，必须严格按照以下格式发送，否则合成会失败或生成错误的音频。**

#### 方式 A：直接输入文本（手动模式）

当用户直接提供文本时，使用 `sourceType: "manual"`，将文本按行拆分放入 `shots` 数组，每行一个对象：

```bash
curl -s -X POST http://localhost:7073/api/tts/member/create \
  -H "Content-Type: application/json" \
  -d '{
    "sourceType": "manual",
    "title": "",
    "voiceId": "zh_female_sweet",
    "voiceName": "甜美女声",
    "shots": [
      {
        "content": "嗨各位宝子们，今天必须给你们安利这款绝绝子面膜"
      },
      {
        "content": "主打一个补水，敷完脸蛋嫩得能掐出水来"
      }
    ],
    "speedRatio": 1.0,
    "audioFormat": "mp3",
    "sampleRate": 24000
  }'
```


#### 方式 B：导入文案裂变结果（矩阵模式）

当从文案裂变 JSON 文件导入时，使用 `sourceType: "copywriting"`，每个选中的版本构建一条 shot：

```bash
curl -s -X POST http://localhost:7073/api/tts/member/create \
  -H "Content-Type: application/json" \
  -d '{
    "sourceType": "copywriting",
    "title": "超好用面膜推荐",
    "voiceId": "zh_female_sweet",
    "voiceName": "甜美女声",
    "shots": [
      {
        "id": "1",
        "name": "开场招呼",
        "version": "版本1",
        "content": "嗨各位宝子们，今天必须给你们安利这款绝绝子面膜"
      },
      {
        "id": "2",
        "name": "产品卖点",
        "version": "版本1",
        "content": "主打一个补水，敷完脸蛋嫩得能掐出水来"
      }
    ],
    "speedRatio": 1.0,
    "audioFormat": "mp3",
    "sampleRate": 24000
  }'
```

**请求参数说明：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `sourceType` | string | 是 | `"manual"`（手动输入）或 `"copywriting"`（文案裂变导入） |
| `shots` | array | 是 | 文本数组，每条包含 `content` 字段（**不是** `text`，**不是** `items`） |
| `shots[].content` | string | 是 | 要合成的文本内容 |
| `shots[].id` | string | 否 | 分镜 ID（仅 copywriting 模式） |
| `shots[].name` | string | 否 | 分镜名称（仅 copywriting 模式） |
| `shots[].version` | string | 否 | 版本标识（仅 copywriting 模式） |
| `voiceId` | string | 是 | 音色 ID（从 Step 2 获取的 `voiceId`） |
| `voiceName` | string | 是 | 音色名称（从 Step 2 获取的 `name`） |
| `title` | string | 否 | 任务标题（手动模式可留空） |
| `speedRatio` | float | 是 | 语速倍率，默认 `1.0`（范围 0.5-2.0） |
| `audioFormat` | string | 是 | 音频格式，固定填 `"mp3"` |
| `sampleRate` | int | 是 | 采样率，固定填 `24000` |

**注意：** 不需要手动传 `Authorization` 头。Go 服务会自动使用当前登录用户的 token 进行认证（前提是用户已在容剪客户端登录）。如果返回"无效token"错误，提示用户先在容剪客户端登录账号。

**返回示例：**
```json
{
  "code": 200,
  "data": {
    "taskNo": "tts_task_20260311_abc123"
  }
}
```

保存 `taskNo` 用于轮询。

---

### Step 6 — 轮询合成状态

每 **3秒** 执行一次，使用 Step 5 返回的 `taskNo`：

```bash
curl -s "http://localhost:7073/api/tts/task/status/tts_task_20260311_abc123"
```

**任务状态码说明：**

| taskStatus | 含义 |
|------------|------|
| 0 | 待处理 |
| 1 | 合成中 |
| 2 | 全部完成 |
| 3 | 部分失败 |
| 4 | 全部失败 |

**合成中返回（taskStatus = 0 或 1）：**
```json
{
  "code": 200,
  "data": {
    "taskStatus": 1,
    "id": 12345,
    "taskNo": "tts_task_20260311_abc123"
  }
}
```

**完成返回（taskStatus = 2）：**
```json
{
  "code": 200,
  "data": {
    "taskStatus": 2,
    "id": 12345,
    "taskNo": "tts_task_20260311_abc123"
  }
}
```

当 `taskStatus` 为 0 或 1 时继续轮询，当 `taskStatus >= 2` 时停止轮询。

保存 `data.id`（数字类型，**不是** `taskNo`），Step 7 需要用 `id` 来下载。

向用户展示进度：`正在合成语音，请稍候...`

---

### Step 7 — 下载合成结果到本地（必须执行）

> **此步骤不可跳过。** Step 6 完成后音频仅存在于远程服务器，必须调用 `save-to-folder` 接口才能将音频下载保存到用户的本地电脑。

合成完成后（`taskStatus >= 2`），使用 Step 6 中获取的 `data.id`（数字类型）调用 `save-to-folder` 接口：

```bash
curl -s -X POST http://localhost:7073/api/tts/member/save-to-folder \
  -H "Content-Type: application/json" \
  -d '{
    "id": 12345,
    "destPath": "D:/tts-output"
  }'
```

**参数说明：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | int | 是 | Step 6 轮询返回的 `data.id`（数字类型，**不是** `taskNo` 字符串） |
| `destPath` | string | 是 | 本地保存目录。如果用户指定了路径则使用用户的路径；如果未指定，使用桌面路径（Windows: `%USERPROFILE%/Desktop`，macOS/Linux: `$HOME/Desktop`），**不要写死用户名** |

**返回示例：**
```json
{
  "code": 200,
  "data": {
    "savedPath": "D:/tts-output/超好用面膜推荐"
  },
  "msg": "保存成功"
}
```

`data.savedPath` 是实际保存的文件夹路径，音频文件（MP3 格式）保存在该文件夹内。

---

### Step 8 — 告知用户结果

下载成功后，向用户展示：

```
✅ 语音合成完成！

| 信息 | 详情 |
|------|------|
| 音色 | 甜美女声 |
| 文本条数 | 2 |
| 保存路径 | D:/tts-output/超好用面膜推荐 |
```

告知用户：
- 合成的音色名称
- 合成的文本条数
- **本地音频保存路径**：Step 7 的 `data.savedPath`

---

## 接口一览

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/api/tts/voices/categories` | 获取音色分类 |
| GET | `/api/tts/voices?category=&language=&keyword=` | 获取音色列表 |
| GET | `/api/tts/voices/preview/{voiceId}` | 音色试听（返回 MP3 流） |
| POST | `/api/tts/member/create` | 创建合成任务 |
| GET | `/api/tts/task/status/{taskNo}` | 查询任务状态 |
| POST | `/api/tts/member/save-to-folder` | 下载音频到本地文件夹 |

---

## 错误处理

> **重要：不要向用户展示原始的后端错误信息（如 IP 地址、端口号、内部服务名、日志路径等），应转换为用户友好的提示。**

| 错误情况 | 处理方式 |
|----------|----------|
| 服务连接失败 | 提示用户：**"容剪服务未启动，请先打开容剪客户端"** |
| 获取音色列表失败 | 提示用户：**"语音合成服务暂时不可用，请检查网络连接并重试"** |
| JSON 文件读取失败 | 检查文件路径是否正确，文件格式是否为有效 JSON |
| 合成任务创建失败 | 检查文本内容是否为空，音色 ID 是否有效 |
| `status: "failed"` | 提示用户：**"语音合成失败，请检查文本内容后重试"**，不要直接展示系统内部错误信息 |
| 下载保存失败 | 检查目标目录是否有写入权限 |
| 合成超时（> 5分钟） | 文本过长时合成耗时较长，建议拆分为更小的文本段 |
| 其他未知错误 | 提示用户：**"操作失败，请重试"**，不要暴露内部错误堆栈 |

---

## 附：音色试听

在用户选择音色前，可以让用户试听：

```bash
curl -s "http://localhost:7073/api/tts/voices/preview/{voiceId}" --output preview.mp3
```

这会下载一段音色试听音频（MP3 格式）。首次试听某个音色可能需要较长时间（最长 60 秒），因为需要进行实时合成。
