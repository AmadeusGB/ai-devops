---
name: infrastructure-as-code
description: 基础设施即代码（IaC）最佳实践指南。涵盖 Terraform、Ansible、CloudFormation 等工具的使用规范、配置模式和 DevOps 最佳实践。适用于编写、修改和审查基础设施配置文件。触发关键词包括 terraform, ansible, infrastructure, deployment config, cloudformation, iac, 基础设施, 部署配置, 自动化部署。
---

# 基础设施即代码（Infrastructure as Code）

## 目的

为 DevOps 工程师和运维人员提供基础设施即代码的最佳实践指南，确保基础设施配置代码的质量、可维护性和安全性。涵盖主流 IaC 工具（Terraform、Ansible、CloudFormation 等）的使用规范和常见模式。

## 何时使用

此技能在以下场景自动激活：

- 编写或修改 Terraform 配置文件（.tf, .tfvars）
- 创建或更新 Ansible playbooks（.yml, .yaml）
- 编辑 CloudFormation 模板（.json, .yaml）
- 提到 IaC、infrastructure、基础设施、部署配置等关键词
- 处理基础设施自动化、资源编排相关任务

## 核心原则

### 1. 声明式优先

**原则**：优先使用声明式（declarative）而非命令式（imperative）配置

**理由**：
- 更容易理解和维护
- 幂等性（idempotent）保证
- 更好的状态管理

### 2. 模块化设计

**原则**：将基础设施拆分为可复用的模块

**好处**：
- 代码复用
- 降低复杂度
- 便于测试和维护

### 3. 版本控制

**原则**：所有 IaC 配置必须使用 Git 进行版本管理

**要求**：
- 有意义的提交信息
- 使用分支策略
- Code Review 流程

### 4. 状态管理

**原则**：妥善管理基础设施状态文件

**注意**：
- 使用远程后端（S3、GCS、Azure Storage）
- 启用状态锁定
- 定期备份状态文件

## 快速参考

### Terraform 基础

```hcl
# ✅ 正确的资源定义
resource "aws_instance" "web_server" {
  ami           = var.web_ami
  instance_type = var.instance_type

  tags = {
    Name        = "${var.environment}-web-server"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# 变量定义带验证
variable "instance_type" {
  description = "EC2 实例类型"
  type        = string
  default     = "t3.medium"

  validation {
    condition     = can(regex("^t3\\.", var.instance_type))
    error_message = "实例类型必须是 t3 系列"
  }
}

# 远程后端配置
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

**详细指南**: [Terraform 详细指南](resources/terraform-guide.md)

### Ansible 基础

```yaml
# ✅ 正确的 playbook 结构
---
- name: 部署 Web 应用
  hosts: webservers
  become: yes

  vars:
    app_version: "1.0.0"

  pre_tasks:
    - name: 检查系统要求
      command: uname -a
      register: system_info
      changed_when: false

  roles:
    - common
    - nginx
    - app

  post_tasks:
    - name: 验证部署
      uri:
        url: "http://localhost"
        status_code: 200

# 幂等性示例
- name: 确保 nginx 已安装
  apt:
    name: nginx
    state: present
    update_cache: yes

- name: 确保配置文件存在
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    mode: '0644'
  notify: 重启 nginx
```

**详细指南**: [Ansible 详细指南](resources/ansible-guide.md)

### CloudFormation 基础

```yaml
# ✅ 正确的 CloudFormation 模板
AWSTemplateFormatVersion: '2010-09-09'
Description: '生产环境 VPC 配置'

Parameters:
  EnvironmentName:
    Type: String
    AllowedValues:
      - dev
      - staging
      - production
    Default: dev

  VpcCIDR:
    Type: String
    Default: 10.0.0.0/16
    AllowedPattern: '^(\d{1,3}\.){3}\d{1,3}/\d{1,2}$'

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-vpc'

Outputs:
  VPCId:
    Value: !Ref VPC
    Export:
      Name: !Sub '${EnvironmentName}-VPC-ID'
```

## 安全最佳实践

### 敏感信息管理

```hcl
# ✅ 正确：使用变量和密钥管理
variable "db_password" {
  description = "数据库密码"
  type        = string
  sensitive   = true
}

data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "production/database/password"
}

# ❌ 错误：硬编码密码
resource "aws_db_instance" "main" {
  password = "hardcoded_password"  # 绝对不要这样做！
}
```

### 最小权限原则

```hcl
# ✅ 正确：严格的安全组规则
resource "aws_security_group" "web" {
  name = "web-sg"

  ingress {
    description = "HTTPS from internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description     = "HTTP from ALB only"
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }
}
```

**详细安全指南**: [安全最佳实践](resources/security-guide.md)

## 测试和验证

### Terraform

```bash
# 格式检查
terraform fmt -check -recursive

# 验证配置
terraform validate

# 安全检查
tfsec .

# Plan 预览
terraform plan -out=tfplan
```

### Ansible

```bash
# 语法检查
ansible-playbook --syntax-check playbook.yml

# 模拟运行
ansible-playbook --check playbook.yml

# 最佳实践检查
ansible-lint playbook.yml
```

## 常见错误和解决方案

### Terraform 状态不一致

```bash
# 刷新状态
terraform refresh

# 导入现有资源
terraform import aws_instance.example i-1234567890abcdef0

# 移除状态中的资源
terraform state rm aws_instance.example
```

### Ansible 幂等性问题

```yaml
# ✅ 正确：对于 command 模块指定 changed_when
- name: 检查配置
  command: /usr/bin/check-config
  register: config_check
  changed_when: config_check.rc != 0
  failed_when: false
```

### Terraform 循环依赖

```hcl
# ✅ 正确：使用独立的规则资源
resource "aws_security_group" "sg1" {
  # ...
}

resource "aws_security_group" "sg2" {
  # ...
}

resource "aws_security_group_rule" "sg1_to_sg2" {
  type                     = "ingress"
  security_group_id        = aws_security_group.sg1.id
  source_security_group_id = aws_security_group.sg2.id
}
```

## 项目结构最佳实践

### Terraform

```
terraform/
├── modules/          # 可复用模块
├── environments/     # 环境配置
│   ├── dev/
│   ├── staging/
│   └── production/
├── backend.tf       # 后端配置
├── variables.tf     # 变量定义
└── main.tf          # 主配置
```

### Ansible

```
ansible/
├── inventories/     # 清单文件
│   ├── dev/
│   └── production/
├── roles/          # 角色
├── playbooks/      # Playbook
├── group_vars/     # 组变量
└── ansible.cfg     # 配置
```

## 检查清单

### Terraform
- [ ] 使用远程后端存储状态
- [ ] 启用状态锁定
- [ ] 所有资源都有名称和标签
- [ ] 敏感信息使用 sensitive = true
- [ ] 使用变量验证
- [ ] 模块化设计
- [ ] 运行 terraform fmt
- [ ] 运行 terraform validate
- [ ] 运行安全扫描（tfsec）

### Ansible
- [ ] 所有 task 都有描述性 name
- [ ] 使用幂等的模块
- [ ] 合理使用 handler
- [ ] 变量按环境组织
- [ ] 敏感信息使用 ansible-vault
- [ ] 运行 ansible-lint
- [ ] 使用 --check 模式测试

### 通用
- [ ] 代码已版本控制
- [ ] 清晰的提交信息
- [ ] 通过 Code Review
- [ ] 更新相关文档
- [ ] 非生产环境测试
- [ ] 有回滚计划

## 相关资源

- [Terraform 详细指南](resources/terraform-guide.md) - 完整的 Terraform 最佳实践
- [Ansible 详细指南](resources/ansible-guide.md) - 完整的 Ansible 最佳实践
- [安全最佳实践](resources/security-guide.md) - IaC 安全指南

---

**提示**：基础设施即代码是 DevOps 的核心实践，确保每次更改都经过充分测试和审查。
