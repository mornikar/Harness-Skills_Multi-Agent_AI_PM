# 安全钩子（Security Hooks）— Harness 公共层

> 从 `pm-core/security/secrets-protection.md` 衍生。
> 定义所有 Agent 共享的安全扫描规则和受保护路径。
> 各 Agent 特化层通过 `hooks_override` 增减。

---

## 1. 写入前扫描（Pre-Write Scan）

```yaml
pre_write_scan:
  trigger: "write_to_file / replace_in_file 执行前"
  type: "computational"                    # 确定性正则，不依赖推理
  mandatory: true                          # 不可跳过
  
  patterns:
    - id: "PAT-001"
      name: "API Key"
      regex: '(sk-|pk_|AKIA|ghp_|gho_)[a-zA-Z0-9]{20,}'
      severity: CRITICAL
      
    - id: "PAT-002"
      name: "数据库连接串"
      regex: '(mongodb|mysql|postgres|redis)://[^\s]+:[^\s]+@'
      severity: CRITICAL
      
    - id: "PAT-003"
      name: "密码字段"
      regex: '(password|passwd|pwd|secret|token)\s*[:=]\s*[''""][^'""]{8,}'
      severity: HIGH
      
    - id: "PAT-004"
      name: "私钥"
      regex: '-----BEGIN (RSA |EC |DSA )?PRIVATE KEY-----'
      severity: CRITICAL
      
    - id: "PAT-005"
      name: "Bearer Token"
      regex: 'Bearer\s+[a-zA-Z0-9\-._~+/]+=*'
      severity: CRITICAL
      
    - id: "PAT-006"
      name: "AWS 凭证"
      regex: '(aws_access_key_id|aws_secret_access_key)\s*[:=]\s*[''""][^'""]+'
      severity: CRITICAL

  on_hit:
    action: "abort_write"
    remediation: |
      1. 将敏感值替换为环境变量引用（process.env.{KEY_NAME}）
      2. 在文件头部添加环境变量注释声明
      3. 使用替换后的内容重新执行写入
    notification: |
      send_message(type="message", recipient="main",
        event_type="security_scan_hit",
        pattern_id: "{PAT-XXX}",
        pattern_name: "{pattern_name}",
        severity: "{severity}",
        file: "{file_path}",
        remediation: "已替换为环境变量引用")
    audit: "写入审计日志 security_scan_hit 事件"
```

---

## 2. 受保护路径

```yaml
protected_paths:
  never_read:
    - "**/.env"
    - "**/.env.*"
    - "**/credentials/**"
    - "**/.ssh/**"
    - "**/*.pem"
    - "**/*.key"
    - "**/secrets.*"
    - "**/.aws/credentials"
  enforcement: |
    读取前检查路径是否匹配 never_read 列表。
    匹配 → 阻止读取 + 记录审计 + 通知 orchestrator。
    
  read_only:
    - "package.json"
    - "tsconfig.json"
    - "docker-compose.yml"
    - ".gitignore"
    
  agent_specific_override: {}             # 由各特化层声明
```

---

## 3. 网络访问控制

```yaml
network_policy:
  default: DENY
  allowed_domains:
    - "npmjs.org"
    - "registry.npmjs.org"
    - "pypi.org"
    - "files.pythonhosted.org"
    - "github.com"
    - "api.github.com"
  approval_required: ["任何其他域名"]
  prohibited:
    - "内网IP段（10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16）"
    - "localhost / 127.0.0.1"
    - "169.254.169.254"
```

---

## 4. 环境变量模板

```yaml
env_template:
  trigger: "敏感信息扫描命中并替换后"
  action: |
    1. 创建或更新 .env.example，添加环境变量名（不含真实值）
    2. 确保 .env 在 .gitignore 中
```

---

*Ironforge Base: Security Hooks v1.0 — 2026-04-23*
