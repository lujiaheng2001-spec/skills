# Vibe Coding 安全检查清单

> 数据：5,600 个 vibe-coded 应用中发现 2,000+ 漏洞、400+ 暴露密钥、175 个 PII 泄露。
> AI 生成代码的漏洞率是人类代码的 2.74 倍。这不是偏执，这是现实。
>
> 本清单按开发阶段组织，每个阶段都有对应的自动化检测命令。
> 不需要全部记住——养成习惯，每次交付前跑一遍对应阶段的检查。

---

## 目录

1. [编码阶段：逐行检查](#1-编码阶段逐行检查)
2. [集成阶段：模块间检查](#2-集成阶段模块间检查)
3. [上线阶段：最终门禁](#3-上线阶段最终门禁)
4. [运行阶段：持续监控](#4-运行阶段持续监控)
5. [自动化检测命令速查](#5-自动化检测命令速查)

---

## 1. 编码阶段：逐行检查

每次让 AI 生成代码后，立即检查以下项目。不要等到"最后一起 review"——越晚发现，修复成本越高。

### 1.1 密钥与凭证

| 检查项 | 怎么查 | 修复方式 |
|--------|--------|---------|
| API 密钥硬编码在源码中 | `grep -rn "sk_live\|sk_test\|api_key\|apikey\|secret_key\|password\s*=" --include="*.{js,ts,py,env}"` | 移到 `.env` 文件 + `process.env.XXX` |
| `.env` 文件被提交到 git | `git ls-files \| grep -i env` | 添加 `.env*` 到 `.gitignore`，用 `git rm --cached` 移除 |
| 前端代码中包含密钥 | 检查 `src/` 或 `app/` 目录下的客户端文件 | 密钥只能在服务端代码中使用，前端通过 API 调用获取数据 |
| Docker / CI 配置中有明文密钥 | `grep -rn "password\|secret\|token" Dockerfile docker-compose.yml .github/` | 使用 Docker secrets 或 CI 环境变量 |

**关键认知**：AI 写代码的首要目标是"让它跑起来"，最快的方式就是把密钥直接写死。你必须每次检查，因为 AI 每次都可能这么做。

### 1.2 注入漏洞

| 漏洞类型 | 危险代码模式 | 安全替代 |
|---------|------------|---------|
| SQL 注入 | `` `SELECT * FROM users WHERE id = ${userId}` `` | 参数化查询：`db.query('SELECT * FROM users WHERE id = $1', [userId])` |
| XSS | `element.innerHTML = userInput` | `element.textContent = userInput` 或使用 React（自动转义） |
| 命令注入 | `exec('ls ' + userInput)` | 使用 `execFile` + 参数数组，或完全避免 shell 命令 |
| 路径遍历 | `fs.readFile('/uploads/' + filename)` | 校验 filename 不包含 `..`，使用 `path.resolve` 后检查是否在允许的目录内 |
| NoSQL 注入 | `db.find({ user: req.body.user })` | 校验输入类型，不接受 `$gt`、`$regex` 等操作符 |

**关键认知**：AI 生成的代码中 40-62% 包含注入漏洞。这是因为字符串拼接是"最直接"的实现方式，而 AI 倾向于走最短路径。

### 1.3 认证与授权

| 检查项 | 红旗标志 | 正确做法 |
|--------|---------|---------|
| 认证逻辑在前端 | 前端 JS 里有 `if (user.role === 'admin')` 作为安全控制 | 所有权限检查必须在服务端 API 层 |
| Token 存在 localStorage | `localStorage.setItem('token', ...)` | 使用 HTTP-only、Secure、SameSite cookie |
| 没有检查资源所有权 | API 直接用 URL 中的 ID 查数据库，不校验是否属于当前用户 | 每个查询都加 `WHERE userId = currentUser.id` |
| 忘记密码流程不安全 | 重置链接不过期，或可被暴力枚举 | token 有效期 15 分钟，单次使用，rate limit |
| API 没有 rate limiting | 任何人可以无限次调用 API | 使用 express-rate-limit 或 API gateway 级别限流 |

### 1.4 数据处理

| 检查项 | 常见错误 | 正确做法 |
|--------|---------|---------|
| 密码存储 | 明文存储，或使用 MD5/SHA1 | bcrypt（cost factor ≥ 12）或 Argon2 |
| 日志打印敏感数据 | `console.log(req.body)` 会打印密码 | 创建 sanitize 函数，过滤 password/token/card 字段 |
| 错误响应泄露信息 | 返回完整的 stack trace 或数据库错误 | 生产环境返回通用错误消息，详细信息只记录到服务端日志 |
| 过度收集数据 | 注册时要求身份证、地址等不必要信息 | 只收集业务必需的最少数据 |
| 文件上传不校验 | 接受任何类型和大小的文件 | 限制文件类型（白名单）、大小（如 5MB）、并扫描恶意内容 |

---

## 2. 集成阶段：模块间检查

当多个模块/功能拼在一起时，会出现单独测试时看不到的问题。

### 2.1 跨模块安全

| 检查项 | 说明 |
|--------|------|
| CORS 配置 | 不要用 `*`。明确列出允许的域名。如果是单一域名应用，CORS 可以更严格。 |
| CSP（Content Security Policy） | 配置 CSP header，限制脚本来源。防止即使有 XSS 漏洞也难以利用。 |
| HTTPS | 全站 HTTPS，HTTP 自动 301 到 HTTPS。API 只接受 HTTPS 请求。 |
| Cookie 安全属性 | `HttpOnly`（防 XSS 窃取）、`Secure`（仅 HTTPS）、`SameSite=Lax`（防 CSRF） |
| 请求头安全 | `X-Content-Type-Options: nosniff`、`X-Frame-Options: DENY`、`Referrer-Policy: strict-origin` |

### 2.2 数据流安全

画出数据流图，跟踪敏感数据在系统中的完整路径：

```
用户输入 → 前端校验（UX） → API 请求 → 后端校验（安全边界） → 业务逻辑 → 数据库
                                                                        ↓
用户展示 ← 前端渲染 ← API 响应（过滤敏感字段） ← 数据查询（RLS/权限过滤）
```

每个节点问自己：
- 这里是否有校验？
- 校验是在服务端还是客户端？
- 敏感数据是否被意外暴露？
- 数据是否被正确过滤（用户只能看到自己的数据）？

### 2.3 第三方依赖

| 检查项 | 怎么做 |
|--------|--------|
| 依赖是否真实存在 | `npm view <package-name>` — 如果 404 就是幻觉包 |
| 依赖是否有已知漏洞 | `npm audit` 或 `snyk test` |
| 依赖是否仍在维护 | 看 npm 上的最后更新日期和 GitHub star 数 |
| 是否引入了不必要的依赖 | review `package.json`，问"这个包能用原生实现吗？" |
| 依赖版本是否锁定 | 检查 `package-lock.json` 是否被提交 |

---

## 3. 上线阶段：最终门禁

这些检查必须在每次部署到生产环境之前完成。可以做成 CI/CD 的 gate。

### 3.1 上线前检查脚本

```bash
#!/bin/bash
# pre-deploy-check.sh — 上线前自动检查脚本

echo "=== 密钥扫描 ==="
# 检测是否有密钥泄露到代码中
grep -rn "sk_live\|sk_test\|AKIA[A-Z0-9]\|password\s*=\s*['\"]" \
  --include="*.{js,ts,jsx,tsx,py}" \
  --exclude-dir={node_modules,.next,dist} . && echo "⚠️  发现可能的硬编码密钥！" || echo "✅ 无硬编码密钥"

echo ""
echo "=== .env 检查 ==="
# 确认 .env 不在 git 中
git ls-files | grep -i "\.env" && echo "⚠️  .env 文件在 git 仓库中！" || echo "✅ .env 未被追踪"

echo ""
echo "=== 依赖漏洞扫描 ==="
npm audit --production 2>/dev/null || echo "⚠️  建议运行 npm audit 检查依赖漏洞"

echo ""
echo "=== 生产环境变量检查 ==="
# 检查必要的环境变量是否已设置
for var in DATABASE_URL NEXTAUTH_SECRET NEXTAUTH_URL; do
  if [ -z "${!var}" ]; then
    echo "⚠️  缺少环境变量: $var"
  else
    echo "✅ $var 已设置"
  fi
done

echo ""
echo "=== 测试 ==="
npm test 2>/dev/null || echo "⚠️  测试未通过或未配置"
```

### 3.2 人工检查项

自动化解决不了所有问题，以下需要人工确认：

- [ ] 用两个不同用户账号测试，确认互相看不到对方数据
- [ ] 尝试不登录直接访问 API，确认返回 401
- [ ] 尝试访问不属于自己的资源（改 URL 中的 ID），确认返回 403 或 404
- [ ] 检查浏览器 Network 面板，确认 API 响应中没有多余的敏感字段
- [ ] 检查浏览器 Console，确认没有输出敏感信息
- [ ] 在移动设备上走一遍核心流程

---

## 4. 运行阶段：持续监控

上线不是终点。以下是持续运行的安全实践：

| 监控项 | 工具推荐 | 频率 |
|--------|---------|------|
| 依赖漏洞扫描 | GitHub Dependabot / Snyk | 每天自动 |
| 错误监控 | Sentry | 实时 |
| 访问日志分析 | 观察异常 IP、高频请求 | 每周 review |
| SSL 证书过期 | UptimeRobot | 自动提醒 |
| 密钥轮换 | 手动（设定日历提醒） | 每 90 天 |

---

## 5. 自动化检测命令速查

复制粘贴即可使用的检测命令合集：

```bash
# 1. 扫描硬编码密钥
grep -rn "sk_live\|sk_test\|AKIA\|password\s*=\s*['\"]" \
  --include="*.{js,ts,jsx,tsx}" --exclude-dir=node_modules .

# 2. 查找不安全的 innerHTML 使用
grep -rn "innerHTML\s*=" --include="*.{js,ts,jsx,tsx}" --exclude-dir=node_modules .

# 3. 查找 eval 使用（几乎永远不该在生产代码中出现）
grep -rn "eval(" --include="*.{js,ts,jsx,tsx}" --exclude-dir=node_modules .

# 4. 查找 console.log（生产代码中应移除或替换为 logger）
grep -rn "console\.log" --include="*.{js,ts,jsx,tsx}" --exclude-dir=node_modules .

# 5. 查找宽松的 CORS 配置
grep -rn "Access-Control-Allow-Origin.*\*\|cors({.*origin.*true\|cors()" \
  --include="*.{js,ts}" --exclude-dir=node_modules .

# 6. 查找字符串拼接的 SQL（可能的 SQL 注入）
grep -rn "SELECT.*\+\|INSERT.*\+\|UPDATE.*\+\|DELETE.*\+" \
  --include="*.{js,ts}" --exclude-dir=node_modules .

# 7. 查找 localStorage 存储 token（不安全）
grep -rn "localStorage.*token\|sessionStorage.*token" \
  --include="*.{js,ts,jsx,tsx}" --exclude-dir=node_modules .

# 8. 运行 npm 漏洞扫描
npm audit --production

# 9. 检查未使用的依赖
npx depcheck

# 10. 检查 .env 是否在版本控制中
git ls-files | grep -i "\.env"
```

---

## 紧急响应：发现漏洞后怎么办

如果你发现已经上线的代码有安全问题：

1. **密钥泄露**：立即吊销泄露的密钥，生成新密钥，更新到环境变量。不是"等下个版本修"——现在就做。
2. **数据泄露**：评估影响范围（什么数据、多少用户），根据法规要求（GDPR 72 小时）决定是否需要通知用户。
3. **注入漏洞**：立即修复并部署。检查日志确认是否已被利用。如果有被利用的迹象，当作数据泄露处理。
4. **认证绕过**：立即修复。考虑强制所有用户重新登录（使所有现有 session 失效）。
