# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目当前状态

QuestLearn 是一款微信小程序，**目前仅有 PRD 和方案设计文档（见 `docs/`），尚无任何应用代码**。当前优先实现 P0：纯文本链路端到端跑通 AI 编排（文本输入 → 生成题目 → 答题 → 复盘报告）。

- `docs/PRD.md` —— 需求、用户故事、验收标准、非功能需求
- `docs/方案设计文档.md` —— 技术方案 v1.2，**架构/选型/数据模型/Prompt 工程的权威来源，动手前先查**

## 架构概览

用户输入知识（文本或网页链接）→ AI 生成单选/多选/判断题 → 逐题闯关即时反馈 → 通关生成复盘报告 → 可分享微信好友。

monorepo（待创建）：

```
docs/        PRD + 方案设计
frontend/    Taro 4.x 小程序（首页主包；答题/结果/历史分包）
backend/     FastAPI（app/{api,schemas,services,prompts,core} + tests/）
```

分层链路：Taro 小程序 ──HTTPS(已备案域名)──> FastAPI ──LangChain──> DeepSeek V4；P1 网页抓取走后端 httpx+trafilatura（**前端不能直接抓任意 URL，微信白名单制，必须后端抓**）。

后端 P0-P2 **无状态不落库**：题目集不持久化，直接返前端；挑战历史走前端微信 storage（M7）。

## 技术栈（硬约束，勿擅自更换）

前端：
- Taro 4.x + React 18（**锁死 `react@^18`**）+ TypeScript strict
- 状态：Zustand（仅挑战进行中持题目集+作答，简单页用页面内状态）
- 请求：基于 `Taro.request` 统一封装
- UI：优先自定义轻量组件，MVP 不引入过重 UI 框架

后端：
- FastAPI + Python 3.11+ + Pydantic v2
- LangChain **仅** `ChatPromptTemplate` 模板化 + `with_structured_output` 输出约束；**不引入** Agent/Chain/Memory/ReAct
- DeepSeek V4（OpenAI SDK 兼容，`base_url=https://api.deepseek.com`）：首选 `deepseek-v4-flash`（non-thinking，保 15s 出题），次选 `deepseek-v4-pro`（复杂知识/复盘报告）
- 配置 pydantic-settings、日志 structlog、测试 pytest
- 辅助库：tenacity（重试）、httpx + trafilatura（P1 抓取正文）

## 关键工程纪律

1. **React 18 锁死**：`package.json` 固定 `react@^18`，引入第三方库前查 peerDeps（taro-ui 3.x 强制 React 19 类型，与 Taro 4.x 冲突致构建失败）
2. **立项即分包**：主包只放首页，答题/报告/历史放分包 + `mini-split-chunks-plugin`；主包 <2MB 是硬约束
3. **答题页 CompileMode + Skyline**：`experimental.compileMode: true` + Skyline worklet 保 <200ms 交互，不可用页面降级 WebView
4. **TS strict + props 引用稳定**：基础组件 props 用 `useMemo`/`useRef`，避免内联对象/数组触发无谓 setData
5. **题型统一**：`answer` 统一 `list[str]`，判定逻辑 `set(user_answer) == set(question.answer)`，不分三套代码
6. **结构化输出**：`with_structured_output(QuestionSet, method="function_calling")`（DeepSeek 官方支持 Tool Calls，无 `response_format json_schema`），返回校验后 Pydantic 对象，勿手写 `json.loads`+`model_validate`
7. **后处理校验（不靠模型自觉，代码兜底）**：答案分布/选项去重/解析泄露正则扫描（"正确答案是"等）/判断题必须 2 选项/题干不照搬原文，失败即重试
8. **重试分离**：修复型（校验失败，max 1 次改 prompt）vs 退避型（限流/超时，max 3 次指数退避），用 tenacity 精细控制，勿混用
9. **SSRF 防护（P1 抓取必做）**：协议白名单、内网 IP 段过滤、DNS 解析后检查（不能只看 hostname）、重定向逐跳检查、超时、Content-Type 校验
10. **微信 storage 限制**：单 key 1MB / 总 10MB，按挑战 ID 分 key 存历史，摘要+详情分离，留最近 50 条，卸载自动清理

## 题目数据模型（见方案设计文档 4.2）

- `Option.label: Literal["A","B","C","D","T","F"]`（T/F 判断题），防模型输出大小写/格式不一致
- `Question.answer: list[str]`（单选/判断长度 1，多选≥2），跨字段校验用 `field_validator` 读 `info.data`
- schema 用 `QuestionSet.model_json_schema()` dump 进 prompt，比手写描述可靠

## 开发命令

代码尚待创建。各子目录搭建后的标准命令：

前端（`frontend/`）：
- 开发：`npm run dev:weapp`（Taro 编译，用微信开发者工具打开 `dist/`）
- 构建：`npm run build:weapp`
- 类型检查：`tsc --noEmit`

后端（`backend/`）：
- 装依赖：`pip install -e ".[dev]"`（或 `uv sync`）
- 开发：`uvicorn app.main:app --reload`
- 测试：`pytest` / 单个测试：`pytest tests/test_xxx.py::test_func -v`

## 分阶段策略与 P0 步骤

- **P0（当前）** M1 文本输入 + M3 生成题目 + M4 答题 + M5 结果报告
- P1 M2 网页抓取（`/api/extract` + SSRF 防护 + 摘要确认页）
- P2 M6 微信分享 + M7 挑战历史（本地 storage）
- P3 S1 知识库 / S2 难度 / S3 题目数量 / S4 错题本
- P4 N1 RAG / N2 AI 生图 / N3 VIP / N4 多人对战

P0 后端步骤（每步验证标准见方案设计文档 7.2）：①骨架+DeepSeek 连通 → ②generate 接口（schema+prompt+`with_structured_output`+后处理校验+tenacity 修复重试）→ ③report 接口 → ④pytest 兜底 → ⑤前端 Taro 骨架+分包+首页 → ⑥答题页 CompileMode → ⑦结果页端到端集成（真实 DeepSeek key 跑通）。

<!-- superpowers-zh:begin (do not edit between these markers) -->
# Superpowers-ZH 中文增强版

本项目已安装 superpowers-zh 技能框架（20 个 skills）。

## 核心规则

1. **收到任务时，先检查是否有匹配的 skill** — 哪怕只有 1% 的可能性也要检查
2. **设计先于编码** — 收到功能需求时，先用 brainstorming skill 做需求分析
3. **测试先于实现** — 写代码前先写测试（TDD）
4. **验证先于完成** — 声称完成前必须运行验证命令

## 可用 Skills

Skills 位于 `.claude/skills/` 目录，每个 skill 有独立的 `SKILL.md` 文件。

- **brainstorming**: 在任何创造性工作之前必须使用此技能——创建功能、构建组件、添加功能或修改行为。在实现之前先探索用户意图、需求和设计。
- **chinese-code-review**: 中文 review 沟通参考——话术模板、分级标注（必须修复/建议修改/仅供参考）、国内团队常见反模式应对。仅在用户显式 /chinese-code-review 时调用，不要根据上下文自动触发。
- **chinese-commit-conventions**: 中文 commit 与 changelog 配置参考——Conventional Commits 中文适配、commitlint/husky/commitizen 中文模板、conventional-changelog 中文配置。仅在用户显式 /chinese-commit-conventions 时调用，不要根据上下文自动触发。
- **chinese-documentation**: 中文文档排版参考——中英文空格、全半角标点、术语保留、链接格式、中文文案排版指北约定。仅在用户显式 /chinese-documentation 时调用，不要根据上下文自动触发。
- **chinese-git-workflow**: 国内 Git 平台配置参考——Gitee、Coding.net、极狐 GitLab、CNB 的 SSH/HTTPS/凭据/CI 接入差异与镜像同步配置。仅在用户显式 /chinese-git-workflow 时调用，不要根据上下文自动触发。
- **dispatching-parallel-agents**: 当面对 2 个以上可以独立进行、无共享状态或顺序依赖的任务时使用
- **executing-plans**: 当你有一份书面实现计划需要在单独的会话中执行，并设有审查检查点时使用
- **finishing-a-development-branch**: 当实现完成、所有测试通过、需要决定如何集成工作时使用——通过提供合并、PR 或清理等结构化选项来引导开发工作的收尾
- **mcp-builder**: MCP 服务器构建方法论 — 系统化构建生产级 MCP 工具，让 AI 助手连接外部能力
- **receiving-code-review**: 收到代码审查反馈后、实施建议之前使用，尤其当反馈不明确或技术上有疑问时——需要技术严谨性和验证，而非敷衍附和或盲目执行
- **requesting-code-review**: 完成任务、实现重要功能或合并前使用，用于验证工作成果是否符合要求
- **subagent-driven-development**: 当在当前会话中执行包含独立任务的实现计划时使用
- **systematic-debugging**: 遇到任何 bug、测试失败或异常行为时使用，在提出修复方案之前执行
- **test-driven-development**: 在实现任何功能或修复 bug 时使用，在编写实现代码之前
- **using-git-worktrees**: 当需要开始与当前工作区隔离的功能开发，或在执行实现计划之前使用——通过原生工具或 git worktree 回退机制确保隔离工作区存在
- **using-superpowers**: 在开始任何对话时使用——确立如何查找和使用技能，要求在任何响应（包括澄清性问题）之前调用 Skill 工具
- **verification-before-completion**: 在宣称工作完成、已修复或测试通过之前使用，在提交或创建 PR 之前——必须运行验证命令并确认输出后才能声称成功；始终用证据支撑断言
- **workflow-runner**: 在 Claude Code / OpenClaw / Cursor 中直接运行 agency-orchestrator YAML 工作流——无需 API key，使用当前会话的 LLM 作为执行引擎。当用户提供 .yaml 工作流文件或要求多角色协作完成任务时触发。
- **writing-plans**: 当你有规格说明或需求用于多步骤任务时使用，在动手写代码之前
- **writing-skills**: 当创建新技能、编辑现有技能或在部署前验证技能是否有效时使用

## 如何使用

当任务匹配某个 skill 时，使用 `Skill` 工具加载对应 skill 并严格遵循其流程。绝不要用 Read 工具读取 SKILL.md 文件。

如果你认为哪怕只有 1% 的可能性某个 skill 适用于你正在做的事情，你必须调用该 skill 检查。
<!-- superpowers-zh:end -->
