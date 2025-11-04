# Ansible 详细指南

## 项目结构

```
ansible/
├── inventories/         # 清单文件
│   ├── dev/
│   ├── staging/
│   └── production/
├── roles/              # 角色
│   ├── common/
│   ├── web/
│   └── db/
├── playbooks/          # Playbook
│   ├── deploy.yml
│   └── maintenance.yml
├── group_vars/         # 组变量
├── host_vars/          # 主机变量
└── ansible.cfg         # Ansible 配置
```

## Playbook 编写规范

```yaml
# ✅ 正确：清晰的 playbook 结构
---
- name: 部署 Web 应用
  hosts: webservers
  become: yes

  vars:
    app_version: "1.0.0"
    deploy_path: "/opt/app"

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
      retries: 3
      delay: 5

# ❌ 错误：缺少描述和错误处理
---
- hosts: all
  tasks:
    - command: do_something
```

## 幂等性原则

```yaml
# ✅ 正确：幂等的任务
- name: 确保 nginx 已安装
  apt:
    name: nginx
    state: present
    update_cache: yes

- name: 确保配置文件存在
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
  notify: 重启 nginx

# ❌ 错误：不幂等的命令
- name: 安装软件
  command: apt-get install nginx
```

## 变量优先级

```yaml
# group_vars/production.yml
---
environment: production
debug_mode: false
db_host: "prod-db.example.com"

# host_vars/web01.yml
---
server_role: "primary"
backup_enabled: true

# playbook 中使用
- name: 配置应用
  template:
    src: app.conf.j2
    dest: "/etc/app/config.yml"
  vars:
    # Playbook 级别变量（最高优先级）
    custom_setting: "{{ environment }}-custom"
```

## Handler 使用

```yaml
# ✅ 正确：使用 handler 处理服务重启
tasks:
  - name: 更新 nginx 配置
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify:
      - 验证 nginx 配置
      - 重启 nginx

handlers:
  - name: 验证 nginx 配置
    command: nginx -t

  - name: 重启 nginx
    service:
      name: nginx
      state: restarted
```

## 测试

```bash
# 语法检查
ansible-playbook --syntax-check playbook.yml

# 模拟运行（不实际执行）
ansible-playbook --check playbook.yml

# 列出任务
ansible-playbook --list-tasks playbook.yml

# 列出标签
ansible-playbook --list-tags playbook.yml

# 使用 molecule 测试 role
molecule test
```

## 常见错误

### 错误：忽略幂等性

**问题**：Ansible 任务每次都显示 changed

**解决方案**：
```yaml
# ✅ 正确：使用幂等的模块
- name: 确保文件存在
  copy:
    content: "{{ config_content }}"
    dest: /etc/app/config.yml
    owner: app
    group: app
    mode: '0644'

# 对于必须使用 command 的情况
- name: 检查配置
  command: /usr/bin/check-config
  register: config_check
  changed_when: config_check.rc != 0
  failed_when: false
```

## 检查清单

- [ ] 所有 playbook 和 task 都有描述性的 name
- [ ] 使用幂等的模块
- [ ] 合理使用 handler
- [ ] 变量文件按环境组织
- [ ] 敏感信息使用 ansible-vault 加密
- [ ] 运行 ansible-lint 检查
- [ ] 使用 --check 模式测试
