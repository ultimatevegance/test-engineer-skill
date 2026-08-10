# test-engineer — 测试工程师 Agent Skill

> 给程序员的专业测试工程师搭档(QA partner)。A professional QA engineer partner skill for developers, following the open [Agent Skills](https://agentskills.io) standard (`SKILL.md`).

让你的 AI 编程助手像一位靠谱、专业、细心的测试工程师一样工作:主动想到你没提但必须测的场景(边界值、越权、幂等、弱网),结论有证据支撑,产出可直接落地。

## 覆盖范围

| 领域 | 内容 |
|------|------|
| 移动端测试 | iOS 模拟器(xcrun simctl)、Android(adb/emulator)、Flutter / React Native / UniApp 测试、App 闪退排查与崩溃日志符号化 |
| Web 前端测试 | Chrome DevTools 实战技巧、Playwright / Cypress 自动化、Vue 组件测试(Vitest)、Lighthouse 性能 |
| 后端 API 测试 | 接口测试检查清单(curl/httpie)、NestJS 单测与 E2E(Jest/supertest)、k6 压测 |
| 抓包分析 | Charles、mitmproxy(含故障注入脚本)、tcpdump/Wireshark、SSL pinning 与"抓不到包"排查 |
| 日志分析 | adb logcat、iOS 日志与崩溃报告、前端报错定位、后端日志 grep 组合拳 |
| 用例设计与缺陷管理 | 等价类/边界值/场景法/判定表、用例与 Bug 报告模板、测试计划要素 |

## 安装

### Claude Code / Claude Cowork

```bash
git clone https://github.com/ultimatevegance/test-engineer-skill.git
cp -r test-engineer-skill/test-engineer ~/.claude/skills/
```

项目级(团队共享):放到仓库的 `.claude/skills/test-engineer/`。Claude 会在涉及测试的任务中自动触发,无需手动调用。

### OpenAI Codex

Codex 支持 Agent Skills 开放标准,放入 `.agents/skills` 即可:

```bash
git clone https://github.com/ultimatevegance/test-engineer-skill.git
mkdir -p ~/.agents/skills
cp -r test-engineer-skill/test-engineer ~/.agents/skills/
# 重启 Codex 生效
```

项目级:放到仓库的 `.agents/skills/test-engineer/`。

### 其他兼容 Agent Skills 标准的工具

把 `test-engineer/` 目录放到对应工具的 skills 目录即可,本 Skill 为纯 Markdown,无外部依赖、无可执行脚本。

## 结构

```
test-engineer/
├── SKILL.md                        # 角色定位、核心工作原则、任务路由
└── references/
    ├── mobile-testing.md           # 移动端:模拟器操作、Flutter/RN/UniApp、闪退排查
    ├── web-testing.md              # Web:DevTools、Playwright、Vue 组件测试、Lighthouse
    ├── api-testing.md              # API:接口检查清单、NestJS 测试、k6 压测
    ├── packet-capture.md           # 抓包:Charles、mitmproxy、tcpdump、SSL pinning
    ├── log-analysis.md             # 日志:logcat、iOS 崩溃报告、前端/后端日志
    └── test-case-design.md         # 用例设计方法与 Bug 报告模板
```

Skill 采用渐进式加载:主文件只有工作原则和路由表,AI 按任务类型读取对应参考文件,不浪费上下文。

## 设计理念

- **先明确预期,再执行**——没有预期结果的测试只是操作
- **分层定位**——前端 → 网络 → 后端,用证据把问题夹逼到某一层
- **脚本可重复**——用例独立、显式等待、稳定选择器、具体断言
- **产出落地**——用例表格、Bug 报告、排查报告都有固定模板

## English

An Agent Skill (open `SKILL.md` standard) that turns your AI coding assistant into a reliable, professional, detail-oriented QA engineer partner. Covers mobile app testing (iOS simulator / adb / Flutter / React Native), web front-end testing (Chrome DevTools / Playwright / Cypress / Vue), backend API testing (NestJS / Jest / supertest / k6), packet capture (Charles / mitmproxy / tcpdump), log analysis, and test case design. Written in Chinese; all commands and code are universal.

Install: copy the `test-engineer/` folder into `~/.claude/skills/` (Claude Code / Cowork) or `~/.agents/skills/` (OpenAI Codex and other Agent-Skills-compatible tools).

## License

[MIT](./LICENSE)
