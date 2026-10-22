# verify-change-safety

一个通用的「改完复查」工具，给 Cursor Agent 用。它会在你改完代码后主动复查
**正确性 / 风险 / 性能**，派一个**独立的红队子 Agent** 专门推翻自己的结论、找遗漏，
按**三态裁决**给出结论，能安全快修的就修，通过则按规范**自动提交**。

> 核心机制（对抗式复核 + 三态裁决）的灵感来自
> [Cloudflare security-audit-skill](https://github.com/cloudflare/security-audit-skill)——
> 把「AI 做主观评审」最容易翻车的三个毛病（自我确认、静默跳项、虚报严重程度）用结构框死。

## 功能特性

- **通用化**：不绑定任何技术栈。仓库若有自己的约定（`AGENTS.md` / `CLAUDE.md` / 项目级
  `.cursor/rules/`），优先遵循；没有就退回内置通用检查。
- **对抗式复核**：主 Agent 出结论后，派发独立子 Agent（`verify-change-reviewer`）证伪自己，
  每条推翻点强制带「入口 → 传播 → 触发」证据链。
- **三态裁决**：`确认问题` / `待验证` / `通过`。`待验证` 态**禁止填严重程度**
  （不确定 ≠ 低危），综合风险不得超过已证明的实际影响。
- **自动提交**：结论为「通过 / 已修复后通过」且存在相关未提交改动时，按规范创建 commit
  （调用本工具即视为授权提交，不另问；不 push，除非你明确要求）。
- **零依赖、零配置**：不需要 API key、不需要 `npm install`、不需要联网。引擎就是你手头
  Cursor 里已经在跑的那个模型。

## 文件结构

```
.cursor/
├── agents/
│   ├── verify-change-safety/SKILL.md     # 主工具（复查→对抗→修复→提交）
│   └── verify-change-reviewer/SKILL.md   # 红队子 Agent（独立证伪）
└── rules/
    ├── post-edit-verification.mdc        # 终检清单（通用）
    ├── minimal-change-scope.mdc          # 最小改动纪律（通用）
    ├── backend-guardrails.mdc            # 后端通用底线（长 I/O 不持连接等）
    └── frontend-guardrails.mdc          # 前端通用底线（统一封装、XSS 等）
```

## 安装

两种方式，任选其一。

### 方式 A：项目级（推荐，克隆即生效）

把本仓库的 `.cursor/` 目录合并进你的项目根目录：

```bash
# 在你的项目里
cp -r path/to/verify-change-safety/.cursor ./
```

用 Cursor 打开该项目，`.cursor/agents/` 与 `.cursor/rules/` 会被自动识别，无需其它操作。

### 方式 B：全局（对所有项目生效）

```bash
cp -r verify-change-safety/.cursor/agents/* ~/.cursor/agents/
cp -r verify-change-safety/.cursor/rules/*  ~/.cursor/rules/
```

## 使用

在 Cursor 里对已改动的工作区说「检查代码」「确保实现正确」「有无风险/性能回归」，
或直接输入：

```
/verify-change-safety
```

它会：拉齐 diff → 对照需求复查 → 派发红队复核 → 综合三态结论 → 能修就修 → 通过则提交。

输出示例：

```markdown
复查结论：已修复后通过

**正确性** - …
**风险** - 无明显新增风险
**性能** - 无性能回归
**对抗复核** - reviewer 指出 X，已吸收修复
**提交** - 已提交：<sha> Fix: …
```

## 前置要求

- **支持子 Agent（Subagents）的 Cursor 版本**（作者使用 3.26.x）。对抗复核这一步依赖主 Agent
  能派发独立子 Agent。
- 主 Agent 与红队 Agent 均设 `disable-model-invocation: true`：只能由你手动触发
  （`/verify-change-safety`）或彼此调用，**不会被模型自动误触发**。

## 已知限制 / 降级

- 若你的 Cursor 版本暂不支持「Agent 派发 Agent」，主工具会退化为**内联红队**：在同一 Agent
  内切换视角自我证伪（效果略弱，但零依赖）。届时 `verify-change-reviewer` 仍可保留以备升级。
- 本工具是**复查 + 修复 + 提交**闭环，不是只出报告的审计器；它的目标是「改完快速收尾」，
  不是「全仓深度体检」——后者请直接用 Cloudflare security-audit-skill。
- 三态裁决与对抗复核提升的是**可靠性**（少漏报、少自欺），不是数量级跃迁。

## 致谢

对抗式验证与三态裁决的设计思路借鉴自
[Cloudflare/security-audit-skill](https://github.com/cloudflare/security-audit-skill)
（Apache-2.0）。本仓库的适配与通用化改造独立于原项目。

## License

MIT —— 见 [LICENSE](./LICENSE)。
