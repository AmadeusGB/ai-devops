# Terraform 详细指南

## 项目结构

```
terraform/
├── modules/              # 可复用模块
│   ├── vpc/
│   ├── eks/
│   └── rds/
├── environments/         # 环境配置
│   ├── dev/
│   ├── staging/
│   └── production/
├── backend.tf           # 后端配置
├── variables.tf         # 变量定义
├── outputs.tf          # 输出定义
└── main.tf             # 主配置文件
```

## 命名规范

```hcl
# ✅ 正确：使用有意义的名称
resource "aws_instance" "web_server" {
  ami           = var.web_ami
  instance_type = var.instance_type

  tags = {
    Name        = "${var.environment}-web-server"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# ❌ 错误：使用模糊的名称
resource "aws_instance" "server1" {
  ami           = "ami-xxx"
  instance_type = "t2.micro"
}
```

## 变量管理

```hcl
# ✅ 正确：明确的变量定义和验证
variable "instance_type" {
  description = "EC2 实例类型"
  type        = string
  default     = "t3.medium"

  validation {
    condition     = can(regex("^t3\\.", var.instance_type))
    error_message = "实例类型必须是 t3 系列"
  }
}

# ✅ 正确：使用 locals 处理复杂逻辑
locals {
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
    CreatedAt   = timestamp()
  }
}
```

## 状态后端配置

```hcl
# ✅ 正确：使用远程后端和状态锁定
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

## 模块化示例

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(
    var.tags,
    {
      Name = "${var.name}-vpc"
    }
  )
}

# 使用模块
module "vpc" {
  source = "./modules/vpc"

  name     = "production"
  vpc_cidr = "10.0.0.0/16"

  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
  }
}
```

## 测试和验证

```bash
# 格式检查
terraform fmt -check -recursive

# 验证配置
terraform validate

# 安全检查（使用 tfsec）
tfsec .

# Plan 预览
terraform plan -out=tfplan

# 应用前二次确认
terraform show tfplan
```

## 常见错误

### 错误 1：状态不一致

**问题**：Terraform 状态与实际资源不一致

**解决方案**：
```bash
# 刷新状态
terraform refresh

# 导入现有资源
terraform import aws_instance.example i-1234567890abcdef0

# 移除状态中的资源
terraform state rm aws_instance.example
```

### 错误 2：循环依赖

**问题**：资源之间存在循环依赖

**解决方案**：
```hcl
# ❌ 错误：循环依赖
resource "aws_security_group" "sg1" {
  # ...
  ingress {
    security_groups = [aws_security_group.sg2.id]
  }
}

resource "aws_security_group" "sg2" {
  # ...
  ingress {
    security_groups = [aws_security_group.sg1.id]
  }
}

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
  # ...
}
```

## 检查清单

- [ ] 使用远程后端存储状态
- [ ] 启用状态锁定
- [ ] 所有资源都有有意义的名称和标签
- [ ] 敏感信息使用 sensitive = true
- [ ] 使用变量验证
- [ ] 模块化设计
- [ ] 运行 terraform fmt 格式化代码
- [ ] 运行 terraform validate 验证语法
- [ ] 运行安全扫描工具（tfsec, checkov）
