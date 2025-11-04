# 运维任务管理

本目录用于管理所有运维操作的文档和计划。

## 目录结构

```
ops/
├── active/          # 当前活跃的运维任务
│   └── [task-name]/
│       ├── [task-name]-runbook.md            # 操作手册
│       ├── [task-name]-rollback.md           # 回滚方案
│       ├── [task-name]-checklist.md          # 检查清单
│       └── [task-name]-postmortem-template.md # 事后总结模板
├── archive/         # 已完成的运维任务归档
└── templates/       # 运维文档模板
```

## 使用 /ops-docs 命令

使用 Claude Code 的 `/ops-docs` 命令可以自动生成完整的运维文档结构：

```bash
# 在 Claude Code 中执行
/ops-docs database migration from PostgreSQL 12 to 15
/ops-docs kubernetes cluster upgrade to v1.28
/ops-docs CDN provider switchover from CloudFlare to AWS CloudFront
```

## 运维任务生命周期

### 1. 规划阶段
- 使用 `/ops-docs` 命令生成初始文档
- 文档进入 `ops/active/[task-name]/` 目录
- 团队评审运维文档
- 在非生产环境验证操作步骤

### 2. 执行前准备
- [ ] 完成前置条件检查
- [ ] 获得变更审批
- [ ] 通知相关团队
- [ ] 确认时间窗口
- [ ] 准备回滚方案
- [ ] 打印检查清单

### 3. 执行阶段
- 严格按照 runbook 执行
- 在 checklist 上记录每个步骤
- 在关键节点进行 Go/No-Go 决策
- 持续监控系统指标
- 记录所有异常情况

### 4. 执行后
- 完成验证检查
- 发送完成通知
- 填写 postmortem 模板
- 归档文档到 `ops/archive/`
- 更新 runbook（如有改进）

## 文档命名规范

### 任务名称格式
```
[日期]-[系统]-[操作类型]-[简要描述]

示例：
2025-11-03-postgres-upgrade-v12-to-v15
2025-11-10-k8s-cluster-upgrade-v1.28
2025-11-15-cdn-switchover-cloudflare-to-aws
```

### 文件命名
- Runbook: `[task-name]-runbook.md`
- Rollback: `[task-name]-rollback.md`
- Checklist: `[task-name]-checklist.md`
- Postmortem: `[task-name]-postmortem-template.md`

## 运维文档标准

### Runbook 必须包含
1. ✅ 执行概要（Executive Summary）
2. ✅ 影响范围评估
3. ✅ 详细操作步骤（可执行命令）
4. ✅ 每个步骤的验证检查点
5. ✅ 预期输出和错误处理
6. ✅ 时间估计
7. ✅ 决策点（Go/No-Go）
8. ✅ 监控指标

### Rollback Plan 必须包含
1. ✅ 回滚触发条件
2. ✅ 完整回滚步骤
3. ✅ 回滚时间估计
4. ✅ 数据恢复程序
5. ✅ 回滚后验证

### Checklist 特点
- 可打印格式
- 包含签字确认栏
- 记录实际执行时间
- 记录验证结果

## 运维最佳实践

### 操作前
1. **评审文档**：至少 2 人评审 runbook
2. **演练验证**：在非生产环境完整演练
3. **备份数据**：执行前完成所有必要备份
4. **通知团队**：提前通知所有相关方
5. **准备回滚**：确认回滚方案可行

### 操作中
1. **严格执行**：不偏离 runbook
2. **实时记录**：在 checklist 上记录进度
3. **持续监控**：关注系统指标
4. **及时沟通**：定期更新操作状态
5. **果断决策**：遇到问题立即评估是否回滚

### 操作后
1. **全面验证**：完成所有验证检查
2. **持续监控**：操作后 24-48 小时密切监控
3. **事后总结**：填写 postmortem 文档
4. **知识沉淀**：更新 runbook 和最佳实践
5. **归档文档**：将文档移至 archive 目录

## 应急响应

如果运维操作出现问题：

1. **立即评估**：评估问题严重程度
2. **决策回滚**：如果问题严重，立即执行回滚
3. **升级通知**：通知相关负责人
4. **记录日志**：详细记录问题现象和处理过程
5. **故障恢复**：按照应急预案执行

## 相关资源

- [Infrastructure as Code Skill](../.claude/skills/infrastructure-as-code/) - IaC 最佳实践
- [Container Orchestration Skill](../.claude/skills/container-orchestration/) - 容器编排最佳实践
- `/ops-docs` - 自动生成运维文档命令

## 归档规则

任务完成后 30 天内归档：

```bash
# 归档示例
mv ops/active/2025-11-03-postgres-upgrade-v12-to-v15/ \
   ops/archive/2025/11/2025-11-03-postgres-upgrade-v12-to-v15/
```

归档目录结构：
```
ops/archive/
└── [YYYY]/
    └── [MM]/
        └── [task-name]/
```

## 常用命令

```bash
# 创建新运维任务
/ops-docs [描述运维任务]

# 查看活跃任务
ls ops/active/

# 查看归档任务
ls ops/archive/
```

---

**重要提示**：所有重大运维操作必须生成完整的运维文档，并经过评审和演练后才能在生产环境执行。
