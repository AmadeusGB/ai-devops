---
description: Create comprehensive operations documentation for critical infrastructure tasks
argument-hint: Describe the operations task (e.g., "database migration", "kubernetes cluster upgrade", "CDN switchover")
---

你是一位资深 SRE 和运维专家。为以下运维任务创建完整的操作文档：$ARGUMENTS

## 指令

1. **分析运维任务** 并确定任务范围和影响
2. **检查相关基础设施** 了解当前状态和配置
3. **创建结构化运维文档**，包含：
   - 执行概要（Executive Summary）
   - 任务目标和背景
   - 现状分析（Current State）
   - 影响范围评估（Impact Assessment）
   - 操作计划（Operation Plan）
   - 详细操作步骤（Step-by-Step Procedures）
   - 验证检查点（Validation Checkpoints）
   - 回滚方案（Rollback Plan）
   - 风险评估和应急预案（Risk Assessment & Contingency）
   - 成功标准（Success Criteria）
   - 所需资源和依赖（Required Resources）
   - 时间窗口和计划（Timeline & Schedule）
   - 通知和协调计划（Communication Plan）

4. **操作步骤结构**：
   - 每个步骤必须包含：
     * 操作命令（实际可执行的命令）
     * 预期输出（Expected Output）
     * 检查点（Checkpoint）
     * 估计时间
   - 标注关键步骤和不可逆操作
   - 包含前置条件和依赖检查
   - 在关键节点添加决策点（Go/No-Go Decision Points）

5. **回滚方案要求**：
   - 必须包含完整的回滚步骤
   - 明确回滚触发条件
   - 回滚所需时间估计
   - 数据备份和恢复程序
   - 回滚后验证步骤

6. **创建运维文档结构**：
   - 创建目录：`ops/active/[task-name]/`（相对于项目根目录）
   - 生成四个文件：
     * `[task-name]-runbook.md` - 完整的操作手册
     * `[task-name]-rollback.md` - 详细的回滚方案
     * `[task-name]-checklist.md` - 操作检查清单（可打印）
     * `[task-name]-postmortem-template.md` - 事后总结模板
   - 每个文件包含 "最后更新: YYYY-MM-DD HH:MM" 时间戳

## 质量标准

- 所有命令必须可直接复制执行
- 使用清晰、明确的操作语言（"执行 X"而非"可以考虑执行 X"）
- 包含所有必要的技术细节和配置参数
- 考虑多种故障场景和边界情况
- 所有关键操作必须有验证步骤
- 回滚方案必须经过验证可行

## 安全和合规要求

- 标注所有需要特殊权限的操作
- 包含审计日志收集步骤
- 明确数据备份和验证程序
- 标识所有不可逆操作并要求确认
- 包含安全检查清单

## 上下文引用

- 检查 `INFRASTRUCTURE.md` 了解基础设施架构（如存在）
- 查阅 `RUNBOOKS.md` 参考已有运维流程（如存在）
- 参考 `INCIDENT_RESPONSE.md` 了解应急响应流程（如存在）
- 使用 `ops/README.md` 了解运维任务管理规范（如存在）
- 检查 `.claude/skills/infrastructure-as-code/` 和 `.claude/skills/container-orchestration/` 获取最佳实践

## 通知模板

在操作手册中包含以下通知模板：

**操作前通知**：
- 发送对象、时间、内容
- 包含操作窗口、影响范围、联系方式

**操作中通知**：
- 关键节点更新频率
- 升级路径和联系方式

**操作后通知**：
- 成功/失败通知模板
- 需要的监控数据和验证结果

## 输出格式规范

### Runbook 必须包含的章节
```markdown
# [任务名称] 运维操作手册

## 1. 执行概要
## 2. 任务信息
## 3. 影响范围
## 4. 前置条件
## 5. 操作步骤
## 6. 验证检查
## 7. 监控指标
## 8. 故障处理
## 9. 联系方式
## 10. 附录
```

### Checklist 格式要求
```markdown
# [任务名称] 执行检查清单

**操作日期**: ___________
**操作人员**: ___________
**审核人员**: ___________

## 操作前检查
- [ ] 项目 1
- [ ] 项目 2

## 操作中检查
- [ ] 步骤 1：___________
  - 执行时间: ___________
  - 验证结果: ___________

## 操作后检查
- [ ] 验证 1
- [ ] 验证 2
```

## 使用场景

此命令适用于：
- 重大基础设施变更（数据库迁移、集群升级）
- 计划内维护操作（系统升级、配置变更）
- 紧急故障恢复演练（灾难恢复、数据恢复）
- 安全加固和合规操作（证书更新、漏洞修复）
- 容量规划和扩容（资源扩容、性能优化）

**最佳实践**：在执行任何重大运维操作前 24-48 小时运行此命令，生成的文档应经过同行评审并在非生产环境演练验证。

**注意**：生成的文档应作为正式的运维工单附件，并在执行过程中实时更新操作记录。
