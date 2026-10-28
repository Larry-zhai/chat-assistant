# 仓库分析：jev-chat-jarvis

## 概述

jev-chat-jarvis 是一个「先判断、再回复」的聊天决策助手。它读取屏幕上的对话内容，由 Jev 判断模型分析对方意图、风险等级和最佳回应方式，然后生成候选回复并排序，最后一键填入输入框——**发送永远由用户手动完成，代码中不存在任何自动发送的路径**。

仓库的核心卖点是「判断先于生成」：大多数 AI 聊天工具直接起草回复，而 jev-chat 先用判断模型算出对方真正想要什么、此刻的风险有多大，再让普通聊天模型基于这个判断起草回复，每条候选回复还要再经过 Jev 逐项检查（硬违规拦截、确认项、完成度与投递分，S = 0.7G + 0.3E）后才排序展示。

仓库内包含两个**互相独立、不共享代码**的 Android 版本：国内版「Jev 聊天助手」（`cn/`，v1.7）和海外版「Jev for WhatsApp」（`global/`，v0.1.0）。此外还有同家族的 Windows、macOS、iOS 产品，但它们各自独立成仓，不在本仓库内。

## 架构

两个版本各自是一个独立的 Gradle 构建，必须在各自目录下运行 `gradlew`，且代码组织哲学不同：

- **`cn/`（国内版，单 App 模块 `com.jev.probe`）**：按层分包——`capture`（采集）/ `jev`（模型客户端）/ `core/kb`（知识库）/ `overlay`（悬浮窗）。采集适配器一个 App 一个，`ChatAppAdapter.kt` 按前台包名查表分发。适配器契约：`extract` 返回 `null` 表示不在聊天窗；返回空列表表示在聊天窗但控件树里没正文——只有后者才触发 `takeScreenshot()` + ML Kit OCR 兜底。
- **`global/`（海外版，多模块工程 `core` / `a11y` / `assistant` / `probe`）**：严格分层为纯 JVM 引擎 + Android 壳。`core` 模块不含任何 Android 类型和第三方库，整条分析流程可用脚本化的 `FakeGateway` 在 JVM 上离线单测；`a11y` 负责采集无障碍节点；`assistant` 是无障碍服务 + 悬浮 UI；`probe` 是开发用的录屏工具（无 `INTERNET` 权限）。

**Jev 判断模型**（两版共用同一思路）：TypeSafe 的判断模型，只回答选择题/打分/是非，不生成文字。走 OpenRouter `POST https://openrouter.ai/api/alpha/decisions`，model `typesafe/jev-1.13`，body `{model, state, questions}`，答案为概率值。题目的 instructions 和 criteria 用英文写（模型主训练语言是英文），聊天内容保留原文。除 OpenRouter 外，国内版还支持博查 Jev、TypeSafe 直连、Vercel AI Gateway、OpenCode Zen、DeepSeek 官方、通义兼容等预设，判断/回复/视觉三路接口可分别配置地址、密钥、模型。

## 核心组件

- **`cn/app/src/main/java/com/jev/probe/capture/`**：无障碍采集层。`ChatCaptureService` 是核心服务，`ChatAppAdapter` 按包名分发到各 App 适配器（QQ、飞书、X、WhatsApp）。各目标 App 的控件树实测结论记录在 `cn/CLAUDE.md`（如：QQ 节点不混淆、飞书正文自绘必须 OCR、X 信息全在 content-desc）。
- **`cn/.../capture/ocr/`**：OCR 兜底。`ScreenCapture` 限频 ≥1s + 失败退避（1s→2s→…→30s 封顶）；`MlKitOcr` 用 bundled 中文模型离线识别，不联网不上传。
- **`cn/.../jev/`**：模型客户端三路——`JudgeClient`（判断）、`ReplyClient`（回复）、`VisionClient`（视觉），共用 `HttpJson`。
- **`cn/.../core/kb/`**：本地知识库。笔记 + 联系人档案存于 `filesDir/kb` 的 JSON 文件，`KbStore` 单锁 + 原子写；`ContextBuilder` 只做 alwaysOn 全带 + 标签/标题匹配，历史默认关闭。
- **`cn/.../overlay/`**：悬浮窗 UI（常驻小卡片，可拖动，候选回复可「换一组」）。
- **`global/core/`**：纯 JVM 分析引擎——`whatsapp` 采集规则、`stitch` 上滚读屏拼消息、`Analysis`（2 个 Jev 请求）、`Goal`、`Drafting`（1 个 chat 请求）、`CandidateCheck`（每回复 Jev 检查）。五个场景矩阵（Work / Romance / Friends / Family / General）经人工批准为基准测试。
- **`global/a11y/` + `global/assistant/`**：Android 壳，`AssistantSession` 是状态机；`RunBudget` 限每轮 17 次模型请求、90 秒等待。
- **`global/probe/`**：开发用录屏工具，产出 `fixtures/screendumps/` 下的控件树数据驱动规则测试。
- **`.github/`**：对外 README（英/中/越三语）、CI（push 到 main 构建国内版 debug APK）、CODEOWNERS。
- **`cn/apk/`**：随仓分发的 release APK。

## 技术栈

- **语言与构建**：Kotlin 1.9.24、AGP 8.7.3、JDK 17、Gradle（两个独立 wrapper）
- **Android**：minSdk 30 / targetSdk 35；国内版 UI 用传统 View + XML（不用 Compose），仅 arm64-v8a；海外版 release 不限 CPU
- **本地 OCR**：ML Kit `com.google.mlkit:text-recognition-chinese:16.0.1`（bundled）
- **模型接入**：OpenRouter TypeSafe decisions API（判断）+ 普通 chat 模型（回复）；密钥从环境变量 `OPENROUTER_API_KEY` 或 App 内加密存储读取（Android Keystore）
- **测试**：JVM 单测；海外版用 `FakeGateway` 离线跑全流程，golden 文件人工批准；国内版用 WhatsApp 录屏数据驱动规则测试
- **CI**：GitHub Actions（`.github/workflows/build-debug.yml`）

## 数据流

一轮典型流程（海外版为例）：点气泡 → `assistant/service` + `a11y` + `core/whatsapp` + `core/stitch` 上滚读屏拼消息 → `Analysis`（Jev 判断意图/风险/最佳动作）→ `Goal`（定目标与姿态）→ `Drafting`（chat 模型起草）→ `CandidateCheck`（每条候选经 Jev 逐项检查打分）→ `Fill` 写输入框 → **人手动发送**。

国内版流程类似：无障碍服务读控件树 → 适配器抽取消息（树里没正文时截屏 + 本地 OCR）→ 命中知识库笔记/历史时一并带给模型 → Jev 判断 → 生成 3 条候选 → Jev 排序 → 悬浮窗展示 → 一键填入。

两版共同的结构性红线：不 hook、不 Xposed、不改目标 App、不读其数据库；不碰转账/红包/收款码；密钥不落盘不进 git；绝不自动发送。

## 团队与归属

- **Finderchangchang**（同一邮箱下还有 Vikicc、柳伟杰 等署名）：项目主要作者，主导国内版全部迭代与仓库运营。
- **@smgonthebeat**：海外版 `global/` 的维护者（见 `.github/CODEOWNERS`），提供了 WhatsApp 采集规则与整个多模块海外版工程（PR #73）。
- **社区贡献者**：通过 PR 参与的包括 xuancanhit99（流式请求修复）、thanvinhtrongsv（越南语 README）、szepeviktor（英语 README）、keyuchen21（文档修正）、kuishou68、ahxoeh（OCR 主线程与重复分析修复）、adudu2017（会话绑定分析修复）等。
