---
title: AIVO Skills — 容剪技能集合
description: OpenClaw AI Agent 技能集合，调用容剪 AIVO 实现视频剪辑、TTS配音、视频下载等能力
---

<p align="center">
  <img src="assets/icon.png" width="120" alt="AIVO Claw Logo" />
</p>

<h1 align="center">AIVO Skills — 容剪技能集合</h1>

<p align="center">
  通过 OpenClaw AI Agent 调用容剪 AIVO 的完整视频处理能力。
</p>

<p align="center">
  <a href="https://aivoclaw.com"><img src="https://img.shields.io/badge/官网-aivoclaw.com-2563eb?style=flat-square" alt="Official Website" /></a>
  <a href="https://github.com/yuanyuekejiJN/AivoSkills/releases"><img src="https://img.shields.io/github/v/release/yuanyuekejiJN/AivoSkills?style=flat-square&color=2563eb" alt="Latest Release" /></a>
  <a href="https://github.com/yuanyuekejiJN/AivoSkills/blob/main/LICENSE"><img src="https://img.shields.io/github/license/yuanyuekejiJN/AivoSkills?style=flat-square" alt="License" /></a>
</p>

<p align="center">
  <a href="https://aivoclaw.com">🌍 官网</a> | <a href="./README_en.md">English</a> | 中文
</p>

---

## 版本说明

当前版本：**v1.0.0** (2026-03-19)

所有技能版本统一管理，详见 [registry.json](./registry.json)。

---

## 快速安装

### 方式一：下载 Release（推荐）

前往 [Releases 页面](https://github.com/yuanyuekejiJN/AivoSkills/releases) 下载最新版本的技能包：

```bash
# 下载 all-in-one 包（含全部技能）
curl -L -o aivo-skills.zip https://github.com/yuanyuekejiJN/AivoSkills/releases/download/v1.0.0/aivo-skills-all-v1.0.0.zip

# 解压到 OpenClaw 技能目录
unzip aivo-skills.zip -d ~/.openclaw/skills/
```

### 方式二：单独下载某个技能

```bash
# 下载单个技能（例如 aivo-tts）
curl -L -o aivo-tts.zip https://github.com/yuanyuekejiJN/AivoSkills/releases/download/v1.0.0/aivo-tts-1.0.0.zip

# 解压到技能目录
unzip aivo-tts.zip -d ~/.openclaw/skills/
```

### 方式三：Git Clone（开发者）

```bash
# 克隆整个仓库
git clone https://github.com/yuanyuekejiJN/AivoSkills.git

# 复制单个技能
cp -r aivo-tts ~/.openclaw/skills/

# 或复制全部技能
cp -r aivo-*/ ~/.openclaw/skills/
```

---

## 包含技能

| 技能 | 说明 | 分类 | 版本 |
|------|------|------|------|
| [aivo-luban](aivo-luban/) | 鲁班模式 — ZGL批量视频剪辑 | editing | 1.0.0 |
| [aivo-nezha](aivo-nezha/) | 哪吒模式 — SWK多轨道矩阵剪辑 | editing | 1.0.0 |
| [aivo-fast-filter](aivo-fast-filter/) | 极速过滤 — 自动删除静音段 | editing | 1.0.0 |
| [aivo-split-scene](aivo-split-scene/) | 按场景拆解 — 检测场景切换点 | editing | 1.0.0 |
| [aivo-tts](aivo-tts/) | AI配音 — 语音合成多音色 | audio | 1.0.0 |
| [aivo-video-download](aivo-video-download/) | 视频下载 — 多平台解析提取 | download | 1.0.0 |
| [aivo-download-split](aivo-download-split/) | 下载+场景拆解 — 组合技能 | download | 1.0.0 |
| [aivo-download-extract](aivo-download-extract/) | 下载+文案提取 — ASR语音识别 | download | 1.0.0 |
| [aivo-copywriting-fission](aivo-copywriting-fission/) | 文案裂变 — AI多版本改写 | copywriting | 1.0.0 |

---

## 前置要求

- **容剪客户端**：需要容剪客户端已启动（端口 7073）
- **OpenClaw**：已安装 OpenClaw AI Agent

---

## 目录结构

```
aivo-skills/
├── registry.json              # 技能注册表索引
├── aivo-luban/
│   ├── manifest.json          # 技能元数据
│   └── SKILL.md              # AI 调用指南
├── aivo-tts/
│   ├── manifest.json
│   └── SKILL.md
└── ...
```

---

## 更新日志

### v1.0.0 (2026-03-19)

- 初始版本发布
- 包含 9 个 AIVO 技能：
  - 视频剪辑：鲁班模式、哪吒模式、极速过滤、场景拆解
  - 音频：AI配音
  - 下载：视频下载、下载+场景拆解、下载+文案提取
  - 文案：文案裂变

---

## 技术交流群

扫码加入 AivoSkills 技术交流群，获取最新动态、反馈问题、交流使用心得：

<p align="center">
  <img src="assets/qr.png" width="200" alt="技术交流群二维码" />
</p>

---

## 许可证

MIT License

---

<p align="center">
  <sub>由 <a href="https://aivoclaw.com">元岳科技</a> 打造</sub>
</p>
