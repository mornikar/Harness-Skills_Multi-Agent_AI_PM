# 敏感信息防护规则（Secrets Protection）

> Ironforge 支柱A 组件。
> v3.1 新增。防止 API Key、密码、私钥等敏感信息泄露到代码中。
>
> 路径变量和操作映射见 `pm-core/platform-adapter.md`。

---

## 1. 设计哲学

```
v3 状态：SKILL.md 说"禁止硬编码敏感信息" → Agent 可能遵守也可能无视
v3.1 状态：写入前自动扫描 → 检测到敏感信息 → 阻止写入 + 替换为环境变量引用
```

**核心原则**：检测可计算（正则匹配），不依赖推理。

---

## 2. 写入前扫描

```yaml
pre_write_scan:
  trigger: "write_to_file / replace_in_file 执行前"
  type: "computational"                    # 确定性正则，不依赖推理
  mandatory: true                          # 不可跳过
  
  # ═══════════════════════════════════════
  # 扫描规则
  # ═══════════════════════════════════════
  patterns:
    - name: "API Key"
      id: "PAT-001"
      regex: '(sk-|pk_|AKIA|ghp_|gho_)[a-zA-Z0-9]{20,}'
      severity: CRITICAL
      remediation: "替换为 process.env.{KEY_NAME} 或配置文件引用"
      
    - name: "数据库连接串"
      id: "PAT-002"
      regex: '(mongodb|mysql|postgres|redis)://[^\s]+:[^\s]+@'
      severity: CRITICAL
      remediation: "替换为 process.env.DATABASE_URL"
      
    - name: "密码字段"
      id: "PAT-003"
      regex: '(password|passwd|pwd|secret|token)\s*[:=]\s*[''""][^'""]{8,}'
      severity: HIGH
      remediation: "替换为 process.env.{KEY_NAME}"
      
    - name: "私钥"
      id: "PAT-004"
      regex: '-----BEGIN (RSA |EC |DSA )?PRIVATE KEY-----'
      severity: CRITICAL
      remediation: "从文件读取或使用密钥管理服务"
      
    - name: "Bearer Token"
      id: "PAT-005"
      regex: 'Bearer\s+[a-zA-Z0-9\-._~+/]+=*'
      severity: CRITICAL
      remediation: "替换为 process.env.AUTH_TOKEN"
      
    - name: "AWS 凭证"
      id: "PAT-006"
      regex: '(aws_access_key_id|aws_secret_access_key)\s*[:=]\s*[''""][^'""]+'
      severity: CRITICAL
      remediation: "使用 IAM Role 或 AWS 配置文件"

  # ═══════════════════════════════════════
  # 命中后的处理
  # ═══════════════════════════════════════
  on_hit:
    # 第一步：阻止原始写入
    action: "abort_write"
    
    # 第二步：自动修复
    remediation: |
      1. 识别命中的 pattern ID 和 severity
      2. 将敏感值替换为环境变量引用：
         - API Key → process.env.{KEY_NAME}
         - 数据库连接串 → process.env.DATABASE_URL
         - 密码字段 → process.env.{KEY_NAME}
         - 私钥 → 从配置路径读取
      3. 在文件头部添加环境变量注释声明：
         // Required env vars: {KEY_NAME} — {purpose}
      4. 使用替换后的内容重新执行写入
    
    # 第三步：通知
    notification: |
      send_message(type="message", recipient="main",
        event_type="security_scan_hit",
        agent: "{当前agent}",
        pattern_id: "{PAT-XXX}",
        pattern_name: "{pattern_name}",
        severity: "{severity}",
        file: "{file_path}",
        remediation: "已替换为环境变量引用")
    
    # 第四步：审计
    audit: |
      写入审计日志 security_scan_hit 事件：
      {agent, pattern_id, severity, file, remediation, timestamp}
```

---

## 3. 受保护路径

```yaml
protected_paths:
  # ═══════════════════════════════════════
  # 任何 Agent 都不应读取的路径
  # ═══════════════════════════════════════
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
    如果匹配 → 阻止读取 + 记录审计 + 通知 orchestrator。
    实施：Harness 在 Layer 2 拦截（file_read 前检查路径白名单）。

  # ═══════════════════════════════════════
  # 可读不可写的路径
  # ═══════════════════════════════════════
  read_only:
    - "package.json"
    - "tsconfig.json"
    - "docker-compose.yml"
    - ".gitignore"
    - "README.md"
  
  agent_specific_override:
    pm-runner:
      read_write: ["package.json"]         # runner 可改依赖
    pm-coder:
      read_only: ["src/**/*.test.*"]       # 测试文件只读（防误改）
    pm-writer:
      read_write: ["README.md"]            # writer 可改文档

  # ═══════════════════════════════════════
  # 路径保护审计
  # ═══════════════════════════════════════
  audit_events:
    - event_type: protected_path_access_denied
      detail: {agent, path, access_type, rule}
```

---

## 4. 网络访问控制

```yaml
network_policy:
  # ═══════════════════════════════════════
  # 出站网络访问策略
  # ═══════════════════════════════════════
  default: DENY                           # 默认禁止网络访问
  
  allowed_domains:
    - "npmjs.org"                         # npm install
    - "registry.npmjs.org"                # npm registry
    - "pypi.org"                          # pip install
    - "files.pythonhosted.org"            # pip download
    - "github.com"                        # git clone/pull
    - "api.github.com"                    # GitHub API
  
  approval_required:
    - "任何其他域名"
  
  prohibited:
    - "内网IP段（10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16）"
    - "localhost / 127.0.0.1（防止SSRF）"
    - "169.254.169.254（云元数据端点）"
  
  enforcement: |
    execute_command 执行前检查是否涉及网络访问。
    curl/wget/axios/fetch → 检查目标域名是否在白名单中。
    不在白名单 → 需审批。
    在 prohibited 列表 → 直接阻止。
```

---

## 5. 环境变量模板

当敏感信息被替换为环境变量引用时，需要在项目中创建对应的 `.env.example`：

```yaml
env_template:
  trigger: "敏感信息扫描命中并替换后"
  action: |
    1. 在项目根目录创建或更新 .env.example 文件
    2. 添加替换后的环境变量名（不含真实值）：
       # .env.example
       OPENAI_API_KEY=your-api-key-here
       DATABASE_URL=your-database-url-here
       AUTH_TOKEN=your-auth-token-here
    3. 确保 .env 已添加到 .gitignore
  file_path: ".env.example"
  gitignore_check: "确保 .env 在 .gitignore 中"
```

---

*Ironforge Secrets Protection v1.0 — 2026-04-23*
