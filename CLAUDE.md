# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 仓库概况

jev-chat 是一个「先判断、再回复」的聊天决策助手：读取屏幕上的对话 → Jev 判断模型分析意图/风险/最佳回应 → 生成候选回复并排序 → 一键填入输入框。**发送永远由用户手动完成，代码中不存在任何点击发送按钮或触发 IME 发送的路径。**

本仓库包含两个**互相独立、不共享代码**的 Android 版本，各自是一个独立的 Gradle 构建，必须在各自目录下运行 `gradlew`：

| 目录 | 产品 | 说明 |
|---|---|---|
| `cn/` | Jev 聊天助手（国内版，v1.7） | 单 App 模块（`com.jev.probe`），悬浮窗覆盖 QQ、飞书、X、WhatsApp；中文场景，带 ML Kit 本地 OCR 兜底 |
| `global/` | Jev for WhatsApp（海外版，v0.1.0） | 多模块工程（`core` / `a11y` / `assistant` / `probe`），仅适配 WhatsApp 英文聊天 |

两个版本的代码组织哲学不同：国内版按层分包（`capture` / `jev` / `core/kb`），采集适配器一 App 一个（`ChatAppAdapter.kt` 按前台包名查表分发）；海外版严格分层为纯 JVM 引擎 + Android 壳——`core` 模块**不含任何 Android 类型和第三方库**，整条分析流程可用脚本化的 `FakeGateway` 在 JVM 上离线单测，`a11y` 负责采集无障碍节点，`assistant` 是无障碍服务 + 悬浮 UI，`probe` 是开发用的录屏工具（无 `INTERNET` 权限）。详见 `global/ARCHITECTURE.md`。

`cn/CLAUDE.md` 是国内版的详细工作手册（硬约束、实测结论、报告格式），在 `cn/` 内工作时必须一并阅读。

## 常用命令

国内版（在 `cn/` 目录下）：

```bash
./gradlew assembleDebug      # app/build/outputs/apk/debug/app-debug.apk
./gradlew assembleRelease    # 需要仓库外的签名配置；用 JEV_KEYSTORE_PROPS 指定路径
./gradlew test               # JVM 单测，含 whatsapp/ 录屏数据驱动的规则测试
```

海外版（在 `global/` 目录下）：

```bash
./gradlew test               # 离线单测，无 key 无网络；模型调用走 FakeGateway
./gradlew :probe:assembleDebug   # 录屏工具 APK
./gradlew assembleDebug      # assistant/build/outputs/apk/debug/assistant-debug.apk
./gradlew assembleRelease    # 需要 JEV_OVERSEAS_KEYSTORE_PROPS 指向仓库外的签名配置
./gradlew -q :core:liveEval -PevalArgs="analysis"   # 付费实测评测，需 OPENROUTER_API_KEY
```

单个测试用 `./gradlew test --tests "类名"`。海外版 golden 文件用 `JEV_UPDATE_GOLDEN=1` 重新生成（仅限有意变更后）；`core/src/test/resources/approved/` 下的 matrix.tsv / apology.tsv 是人工批准的基准，只能因约定好的行为变更而改。完整测试说明在 `global/docs/TESTING.md`。

技术栈：Kotlin 1.9.24、AGP 8.7.3、JDK 17、minSdk 30 / targetSdk 35，国内版 UI 用传统 View + XML（不用 Compose）、仅 arm64-v8a。CI（`.github/workflows/build-debug.yml`）在 push 到 main 时构建国内版 debug APK。

## 关键架构要点

**Jev 判断模型**（两版共用同一思路）：TypeSafe 的判断模型，只回答选择题/打分/是非，不生成文字。走 OpenRouter `POST https://openrouter.ai/api/alpha/decisions`，model `typesafe/jev-1.13`，body `{model, state, questions}`，答案为概率值。题目 instructions 和 criteria 用英文写，聊天内容保留原文。回复文本由普通 chat 模型生成，再逐条经 Jev 检查（硬违规拦截 / 确认项 / G 完成度 + E 投递分，S = 0.7G + 0.3E）后排序。

**国内版采集管道**：无障碍服务读聊天 App 控件树；适配器契约是 `extract` 返回 `null` = 不在聊天窗，返回空列表 = 在聊天窗但树里没正文（只有这种情况触发 `takeScreenshot()` + ML Kit OCR 兜底）。各目标 App 的控件树实测结论（QQ 不混淆、飞书正文自绘必须 OCR、X 信息全在 content-desc 等）记录在 `cn/CLAUDE.md`，改采集逻辑前先查。

**海外版一轮流程**：点气泡 → `assistant/service` + `a11y` + `core/whatsapp` + `core/stitch` 上滚读屏拼消息 → `Analysis`（2 个 Jev 请求）→ `Goal` → `Drafting`（1 个 chat 请求）→ `CandidateCheck`（每回复 Jev 检查）→ `Fill` 写输入框。`AssistantSession` 是状态机；`RunBudget` 限每轮 17 次模型请求、90 秒等待，失败重试一次（认证错误不重试）。

## 硬性红线

- **绝不自动发送消息**：不点击任何 App 的发送按钮、不触发 IME enter。这条在两版代码中都是结构性约束，任何改动不得引入绕过路径。
- **不 hook、不 Xposed、不改目标 App、不读其数据库**：只用系统无障碍服务与截屏；只读目标 App 正常开放给无障碍服务的内容。
- **不碰钱**：不触碰转账、红包、收款码相关界面元素。
- **密钥不落盘、不进日志、不进 git**：OpenRouter key 从环境变量 `OPENROUTER_API_KEY` 或 App 内加密存储读取，任何文件不得出现 `sk-or-` 开头字符串。
- 不要 `git commit` / `git push`，除非用户明确要求。
- 国内版曾踩过的坑：Android 构建工具在 Windows 上不接受中文路径（如遇构建问题考虑路径 ASCII 化）；Kotlin 字符串模板 `$x` 后紧跟中文标点会被解析成标识符，一律写 `${x}`。

## 仓库协作约定

- `global/` 的维护者是 @smgonthebeat（见 `.github/CODEOWNERS`），改动海外版时注意归属。
- 国内版任务交付报告写在 `cn/_reports/<任务名>_report.md`（真实命令输出 + 自验缺口，见 `cn/CLAUDE.md` 报告格式）。
- 根目录没有 README；对外 README 在 `.github/README.md`（多语言副本同目录）。
