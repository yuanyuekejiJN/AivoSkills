---
name: aivo-copywriting-fission
description: |
  调用容剪(AIVO)文案裂变功能，将文案自动拆分为分镜并生成多个裂变版本。
  触发关键词（匹配任意一个）：
  - "文案裂变"、"裂变文案"、"文案改写多版本"、"帮我裂变这段文案"
  - "文案生成"、"AI改写文案"、"多版本文案"、"拆分文案"
  - "生成多个版本的文案"、"文案创作"
  支持分镜维度裂变和整段裂变两种模式。
  通过容剪 Go 服务（localhost:7073）代理调用后端文案处理接口。
metadata:
  openclaw:
    emoji: "📝"
    requires:
      bins:
        - curl
    os:
      - darwin
      - linux
      - win32
---

# 文案裂变

> **严格约束（必须遵守）：**
> 1. **只使用本文档中明确列出的 API 路径**，绝对不要猜测或构造文档中没有的 URL
> 2. 基础地址固定为 `http://localhost:7073`
> 3. `/api/health` 返回 `{"status":"ok"}` 即代表服务可用
> 4. 裂变接口调用可能需要 30-60 秒（AI 生成），设置足够长的超时

将原始文案自动拆分为分镜，然后 AI 生成多个裂变版本，支持保存结果到本地 JSON 文件。

---

## 快速示例

> **"帮我裂变这段文案，生成3个版本：大家好，今天给大家推荐一款超好用的面膜，补水效果特别好，用完皮肤水嫩嫩的..."**

对应参数：
- `raw_text` = 用户提供的文案
- `version_count` = `3`
- `style` = `带货直播口语`（默认）

---

## 完整工作流

### Step 1 — 确认服务状态

文案裂变 API 通过容剪 Go 服务代理调用，先检查 Go 服务是否正常：

```bash
curl -s --connect-timeout 3 http://localhost:7073/api/health
```

若返回 `{"status":"ok"}`，服务正常。

再检查文案裂变后端是否可达：

```bash
curl -s --connect-timeout 5 http://localhost:7073/api/copywriting/health
```

若连接失败，告知用户：**请确认容剪 Go 服务已启动（默认端口 7073）**。

> **注意**：所有文案裂变请求都通过 Go 服务（localhost:7073）代理转发，不直接调用 Python 服务。

---

### Step 2 — 拆分并命名分镜（推荐方式）

```bash
curl -s -X POST http://localhost:7073/api/copywriting/split-and-name \
  -H "Content-Type: application/json" \
  -d '{
    "raw_text": "大家好，今天给大家推荐一款超好用的面膜，补水效果特别好，用完皮肤水嫩嫩的，原价199，今天直播间只要99，数量有限，赶紧下单！",
    "style_hint": "带货直播"
  }'
```

**返回示例：**
```json
{
  "retcode": 200,
  "succ": true,
  "data": {
    "title": "超好用面膜推荐",
    "shots": [
      {"id": 1, "name": "开场招呼", "content": "大家好，今天给大家推荐一款超好用的面膜"},
      {"id": 2, "name": "产品卖点", "content": "补水效果特别好，用完皮肤水嫩嫩的"},
      {"id": 3, "name": "价格促销", "content": "原价199，今天直播间只要99，数量有限，赶紧下单！"}
    ]
  }
}
```

向用户展示分镜拆分结果：标题、各分镜名称和内容。

---

### Step 3 — 裂变文案（分镜维度）

将 Step 2 得到的 `shots` 传入裂变接口：

```bash
curl -s -X POST http://localhost:7073/api/copywriting/fission \
  -H "Content-Type: application/json" \
  -d '{
    "shots": [
      {"id": 1, "name": "开场招呼", "content": "大家好，今天给大家推荐一款超好用的面膜"},
      {"id": 2, "name": "产品卖点", "content": "补水效果特别好，用完皮肤水嫩嫩的"},
      {"id": 3, "name": "价格促销", "content": "原价199，今天直播间只要99，数量有限，赶紧下单！"}
    ],
    "version_count": 3,
    "style": "带货直播口语",
    "shuffle": false
  }'
```

**参数说明：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `shots` | array | 是 | Step 2 返回的分镜列表 |
| `version_count` | int | 否 | 裂变版本数，1-5，默认 3 |
| `style` | string | 否 | 风格提示，默认 "带货直播口语" |
| `shuffle` | bool | 否 | 是否随机打乱，默认 false |

**返回示例：**
```json
{
  "retcode": 200,
  "succ": true,
  "data": {
    "version_count": 3,
    "style": "带货直播口语",
    "matrix": [
      {
        "id": 1,
        "name": "开场招呼",
        "versions": [
          "嗨各位宝子们，今天必须给你们安利这款绝绝子面膜",
          "家人们看过来！今天的主角是一款我自用超久的面膜",
          "亲爱的们晚上好，今天要给姐妹们种草一款宝藏面膜"
        ]
      },
      {
        "id": 2,
        "name": "产品卖点",
        "versions": [
          "主打一个补水，敷完脸蛋嫩得能掐出水来",
          "补水力度拉满，用完皮肤像喝饱了水一样弹润",
          "深层补水不是说说而已，敷一次你就知道什么叫水光肌"
        ]
      },
      {
        "id": 3,
        "name": "价格促销",
        "versions": [
          "平时卖199的，今晚直播间直接砍到99，手慢无！",
          "原价199，今天不要这个价！直播间专属99，冲就完了",
          "199的东西99拿走，就今天这一场，错过等半年！"
        ]
      }
    ]
  }
}
```

向用户展示裂变矩阵结果。

---

### Step 3 备选 — 整段裂变（不拆分镜）

若用户不需要分镜，可以直接整段裂变：

```bash
curl -s -X POST http://localhost:7073/api/copywriting/whole-fission \
  -H "Content-Type: application/json" \
  -d '{
    "raw_text": "大家好，今天给大家推荐一款超好用的面膜...",
    "version_count": 3,
    "style": "带货直播口语"
  }'
```

**返回示例：**
```json
{
  "retcode": 200,
  "succ": true,
  "data": {
    "title": "面膜推荐",
    "version_count": 3,
    "style": "带货直播口语",
    "versions": [
      {"version": 1, "content": "宝子们好呀！今天必须给你们安利一款我的心头好面膜..."},
      {"version": 2, "content": "家人们晚上好！今天的福利品大家一定要抢..."},
      {"version": 3, "content": "各位姐妹看过来！今天给大家带来的是一款回购了无数次的面膜..."}
    ]
  }
}
```

---

### Step 4 — 保存结果到本地

将裂变结果保存为 JSON 文件，方便后续使用（如导入语音合成）。

**保存路径规则：**
- 如果用户指定了保存路径，使用用户指定的路径
- 如果未指定，默认保存到当前工作目录，文件名格式：`fission_结果标题_时间戳.json`

**保存的 JSON 格式（分镜裂变结果）：**
```json
{
  "title": "超好用面膜推荐",
  "version_count": 3,
  "style": "带货直播口语",
  "shots": [
    {"id": 1, "name": "开场招呼", "content": "大家好，今天给大家推荐一款超好用的面膜"},
    {"id": 2, "name": "产品卖点", "content": "补水效果特别好，用完皮肤水嫩嫩的"},
    {"id": 3, "name": "价格促销", "content": "原价199，今天直播间只要99，数量有限，赶紧下单！"}
  ],
  "matrix": [
    {
      "id": 1,
      "name": "开场招呼",
      "versions": [
        {"version": 1, "content": "..."},
        {"version": 2, "content": "..."},
        {"version": 3, "content": "..."}
      ]
    }
  ]
}
```

使用文件系统操作将 JSON 字符串写入文件，然后告知用户保存路径。

---

### Step 5 — 告知用户结果

告知用户：
- 原始文案标题
- 分镜数量
- 裂变版本数
- 保存的 JSON 文件路径
- 提示用户可以将此 JSON 文件导入到语音合成 skill 中继续使用

---

## 接口一览

所有接口均通过容剪 Go 服务（localhost:7073）代理调用：

| 方法 | 路径 | 功能 |
|------|------|------|
| POST | `/api/copywriting/split` | 将长文案拆分为分镜表 |
| POST | `/api/copywriting/name-shots` | 为分镜一键生成名称 |
| POST | `/api/copywriting/split-and-name` | 拆分并命名（推荐） |
| POST | `/api/copywriting/fission` | 分镜维度裂变 |
| POST | `/api/copywriting/whole-fission` | 整段裂变 |
| GET  | `/api/copywriting/health` | 检查文案裂变服务状态 |

---

## 错误处理

> **重要：不要向用户展示原始的后端错误信息（如 IP 地址、端口号、内部服务名、日志路径等），应转换为用户友好的提示。**

| 错误情况 | 处理方式 |
|----------|----------|
| Go 服务连接失败 | 提示用户：**"容剪服务未启动，请先打开容剪客户端"** |
| 文案裂变后端不可用 | 提示用户：**"文案裂变服务暂时不可用，请稍后重试"** |
| `succ` 为 false | 仅展示 `retdesc` 中面向用户的描述（如"文案内容为空"），**不要展示包含 IP、端口、服务名等技术细节的原始错误** |
| 文案为空 | 提示用户提供需要裂变的文案内容 |
| 裂变版本数超范围 | 分镜裂变：1-5；整段裂变：1-10 |
| AI 生成超时 | 裂变调用可能需要 30-60 秒（AI 生成），耐心等待 |
| 其他未知错误 | 提示用户：**"操作失败，请重试。如持续出现问题请联系客服"**，不要暴露内部错误堆栈 |
