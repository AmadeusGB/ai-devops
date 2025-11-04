# [任务名称] 回滚方案

**最后更新**: YYYY-MM-DD HH:MM
**关联 Runbook**: [runbook-文件名.md]
**版本**: v1.0

---

## 1. 回滚概述

### 回滚策略
[描述回滚的总体策略和方法]

### 回滚窗口
- **可回滚时间**: [描述在什么时间范围内可以回滚]
- **回滚耗时**: 预计 XX 分钟
- **数据影响**: [回滚是否会丢失数据]

---

## 2. 回滚触发条件

### 强制回滚（必须立即回滚）
- [ ] 服务完全不可用超过 X 分钟
- [ ] 错误率超过 X%
- [ ] 数据损坏或丢失
- [ ] 安全漏洞暴露
- [ ] [其他关键条件]

### 建议回滚（评估后决定）
- [ ] 性能下降超过 X%
- [ ] 部分功能不可用
- [ ] 告警频繁触发
- [ ] [其他条件]

### 决策矩阵
| 问题严重度 | 影响范围 | 持续时间 | 决策 |
|----------|---------|---------|------|
| 高 | 全部用户 | > 5分钟 | 立即回滚 |
| 高 | 部分用户 | > 15分钟 | 评估回滚 |
| 中 | 全部用户 | > 30分钟 | 评估回滚 |
| 低 | 少量用户 | 任意 | 继续监控 |

---

## 3. 回滚前检查

### 确认项
- [ ] 已确认需要回滚
- [ ] 回滚负责人已到位
- [ ] 备份数据可用且已验证
- [ ] 回滚方案已评审
- [ ] 相关团队已通知

### 数据备份验证
```bash
# 验证备份完整性
backup-verify-command

# 检查备份时间点
backup-info-command

# 预期输出
Backup created at: YYYY-MM-DD HH:MM:SS
Backup size: XXX MB
Backup status: VALID
```

---

## 4. 回滚步骤

### 阶段 1: 准备回滚 (预计 X 分钟)

#### 步骤 1.1: 启用维护模式
**目的**: 停止新的请求，保护数据一致性

```bash
# 启用维护模式
enable-maintenance-mode

# 验证
curl https://app.example.com
# 预期: 503 Maintenance Mode
```

**检查点**:
- [ ] 维护模式已启用
- [ ] 用户看到维护页面
- [ ] 后台任务已暂停

---

#### 步骤 1.2: 停止相关服务
**目的**: 确保服务完全停止，避免数据冲突

```bash
# 停止应用服务
systemctl stop application.service

# 或 Kubernetes
kubectl scale deployment app --replicas=0

# 验证服务已停止
systemctl status application.service
# 预期: inactive (dead)
```

**检查点**:
- [ ] 应用服务已停止
- [ ] 相关进程已终止
- [ ] 无活跃连接

---

### 阶段 2: 执行回滚 (预计 X 分钟)

#### 步骤 2.1: 回滚代码/配置
```bash
# Git 回滚
git checkout <previous-commit-hash>

# 或 Kubernetes
kubectl rollout undo deployment/app

# 或 Terraform
terraform apply -target=module.app -var="version=previous"

# 验证
git log -1 --oneline
# 预期: <previous-commit-hash> Previous version
```

**检查点**:
- [ ] 代码已回滚到前一版本
- [ ] 配置文件已恢复
- [ ] 版本号正确

---

#### 步骤 2.2: 回滚数据库（如需要）
**⚠️ 关键操作 - 不可逆**

```bash
# 停止数据库写入
# 方法1: 设置只读
mysql -e "SET GLOBAL read_only = ON;"

# 恢复数据库备份
mysql -u root -p database_name < /backup/database_backup_YYYYMMDD.sql

# 或 PostgreSQL
pg_restore -d database_name /backup/database_backup_YYYYMMDD.dump

# 验证数据恢复
mysql -e "SELECT COUNT(*) FROM important_table;"
# 预期: 与备份时的数量一致
```

**检查点**:
- [ ] 数据库已停止写入
- [ ] 备份已恢复
- [ ] 数据完整性验证通过
- [ ] 关键表数据正确

---

#### 步骤 2.3: 回滚基础设施（如需要）
```bash
# Terraform 回滚
terraform plan -out=rollback.tfplan
terraform apply rollback.tfplan

# 或 Ansible
ansible-playbook -i inventory rollback.yml

# 验证
terraform show
```

---

### 阶段 3: 启动服务 (预计 X 分钟)

#### 步骤 3.1: 启动服务
```bash
# 启动应用
systemctl start application.service

# 或 Kubernetes
kubectl scale deployment app --replicas=3

# 验证服务状态
systemctl status application.service
# 预期: active (running)

# 验证 Pod 状态
kubectl get pods -l app=app
# 预期: 3/3 Running
```

**检查点**:
- [ ] 服务成功启动
- [ ] 无错误日志
- [ ] 所有实例健康

---

#### 步骤 3.2: 健康检查
```bash
# 应用健康检查
curl http://localhost:8080/health
# 预期: {"status": "healthy"}

# 数据库连接检查
curl http://localhost:8080/health/db
# 预期: {"database": "connected"}
```

---

### 阶段 4: 验证回滚 (预计 X 分钟)

#### 步骤 4.1: 功能验证
- [ ] 关键功能1测试通过
- [ ] 关键功能2测试通过
- [ ] 关键功能3测试通过

```bash
# 自动化测试
./run-smoke-tests.sh

# 预期: All tests passed
```

#### 步骤 4.2: 性能验证
```bash
# 性能测试
ab -n 1000 -c 10 http://localhost:8080/

# 检查响应时间
# 预期: < 200ms 平均响应时间
```

#### 步骤 4.3: 数据验证
```bash
# 数据一致性检查
./check-data-integrity.sh

# 预期: No inconsistencies found
```

---

### 阶段 5: 恢复服务 (预计 X 分钟)

#### 步骤 5.1: 关闭维护模式
```bash
# 关闭维护模式
disable-maintenance-mode

# 验证
curl https://app.example.com
# 预期: 200 OK
```

**检查点**:
- [ ] 维护模式已关闭
- [ ] 用户可以正常访问
- [ ] 流量恢复正常

---

## 5. 回滚后验证

### 完整验证清单
- [ ] 所有服务运行正常
- [ ] 应用版本正确
- [ ] 数据库版本正确
- [ ] 配置文件正确
- [ ] 关键业务功能正常
- [ ] 性能指标正常
- [ ] 错误率在正常范围
- [ ] 无告警触发
- [ ] 日志无异常
- [ ] 监控面板显示正常

### 监控指标
| 指标 | 回滚前 | 回滚后 | 状态 |
|-----|-------|-------|------|
| 可用性 | [值] | [值] | ✅/❌ |
| 错误率 | [值] | [值] | ✅/❌ |
| 响应时间 | [值] | [值] | ✅/❌ |
| 吞吐量 | [值] | [值] | ✅/❌ |

---

## 6. 数据处理

### 数据丢失评估
[描述回滚可能导致的数据丢失]

### 数据补偿方案
[如何处理丢失的数据]

```bash
# 数据补偿脚本
./compensate-data.sh --from "HH:MM" --to "HH:MM"
```

---

## 7. 通知

### 回滚开始通知
**发送对象**: [团队/用户]
**通知方式**: [邮件/短信/IM]
**通知内容**:
```
主题：[系统名称] 正在执行回滚操作

我们检测到 [描述问题]，正在执行回滚操作。
- 开始时间: [时间]
- 预计恢复: [时间]
- 影响范围: [描述]

我们将持续监控并及时更新进展。
```

### 回滚完成通知
**通知内容**:
```
主题：[系统名称] 回滚操作已完成

回滚操作已成功完成，系统已恢复正常。
- 回滚完成时间: [时间]
- 系统状态: 正常
- 验证结果: 所有检查通过

我们将继续监控系统运行状况。
```

---

## 8. 故障排查

### 回滚失败场景

#### 场景 1: 服务启动失败
**现象**: 服务无法启动或持续重启

**排查步骤**:
```bash
# 查看日志
journalctl -u application.service -n 100

# 检查配置
config-validate-command

# 检查依赖
check-dependencies-command
```

**解决方案**:
[步骤]

---

#### 场景 2: 数据库恢复失败
**现象**: 数据库备份恢复失败

**排查步骤**:
```bash
# 检查备份文件
backup-validate-command

# 检查磁盘空间
df -h

# 检查数据库日志
tail -f /var/log/mysql/error.log
```

**解决方案**:
[步骤]

---

## 9. 回滚后操作

### 立即行动
- [ ] 通知所有相关方回滚已完成
- [ ] 更新事件跟踪系统
- [ ] 启动问题根因分析
- [ ] 更新监控告警规则

### 后续行动
- [ ] 24小时内完成 Postmortem
- [ ] 更新运维文档
- [ ] 改进回滚流程
- [ ] 修复导致回滚的问题

---

## 10. 应急联系

### 回滚团队
- **回滚负责人**: [姓名] - [电话]
- **DBA**: [姓名] - [电话]
- **运维工程师**: [姓名] - [电话]

### 升级路径
| 问题类型 | 联系人 | 响应时间 |
|---------|-------|---------|
| 回滚执行问题 | [姓名/电话] | 立即 |
| 数据问题 | [姓名/电话] | 立即 |
| 业务影响 | [姓名/电话] | 15分钟 |

---

**重要提醒**:
1. 回滚是最后的保险措施，决策需谨慎
2. 回滚前必须验证备份可用
3. 每个步骤都要验证结果
4. 保持与业务团队的沟通
5. 详细记录回滚过程和结果
