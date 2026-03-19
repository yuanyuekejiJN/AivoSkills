---
name: aivo-fast-filter
description: |
  调用容剪(AIVO)极速过滤功能，自动检测并删除视频中的静音/停顿段落。
  触发关键词（匹配任意一个）：
  - "极速过滤"、"去掉静音"、"删除停顿"、"快速剪辑静音段"
  - "去掉视频空白"、"删掉没声音的部分"、"自动剪辑静音"
  - "去除沉默"、"过滤静音"、"静音检测"、"删除空白段"
  容剪客户端必须已启动（端口 7073）。
metadata:
  openclaw:
    emoji: "✂️"
    requires:
      bins:
        - curl
    os:
      - darwin
      - linux
      - win32
---

# 极速过滤 — 自动删除静音段

> **严格约束（必须遵守）：**
> 1. **只使用本文档中明确列出的 API 路径**，绝对不要猜测或构造文档中没有的 URL
> 2. 基础地址固定为 `http://localhost:7073`
> 3. `/api/health` 返回 `{"status":"ok"}` 即代表服务可用，不要再告诉用户"服务不可用"或"功能暂不支持"
> 4. Windows 路径在 JSON 中必须用正斜杠 `/`（如 `D:/videos/a.mp4`）

## 接口一览（仅限使用这些路径）

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/api/health` | 全局健康检查 |
| GET | `/api/fast-filter/health` | 极速过滤模块健康检查 |
| POST | `/api/fast-filter/analyze` | 单文件静音分析 |
| POST | `/api/fast-filter/batch-analyze` | 多文件批量分析 |
| POST | `/api/fast-filter/export` | 导出（删除静音后的视频） |
| GET | `/api/fast-filter/export/progress?taskId=` | 轮询导出进度 |

---

## 第一步：健康检查

```bash
curl -s --connect-timeout 3 http://localhost:7073/api/fast-filter/health
```

返回 `{"status":"ok","message":"极速过滤功能正常"}` 即可继续。也可先调用 `/api/health` 验证整体服务可用。

## 第二步：收集参数

- `filePath`：视频文件路径（单文件）或多个文件路径列表
- `outputPath`：输出文件路径
- `silenceThreshold`：静音阈值（dB，默认 -40，越低越严格）
- `minSilenceDuration`：最短静音时长（秒，默认 0.5）

## 第三步：分析静音段

**单文件：**
```bash
curl -s -X POST http://localhost:7073/api/fast-filter/analyze \
  -H "Content-Type: application/json" \
  -d '{
    "filePath": "<视频路径>",
    "silenceThreshold": -40,
    "minSilenceDuration": 0.5
  }'
```

**多文件批量：**
```bash
curl -s -X POST http://localhost:7073/api/fast-filter/batch-analyze \
  -H "Content-Type: application/json" \
  -d '{
    "files": ["<路径1>", "<路径2>"],
    "silenceThreshold": -40,
    "minSilenceDuration": 0.5
  }'
```

返回检测到的静音段列表。向用户展示：检测到 N 处静音，总时长 X 秒，删除后视频缩短约 Y%。

## 第四步：确认并导出

用户确认后执行导出：

```bash
curl -s -X POST http://localhost:7073/api/fast-filter/export \
  -H "Content-Type: application/json" \
  -d '{
    "filePath": "<源视频>",
    "outputPath": "<输出路径>",
    "segments": <保留片段数组>
  }'
```

## 第五步：轮询进度

```bash
curl -s "http://localhost:7073/api/fast-filter/export/progress?taskId=<taskId>"
```

---

## 错误处理

> **重要：不要向用户展示原始的后端错误信息（如 IP 地址、端口号、内部服务名、日志路径等），应转换为用户友好的提示。**

| 错误情况 | 处理方式 |
|----------|----------|
| 连接失败 | 提示用户：**"容剪服务未启动，请先打开容剪客户端"** |
| 视频文件不存在 | 提示用户检查文件路径是否正确 |
| `status: "failed"` | 仅向用户展示面向用户的错误描述，**不要展示包含 IP、端口、文件路径等技术细节的原始错误** |
| 导出失败 | 提示用户：**"导出失败，请检查输出目录是否有写入权限"** |
| 其他未知错误 | 提示用户：**"操作失败，请重试"**，不要暴露内部错误堆栈 |
