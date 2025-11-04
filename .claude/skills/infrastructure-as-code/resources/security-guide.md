# IaC 安全最佳实践

## 敏感信息管理

### Terraform

```hcl
# ✅ 正确：使用变量和密钥管理服务
variable "db_password" {
  description = "数据库密码"
  type        = string
  sensitive   = true
}

data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "production/database/password"
}

resource "aws_db_instance" "main" {
  # 使用密钥管理服务
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}

# ❌ 错误：硬编码密码
resource "aws_db_instance" "main" {
  password = "hardcoded_password_123"  # 绝对不要这样做！
}
```

### Ansible

```yaml
# ✅ 正确：使用 ansible-vault
# 加密敏感文件
ansible-vault encrypt group_vars/production/secrets.yml

# 在 playbook 中使用
- name: 配置数据库
  template:
    src: db.conf.j2
    dest: /etc/db/config
  vars:
    db_password: "{{ vault_db_password }}"
```

## 最小权限原则

```yaml
# ✅ 正确：最小权限 IAM 策略
- name: 创建 IAM 角色
  iam_role:
    name: app-server-role
    assume_role_policy_document:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Principal:
            Service: ec2.amazonaws.com
          Action: sts:AssumeRole
    managed_policy_arns:
      - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
    inline_policies:
      - policy_name: s3-readonly
        policy_document:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - s3:GetObject
                - s3:ListBucket
              Resource:
                - arn:aws:s3:::my-app-bucket
                - arn:aws:s3:::my-app-bucket/*
```

## 网络安全

```hcl
# ✅ 正确：严格的安全组规则
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Web 服务器安全组"
  vpc_id      = aws_vpc.main.id

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

  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "web-security-group"
  }
}
```

## 安全扫描工具

### Terraform

```bash
# tfsec - Terraform 安全扫描
tfsec .

# checkov - 多云 IaC 安全扫描
checkov -d .

# terrascan - 策略即代码扫描
terrascan scan
```

### Ansible

```bash
# ansible-lint - Ansible 最佳实践检查
ansible-lint playbook.yml

# 检查密码未加密
grep -r "password:" . --include="*.yml"
```

## 安全检查清单

- [ ] 所有敏感信息已加密或使用密钥管理服务
- [ ] IAM 策略遵循最小权限原则
- [ ] 网络安全组规则严格限制访问
- [ ] 启用日志和审计
- [ ] 定期运行安全扫描工具
- [ ] 密钥定期轮换
- [ ] 多因素认证（MFA）已启用
- [ ] 所有数据传输使用加密（TLS/SSL）
- [ ] 备份数据已加密
- [ ] 定期进行安全评审
