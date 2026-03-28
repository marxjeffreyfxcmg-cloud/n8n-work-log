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

---

## 2026-03-28 eHunt登录流程精简 - 多节点合并为单Code节点

### 任务描述
将 Python `EHuntClient` 类的逻辑转换为 JavaScript，并把 eHunt-Token自动刷新工作流中原来分散在 9 个节点的登录流程合并为 1 个 Code 节点。

### 完成事项
- [x] 编写独立的 JS 测试脚本 (`/tmp/ehunt_login_test_v4.js`)，在 n8n 容器中通过 `fetch()` API 完整验证登录流程
- [x] 测试通过: 解密凭证 → GET CSRF → POST 登录 → 跟随 redirect → 提取 JWT → API 验证 (code=100000)
- [x] 将 9 个旧节点合并为 1 个 "eHunt登录刷新" Code 节点 + 1 个 "登录验证通过?" IF 节点
- [x] 保留必要的下游节点: 解析JWT过期时间、保存新Token、清理旧Token、登录失败通知等
- [x] 通过 PostgreSQL 直接更新 workflow_entity 表应用变更
- [x] 重启 n8n 确认工作流正常激活

### 移除的节点 (9个)
| 旧节点 | 类型 | 说明 |
|--------|------|------|
| 整理凭证 | Code | 解密逻辑已合并 |
| 获取CSRF Token | HTTP Request | fetch() 替代 |
| 提取CSRF Token | Code | 合并到主节点 |
| 表单登录eHunt | HTTP Request | fetch() 替代 |
| 归一化登录响应 | Code | 不再需要 |
| 提取Token从Cookie | Code | 合并到主节点 |
| 登录成功? | IF | 改用新的 IF 节点 |
| 登录后验证 | Code | API 验证已合并 |
| 登录后验证通过? | IF | 改用新的 IF 节点 |

### 新增的节点 (2个)
| 新节点 | 类型 | 说明 |
|--------|------|------|
| eHunt登录刷新 | Code | 完整登录流程: 解密 → CSRF → POST → JWT → 验证 |
| 登录验证通过? | IF | auth_verified=true → 解析JWT, false → 失败通知 |

### 节点数量变化
- 旧: 23 个节点
- 新: 16 个节点 (减少 7 个)

### 工作流连接图 (精简后)
```
Schedule Trigger → 读取当前Token → 检查Token有效性 → 需要刷新?
  ├─ false → Token正常-跳过 → NoOp-完成
  └─ true → 读取eHunt登录凭证(Postgres) → eHunt登录刷新(Code)
       → 登录验证通过?(IF)
        ├─ true → 解析JWT过期时间 → 保存新Token → 构建刷新成功结果 → 清理旧Token → 输出刷新成功结果 → NoOp-完成
        └─ false → 登录失败通知 → NoOp-完成
```

---

## 2026-03-28 全工作流健康检查与修复

### 检查范围
对 eHunt AI 选品项目全部 17 个工作流进行逐个检查，包括节点配置、连接图、SQL 语法、JS 代码、安全性审查。

### 发现并修复的问题

| # | 工作流 | 问题 | 严重性 | 状态 |
|---|--------|------|--------|------|
| 1 | 竞品动态监控 (FYmjFnswMq989EVi) | splitInBatches 循环回路断裂 | 高 | 已修复 |
| 2 | 配额每日重置 (kirP8eODCFZoBfQb) | cron 时区错误 UTC→北京 | 中 | 已修复 |
| 3 | eHunt-Token自动刷新 (D0o0Q952jalrewzd) | 新合并节点验证 | 中 | 已验证通过 |
| 4 | SOP-步骤2-关键词挖掘 (ZWmD4UFMsnz3vQdr) | Update Execution Log SQL 损坏 | 高 | 已修复 |
| 5 | SOP-步骤1-自动挑类目 (iOL9f9tbQJNKkFBP) | Update Execution Log SQL 损坏 | 高 | 已修复 |
| 6 | SOP-步骤5to9-深度分析 (36mx5obPYV67EsJP) | 5个保存节点SQL注入审查 | 中 | 审查确认安全 |

### 修复后状态
- 全部 17 个工作流重启后正常激活
- 无报错

### 工作方式
- 使用团队模式: fixer (修复) + reviewer (审查) 并行工作
- 3个 agent 并行检查，2个 agent 并行修复和审查

---

## 2026-03-28 eHunt Pro 配额利用率优化

### 背景
用户提供了 eHunt Pro 会员每日 API 配额限制，要求最大化利用率。分析发现：
- **原利用率**: 1-6% (最好的一天仅 204/6200 次调用)
- **8/15 个 API 脚本从未使用**: 03, 05, 06, 09, 11, 13, 14, 15 (共 3100 次/日浪费)
- **Step5-9 的 8 个 eHunt API 调用完全绕过配额追踪系统**

### Phase 1: 数据采集深度扩展（改参数，不加节点）

#### 1.1 Step1 多页抓取 (iOL9f9tbQJNKkFBP)
- 4 个 Code 节点（Category Ranking, Product Ranking, Bestseller, Shop Ranking）从单页改为 5 页循环
- 4 个配额扣减节点从 `+1` 改为 `+5`
- 效果: 4→20 次 API 调用，数据从 50→250 条

#### 1.2 Step2 关键词扩展 (ZWmD4UFMsnz3vQdr)
- "Parse AI Keywords" 节点: `slice(0, 100)` → `slice(0, 200)`
- "Get Category Keywords from eHunt" 节点: 单页改为 3 页循环
- 效果: 每类目从 100→200 关键词

#### 1.3 Step3 蓝海评分扩展 (UM24AFvqv9CEnIK8)
- "读取待评分关键词" SQL: `LIMIT 100` → `LIMIT 200`
- "每批20个关键词" splitInBatches: `batchSize: 20` → `50`
- 效果: 400→800 次 API 调用

#### 1.4 Step4 反查产品扩展 (UzmEtfTxsLI36NWc)
- "读取A+B级关键词" SQL: `LIMIT 80` → `LIMIT 150`
- 效果: 70→450 次 API 调用

### Phase 2: 配额追踪修复

#### 2.1 数据库迁移
- 新增 16 个 quota_config 条目（从 15→31 个脚本）
- 每日总配额: 6200→12400 次
- 覆盖所有 Step5-9 使用的 API 端点

#### 2.2 Step5-9 配额追踪 (36mx5obPYV67EsJP)
- 移除 16 个孤立配额节点（创建但未连接）
- 在 8 个 API Code 节点内嵌静态数据追踪计数器
- 新增 "批量扣减配额" Postgres 节点，连接在"合并5步分析结果"之后
- 追踪脚本映射:
  - eHunt-Listing对比 → `listing_optimize` (100/日)
  - eHunt-店铺搜索 → `06_shop_search` (200/日)
  - eHunt-店铺排行 → `12_shop_ranking` (200/日)
  - eHunt-广告搜索 → `13_ads_search` (500/日)
  - eHunt-广告排行 → `14_ads_chart` (500/日)
  - eHunt-下架产品 → `15_delisted_products` (500/日)
  - eHunt-下架店铺 → `16_delisted_shops` (200/日)
  - eHunt-Amazon手工品 → `17_amazon_handmade` (500/日)

#### 2.3 Step3 配额脚本名修复
- 修复: `14_google_trends` → `13_ads_search` (广告搜索用错了脚本名)

#### 2.4 竞品动态监控配额追踪 (FYmjFnswMq989EVi)
- 新增 "Track Quota Usage" + "Deduct Monitoring Quota" 节点
- 连接链: 生成变化摘要 → Track Quota → Deduct → 发送告警通知

### Phase 3: 激活未使用脚本

#### 3.1 Step1 新增 30 天类目分析 (script 03)
- 新增 7 个节点: Check Quota 03 → IF → Read Token → Call Category Analytics 30d → Deduct 03 → Merge With Branch 5
- 使用 `/ecommerce/top-list/v1_0/get-list?time=30` 获取长期趋势数据
- 与现有 7 天数据互补，提供更全面的类目趋势分析

#### 3.2 Step2 新增关键词搜索追踪 (script 09)
- "Get Keyword Library Data" 节点添加 `09_keyword_search` 追踪

### 预估改善

| 指标 | 优化前 | Phase 1+2 后 | 含 Phase 3 |
|------|--------|-------------|-----------|
| 日 API 调用 | 40-200 | 1300-1600 | 1800-2600 |
| 利用率 | 1-6% | 21-25% | 29-42% |
| 追踪脚本 | 15/31 | 19/31 | 19/31 |
| 盲区 API | 8 个 | 0 个 | 0 个 |

### 修改工作流汇总

| 工作流 | 修改类型 | 节点变化 |
|--------|---------|---------|
| SOP-步骤1 | 多页+新分支+配额扣减 | 新增 7 节点，修改 8 节点 |
| SOP-步骤2 | 关键词上限+分页+script09 | 修改 3 节点 |
| SOP-步骤3 | LIMIT+batchSize+脚本名修复 | 修改 4 节点 |
| SOP-步骤4 | LIMIT 扩大 | 修改 1 节点 |
| SOP-步骤5to9 | 内嵌配额追踪+批量扣减 | 删 16，增 1，改 9 节点 |
| 竞品动态监控 | 配额追踪 | 新增 2 节点 |
| quota_config | DB 迁移 | 新增 16 条 |

---

## 2026-03-28 全面健康检查（配额优化后）

### 检查范围
3 个审查 agent 并行检查全部 17 个工作流：连接图完整性、节点引用有效性、JS 语法、SQL 语法、配额节点正确性、孤立节点。

### 发现并修复的问题

| # | 工作流 | 问题 | 严重性 | 状态 |
|---|--------|------|--------|------|
| 1 | SOP-主调度器 | `触发成功通知`/`发送错误通知` 指向不存在的工作流 `riphnAXbKOz5XKsx` | **严重** | 已修复→`PV4iVgFLPpd7IdwS` |
| 2 | SOP-步骤3-蓝海评分 | 配额节点名 `脚本14` 未完全改为 `脚本13`（连接 source key 残留） | 中 | 已修复 |
| 3 | SOP-步骤3-蓝海评分 | IF 节点 `配额足够-脚本11?` 缺少 false 输出分支 | 中 | 已修复→`等待所有API完成` |
| 4 | SOP-步骤3-蓝海评分 | IF 节点 `配额足够-脚本14?` 缺少 false 输出分支（已更名为脚本13） | 中 | 已修复→`等待所有API完成` |
| 5 | 竞品动态监控 | 2 个孤立旧配额节点残留 (`配额检查-02_product_ranking`, `配额扣减-02_product_ranking`) | 低 | 已清除 |

### 最终验证结果（19 项检查全部通过）

- ✓ 主调度器 10 个子工作流 ID 全部正确
- ✓ Step3 全部 4 个配额 IF 节点有完整 true/false 输出
- ✓ Step5-9 合并→批量扣减→保存链完整
- ✓ 竞品监控 splitInBatches 回路正常
- ✓ Token 刷新链完整
- ✓ 17/17 工作流全部活跃
- ✓ quota_config 31 个脚本，今日 31 条 usage 记录
- ✓ 8 个核心数据表存在
- ✓ 全部工作流无孤立节点
- ✓ 项目可以正常运行
