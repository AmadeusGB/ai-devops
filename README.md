# AI DevOps 实践项目

> 使用 AI（主要是 Claude Code）驱动 DevOps 自动化的探索与实践

## 📖 项目简介

本项目是一个创新性的 DevOps 实践仓库，旨在探索和展示如何利用 AI 技术（特别是 Claude Code）来自动化和优化 DevOps 工作流程。通过精心设计的 Skills、Agents 和 Commands，我们构建了一套完整的 AI 驱动运维解决方案。

## 🎯 项目目标

- **自动化运维**：通过 AI 减少重复性运维工作，提升效率
- **智能决策**：利用 AI 的推理能力辅助复杂运维决策
- **知识沉淀**：将运维经验转化为可复用的 Skills 和 Commands
- **最佳实践**：探索 AI 与 DevOps 结合的最佳实践模式
- **降低门槛**：让运维工作更加易于理解和执行

## 🏗️ 项目架构

```
.claude/
├── skills/                    # 运维技能集合
│   ├── skill-developer/       # 技能开发元技能（必备）
│   ├── error-tracking/        # 错误追踪和监控技能
│   └── skill-rules.json       # 技能触发规则配置
├── agents/                    # 自主执行类 Agent
│   ├── auto-error-resolver.md        # 自动错误解决
│   ├── code-architecture-reviewer.md # 代码架构审查
│   ├── documentation-architect.md    # 文档架构师
│   └── web-research-specialist.md    # 网络研究专家
├── hooks/                     # 事件钩子
│   ├── skill-activation-prompt.ts    # 技能自动激活
│   ├── error-handling-reminder.ts    # 错误处理提醒
│   └── post-tool-use-tracker.sh      # 工具使用追踪
├── commands/                  # 工作流命令（待扩展）
├── settings.json              # Claude Code 配置
└── README.md                  # 配置说明文档
```

## 🔧 核心组件

### 1. Skills（运维技能）

Skills 是可自动激活的知识库，提供领域特定的指导和最佳实践：

#### skill-developer（技能开发元技能）
- **用途**：创建和管理新的 Claude Code 技能
- **何时使用**：需要开发新的 DevOps 技能、理解技能系统、修改技能配置
- **特点**：完整的技能开发指南、中文文档、触发模式设计、Hook 机制说明

#### error-tracking（错误追踪技能）
- **用途**：错误追踪和性能监控模式
- **何时使用**：处理错误、异常、监控相关任务，使用 Sentry 等监控工具
- **特点**：Sentry 集成模式、错误捕获最佳实践、适合 DevOps 运维场景

**更多技能正在开发中**：可使用 skill-developer 创建自定义 DevOps 技能

### 2. Agents（自主执行代理）

Agents 是自主工作的专门代理，处理复杂多步骤任务：

#### auto-error-resolver（自动错误解决）
- **用途**：自动修复 TypeScript/JavaScript 编译错误
- **使用**：`请使用 auto-error-resolver agent 修复构建错误`
- **场景**：构建失败、重构后的类型错误、批量错误解决

#### code-architecture-reviewer（代码架构审查）
- **用途**：审查代码架构一致性和最佳实践
- **使用**：`请使用 code-architecture-reviewer agent 审查代码`
- **场景**：功能实现后审查、合并前检查、验证架构决策

#### documentation-architect（文档架构师）
- **用途**：创建全面的技术文档
- **使用**：`请使用 documentation-architect agent 生成文档`
- **场景**：新功能文档、API 文档、架构概览、开发指南

#### web-research-specialist（网络研究专家）
- **用途**：在线研究技术问题和最佳实践
- **使用**：`请使用 web-research-specialist agent 研究这个问题`
- **场景**：调试疑难问题、查找解决方案、研究最佳实践

### 3. Hooks（事件钩子）

Hooks 在特定事件时自动执行，提供智能提示：

- **skill-activation-prompt**：根据用户意图自动激活相关技能
- **post-tool-use-tracker**：追踪技能使用历史，支持会话管理
- **error-handling-reminder**：温和提醒关注错误处理（不阻塞）

### 4. Commands（工作流命令）

Commands 是预定义的工作流程，自动生成结构化文档：

#### /dev-docs（开发文档命令）
- **用途**：创建开发任务计划和文档
- **使用**：`/dev-docs refactor authentication system`
- **输出**：在 `dev/active/[task-name]/` 生成计划文档、上下文文档、任务清单
- **适用**：大型功能开发、重构任务、架构变更规划

#### /ops-docs（运维文档命令）
- **用途**：创建运维操作文档和检查清单
- **使用**：`/ops-docs database migration from PostgreSQL 12 to 15`
- **输出**：在 `ops/active/[task-name]/` 生成：
  - **运维手册** (runbook) - 完整操作步骤和验证检查
  - **回滚方案** (rollback) - 详细回滚步骤和触发条件
  - **检查清单** (checklist) - 可打印的执行清单
  - **事后总结模板** (postmortem) - 复盘报告模板
- **适用**：数据库迁移、集群升级、CDN 切换、证书更新等重大运维操作

**更多命令**：可在 `.claude/commands/` 添加自定义工作流命令

## 🚀 快速开始

### 前置要求

- [Claude Code](https://claude.com/claude-code) 已安装
- Node.js 和 npm（用于 hooks）
- 基础的 DevOps 知识

### 安装使用

1. **克隆项目**
```bash
git clone https://github.com/AmadeusGB/ai-devops.git
cd ai-devops
```

2. **安装依赖**
```bash
# 安装 hooks 依赖
cd .claude/hooks && npm install && cd ../..
```

3. **验证配置**
```bash
# 检查技能规则配置
cat .claude/skills/skill-rules.json | jq .

# 验证 hooks 权限
ls -la .claude/hooks/*.sh
```

4. **在 Claude Code 中使用**

打开项目后，配置会自动生效。试试这些操作：

```
# 技能会自动激活
"我想创建一个新的监控技能"  → skill-developer 自动激活
"如何追踪错误？"              → error-tracking 自动激活

# 使用 Agent
"请使用 documentation-architect agent 为这个项目生成文档"
"请使用 web-research-specialist agent 研究 Kubernetes 监控方案"
```

### 快速测试

测试技能激活：
```
# 在 Claude Code 中说：
"我想创建一个 Kubernetes 部署技能"

# 应该看到 skill-developer 被激活，提供创建指南
```

## 📚 使用场景

### 场景 1：创建新的 DevOps 技能

使用 skill-developer 快速创建自定义技能：

```
你: "我想创建一个 Kubernetes 部署技能"

Claude: [skill-developer 自动激活]
        好的，我来帮你创建。请告诉我：
        1. 技能名称（建议：kubernetes-deployment）
        2. 主要用途（部署、监控、故障排查等）
        3. 需要包含的内容...
```

### 场景 2：自动错误追踪和监控

error-tracking 技能自动提供监控最佳实践：

```
你: "如何在脚本中添加错误追踪？"

Claude: [error-tracking 自动激活]
        建议使用以下模式...
        [提供 Sentry 集成、日志记录、告警配置等指导]
```

### 场景 3：智能文档生成

使用 documentation-architect agent 生成全面文档：

```
你: "请使用 documentation-architect agent 为部署脚本生成文档"

Claude: [启动 agent]
        [自主分析代码、生成使用说明、API 文档、示例等]
```

### 场景 4：技术问题研究

web-research-specialist agent 帮你快速找到解决方案：

```
你: "请使用 web-research-specialist agent 研究 Prometheus
     在 Kubernetes 中的最佳部署方案"

Claude: [启动 agent]
        [在线搜索、整理方案、对比优缺点、给出建议]
```

### 场景 5：生成运维操作文档

使用 /ops-docs 命令为重大运维任务生成完整文档：

```
你: "/ops-docs PostgreSQL 数据库从 12 升级到 15"

Claude: [自动生成完整运维文档]
        ✅ 已创建运维文档：
        - ops/active/2025-11-03-postgres-upgrade-v12-to-v15/
          ├── postgres-upgrade-runbook.md      (完整操作手册)
          ├── postgres-upgrade-rollback.md     (回滚方案)
          ├── postgres-upgrade-checklist.md    (可打印清单)
          └── postgres-upgrade-postmortem.md   (事后总结模板)

        运维手册包含：
        - 详细操作步骤（可直接执行的命令）
        - 每步验证检查点
        - Go/No-Go 决策点
        - 监控指标

        回滚方案包含：
        - 回滚触发条件
        - 完整回滚步骤
        - 数据恢复程序

        请在执行前：
        1. 团队评审文档
        2. 非生产环境演练
        3. 验证备份可用
```

**更多使用示例**：
```
/ops-docs Kubernetes 集群升级到 v1.28
/ops-docs CDN 从 CloudFlare 切换到 AWS CloudFront
/ops-docs Redis 集群扩容和数据迁移
/ops-docs SSL 证书更新和自动化配置
```

## 🛠️ 技术栈

- **AI 引擎**：Claude Code (Sonnet 4.5)
- **容器编排**：Kubernetes, Docker
- **CI/CD**：GitHub Actions, GitLab CI
- **监控工具**：Prometheus, Grafana, ELK
- **基础设施**：Terraform, Ansible
- **云平台**：AWS, GCP, Azure（可选）

## 📖 文档

### 核心文档

- [📘 .claude 配置说明](.claude/README.md) - Claude Code 完整配置指南
- [🔧 skill-developer 指南](.claude/skills/skill-developer/SKILL.md) - 技能开发完整教程
- [🌏 中文使用指南](.claude/skills/skill-developer/USAGE_GUIDE_CN.md) - 技能开发中文指南
- [📝 中文技能模板](.claude/skills/skill-developer/CHINESE_SKILL_TEMPLATE.md) - 创建中文技能的模板

### 进阶文档

- [⚙️ 技能规则参考](.claude/skills/skill-developer/SKILL_RULES_REFERENCE.md) - skill-rules.json 完整说明
- [🎯 触发模式指南](.claude/skills/skill-developer/TRIGGER_TYPES.md) - 技能触发机制详解
- [🔨 Hook 机制说明](.claude/skills/skill-developer/HOOK_MECHANISMS.md) - Hook 系统深入解析
- [🐛 故障排查指南](.claude/skills/skill-developer/TROUBLESHOOTING.md) - 常见问题解决方案

## 🤝 贡献指南

我们欢迎各种形式的贡献！

1. Fork 本项目
2. 创建特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m '添加某个很棒的功能'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 开启 Pull Request

### 贡献类型

- 🐛 Bug 修复
- ✨ 新功能或 Skill
- 📝 文档改进
- 🎨 代码优化
- 🧪 测试用例

## 📝 更新日志

查看 [CHANGELOG.md](CHANGELOG.md) 了解版本更新历史。

## 📄 许可证

本项目采用 MIT 许可证 - 详见 [LICENSE](LICENSE) 文件

## 🙏 致谢

- [Anthropic](https://www.anthropic.com/) - 提供强大的 Claude AI
- [Claude Code](https://claude.com/claude-code) - 优秀的 AI 开发工具
- 所有贡献者和使用者

## 📮 联系方式

- 项目问题：[GitHub Issues](https://github.com/yourusername/ai-devops/issues)
- 功能建议：[GitHub Discussions](https://github.com/yourusername/ai-devops/discussions)

## 🌟 Star History

如果这个项目对你有帮助，请给我们一个 Star ⭐️

---

**注意**：本项目处于持续开发中，功能和文档会不断完善。欢迎提出建议和反馈！

*由 AI 驱动 · 为 DevOps 赋能 · 让运维更智能*
