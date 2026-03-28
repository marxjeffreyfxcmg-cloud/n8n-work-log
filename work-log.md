# n8n 工作日志

## 2026-03-28 项目初始审查

### 完成事项
- [x] 确认 n8n 实例运行状态 (https://n8n.eeba.cn/)
- [x] 审查 eHunt 选品数据仪表盘 (etsy.html)
- [x] 列出所有 17 个工作流及其状态
- [x] 创建 GitHub 工作日志仓库
- [x] 建立项目概况文档

### 发现摘要
1. **所有 17 个工作流均处于激活状态**，系统运行正常
2. **核心流程**: SOP主调度器 → 13个步骤的完整选品自动化流水线
3. **eHunt仪表盘**展示 Etsy 选品数据，包含价格/销量/收藏/侵权风险等多维度分析
4. **最近更新**: eHunt-Token自动刷新工作流于 3月26日最后更新
5. **配额系统**: 有完整的API配额检查、扣减和每日重置机制

### 待深入了解
- [ ] 各工作流的具体配置和节点详情
- [ ] eHunt API 数据源和调用逻辑
- [ ] AI模型选择和Prompt设计
- [ ] 日报/周报/月报的具体内容和格式
- [ ] 配额限制和使用情况

---

## 2026-03-28 eHunt登录凭证迁移至n8n凭证系统

### 任务描述
将 eHunt-Token自动刷新工作流中硬编码在 PostgreSQL `system_config` 表的登录凭证迁移至 n8n 原生凭证管理系统。

### 完成事项
- [x] 在 n8n 中创建 `eHunt Login Account` 凭证 (httpBasicAuth 类型, ID: YVB0XESA4yYxrPSN)
- [x] 写入 eHunt 登录邮箱和密码到凭证
- [x] 验证 CryptoJS AES-256-CBC 解密逻辑可用
- [x] 在 docker-compose-deploy.yml 中添加 `NODE_FUNCTION_ALLOW_BUILTIN=crypto` 环境变量
- [x] 重启 n8n 容器使配置生效
- [x] 修改工作流 "读取eHunt登录凭证" 节点: 从查询 `system_config` 表改为查询 `credentials_entity` 表
- [x] 修改工作流 "整理凭证" 节点: 新增 CryptoJS 兼容解密逻辑，使用 `crypto` 模块解密凭证数据
- [x] 通过 n8n API 推送工作流更新
- [x] 验证更新后的节点内容正确

### 修改详情

#### 工作流: eHunt-Token自动刷新 (ID: D0o0Q952jalrewzd)

**节点 "读取eHunt登录凭证" (Postgres)**
- 旧SQL: `SELECT key, value FROM system_config WHERE key IN ('ehunt_email', 'ehunt_username', 'ehunt_password')`
- 新SQL: `SELECT data AS encrypted_data FROM credentials_entity WHERE id = 'YVB0XESA4yYxrPSN'`

**节点 "整理凭证" (Code)**
- 旧逻辑: 从 system_config 行中提取 email/password
- 新逻辑: 从 n8n 凭证表读取加密数据，使用 crypto 模块进行 CryptoJS 兼容的 AES-256-CBC 解密

**Docker 配置变更**
- 文件: `/www/wwwroot/n8n/docker-compose-deploy.yml`
- 新增: `NODE_FUNCTION_ALLOW_BUILTIN=crypto`

### 使用方法
- 在 n8n 界面 → 凭证 → `eHunt Login Account` 中修改邮箱/密码
- 工作流下次执行时自动从凭证系统读取新值

### 影响范围
- 仅影响 "eHunt-Token自动刷新" 工作流的凭证读取方式
- 其他工作流不受影响
- n8n 已重启确认健康

### 凭证可见性修复
- 凭证最初被创建到个人项目 (`zOpnVIpmzpGAxqqS`)，在团队项目 "eHunt AI 选品" 中不可见
- 已将凭证移动到团队项目 (`OhuGTTT5zDsyKPN5`)，现在在 UI 中可见

### 用户更新凭证后测试结果 (2026-03-28)
- 用户通过 n8n UI 更新凭证为 `etsyhunt168@gmail.com` / 新密码
- 端到端测试全部通过:
  1. 从 credentials_entity 读取加密数据: OK
  2. AES-256-CBC 解密: OK (正确提取 email/password)
  3. 获取 CSRF Token: OK
  4. 表单登录 eHunt: HTTP 302 重定向到 dashboard-v2
  5. 获得 _identity + ZFSESSID 认证 Cookie: OK
  6. eHunt API 调用验证: OK
- **结论: 凭证系统迁移成功，用户可通过 n8n UI 随时修改 eHunt 账号密码**
