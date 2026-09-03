# GitHub Copilot 用量数据获取指南

> 面向新任 GitHub Copilot 管理员：帮你快速判断 **「我想要的数据在哪里、用哪个 API 取、是什么格式」**，以及 **如何集成到自己的管理控制台**。
>
> 本文所有 UI 路径、API 端点、字段名均基于 GitHub REST API 版本 `2026-03-10`（截至 2026-09-03 仍是最新版本）。全文于 2026-09-03 对照 github/docs 仓库中的 OpenAPI 数据与参考文档逐条核对。GitHub 的 Copilot 相关 API 变动频繁，落地前建议再次对照官方文档。

## 速览

**一句话**：Copilot 的管理数据分散在 **三条互不相通的链路** 里，口径各不相同，**跨链路的数字不能互相对账**。先确定你的问题属于哪一条，再去对应链路取数。

| 链路 | 回答什么问题 | API 路径特征 | 口径 |
|------|------------|-------------|------|
| **A 席位** | 谁有席位、最近何时用过、一共买了多少 | `/copilot/billing/seats` | 许可证（官方称其为 license 与 seat 信息的 source of truth） |
| **B 用量指标** | 谁用了哪些功能 / 模型 / 语言，生成和接受了多少代码，PR 产出如何 | `/copilot/metrics/reports/*` | 遥测（需开策略，受 IDE telemetry 影响） |
| **C 计费** | 谁、什么时间、用哪个模型消耗了多少 AI Credits，烧了多少 token | `/settings/billing/*` | 账单（**唯一的账单口径**） |

### 快速导航

| 章节 | 你想回答的问题 | 链路 | UI 入口 | REST API | 格式 |
|------|--------------|:----:|---------|----------|------|
| [一、席位](#一席位谁被分配了-copilot谁长期未使用买了多少) | 谁被分配了 Copilot？谁长期未使用？一共买了多少席位？ | A | Org → Settings → Copilot → Access<br>Ent → Billing and licensing → Licensing | `/copilot/billing/seats` 系列 | JSON / CSV |
| [二、用量指标](#二用量指标谁在用什么功能模型和语言生成了多少代码) | 谁在用 Chat、Agent、CLI？用了哪些模型和语言？生成和接受了多少代码？PR 产出如何？ | B | Ent → Insights → Copilot usage / Code generation / Copilot impact | `/copilot/metrics/reports/*` | JSON → NDJSON |
| [三、计费](#三计费谁什么时间用哪个模型消耗了多少-ai-credits) | 谁、什么时间、用哪个模型消耗了多少 AI Credits？token 明细？账单多少？ | C | Ent → Billing and licensing → Usage → AI usage | `/settings/billing/ai_credit/usage`<br>`/settings/billing/reports` | JSON / CSV |

> 缩写：`Ent` = Enterprise 层级，`Org` = Organization 层级。
>
> 本文结构：[开始之前](#开始之前策略角色和-token) → 一至三节 → [常见问题](#常见问题) → [补充端点](#补充其他可能用得上的端点) → [官方文档索引](#官方文档索引)。每节按 **用途 → UI → API → 返回数据 → 限制** 展开。

---

## 开始之前：策略、角色和 Token

不配好这三样，后面的 API 会报 403 或返回空。

### 1. 打开 usage metrics 策略（链路 B 必需）

```
Enterprise → AI controls（不是 Policies，GitHub 已重命名该导航）
          → 侧边栏 Copilot
          → "Copilot usage metrics" → Enabled everywhere
```

Org 层级入口：`Org → Settings → Copilot → Policies`。不开这条策略，链路 B 的看板和 API 全部无数据。

### 2. 角色要求

| 数据 | 需要的角色 |
|------|-----------|
| 企业席位 / 活动报告 | Enterprise owner 或 Billing manager |
| 组织席位 / 活动报告 | Organization owner |
| 企业用量指标 | Enterprise owner、Billing manager，或拥有企业自定义角色权限 **"View Enterprise Copilot Metrics"** 的用户 |
| 组织用量指标 | Organization owner，或拥有 **"View Organization Copilot Metrics"** 权限的用户 |
| 计费用量 | Enterprise / Organization administrator 或 Billing manager |

### 3. Token：用 Classic PAT

给自建控制台准备一个专用 **Classic PAT**，勾选 `manage_billing:copilot` + `read:enterprise` + `read:org`，可覆盖本文全部端点。

**计费端点（链路 C）和企业级席位端点不支持 Fine-grained PAT**，用了会报 403。

### 4. 通用请求头

```bash
curl -L \
  -H "Authorization: Bearer <YOUR_CLASSIC_PAT>" \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2026-03-10" \
  https://api.github.com/...
```

---

## 一、席位：谁被分配了 Copilot，谁长期未使用，买了多少

**用途**：许可证盘点、回收长期不活跃的席位、查看企业席位总数与当前账期席位费用。

### UI：Access、Licensing 与活动报告

**组织级**

```
Organization → Settings → 侧边栏 Copilot → Access
```
- 页面顶部显示已分配席位数和预估月度费用
- "Access management" 列表可按 **最后使用时间排序**
- 点击 **Get usage report → Get activity report** 下载 CSV

**企业级**

```
Enterprise → Billing and licensing → Licensing
```
- 页面底部的 **Copilot** 标签页：已分配席位数、当前账期席位总花费；旁边的 **Get activity report** 下载 CSV
- "Enterprise Cloud" 区域点 **More details** 看许可证明细，点 **Download CSV report** 下载
- 点 **Manage** 看许可证变更历史（public preview，每日快照，含操作人和生效日期）

### API：席位查询端点

```http
# 企业：列出所有席位（跨所有 Org 和 Enterprise Team，同一用户只计一次 total_seats）
GET /enterprises/{enterprise}/copilot/billing/seats?page=1&per_page=100

# 企业：查单个用户的席位详情（返回 seats 数组：该用户在每个授予席位的 Org / Team 各一条）
GET /enterprises/{enterprise}/members/{username}/copilot

# 组织：席位汇总和策略设置（注意：不是费用）
GET /orgs/{org}/copilot/billing

# 组织：列出所有席位
GET /orgs/{org}/copilot/billing/seats?page=1&per_page=100

# 组织：查单个用户的席位详情（返回单个 seat 对象）
GET /orgs/{org}/members/{username}/copilot
```

> ⚠️ `GET /orgs/{org}/copilot/billing/seats/{username}` **不存在**。查单用户请用 `/orgs/{org}/members/{username}/copilot`。

### 返回数据：席位 JSON 与活动报告 CSV

`GET /enterprises/{enterprise}/copilot/billing/seats`：

```json
{
  "total_seats": 2,
  "seats": [
    {
      "created_at": "2021-08-03T18:00:00-06:00",
      "updated_at": "2021-09-23T15:00:00-06:00",
      "pending_cancellation_date": null,
      "last_activity_at": "2021-10-14T00:53:32-06:00",
      "last_activity_editor": "vscode/1.77.3/copilot/1.86.82",
      "last_authenticated_at": "2021-10-14T00:53:32-06:00",
      "plan_type": "business",
      "assignee": { "login": "octocat", "id": 1, "...": "..." },
      "assigning_team": { "slug": "justice-league", "...": "..." }
    }
  ]
}
```

`GET /orgs/{org}/copilot/billing` 返回席位汇总 + 策略：

```json
{
  "seat_breakdown": {
    "total": 12,
    "added_this_cycle": 9,
    "pending_invitation": 0,
    "pending_cancellation": 0,
    "active_this_cycle": 12,
    "inactive_this_cycle": 11
  },
  "seat_management_setting": "assign_selected",
  "ide_chat": "enabled",
  "platform_chat": "enabled",
  "cli": "enabled",
  "public_code_suggestions": "block",
  "plan_type": "business"
}
```

**活动报告 CSV**（UI 下载）：

| 字段 | 说明 |
|------|------|
| `report_time` | 报告生成的 UTC 时间戳 |
| `login` | GitHub 用户名 |
| `last_authenticated_at` | 最近一次认证的 UTC 时间 |
| `last_activity_at` | 最近一次 Copilot 交互的 UTC 时间 |
| `last_surface_used` | 最近使用的界面：IDE 名称+版本（如 `VS Code 1.89.1`）、GitHub.com 功能名（如 `Copilot Chat`）或 `Unspecified` |

### 限制：90 天保留期与 24 小时延迟

- **`last_activity_at` 延迟最长 24 小时**；用户必须在 IDE 中开启 telemetry，IDE 内的活动才会被记录。
- **保留期 90 天**，不可修改。超过 90 天无活动，`last_activity_at` 会被置为 `nil`，所以"长期未使用"最多只能识别到 90 天。要看更久，必须自己每天存快照。
- CSV 报告每 30 分钟自动刷新一次。
- 非 GA 功能（如 Copilot Spaces、Copilot Spark）的使用不计入活动报告。
- 该系列端点标记为 **public preview**，字段可能变化。

---

## 二、用量指标：谁在用什么功能、模型和语言，生成了多少代码

**用途**：功能采纳率分析、模型治理、IDE 分布、团队对比、代码贡献量（AI 生成占比、Agent vs 人工）、PR 产出。这些问题 **共用同一套 Usage Metrics Reports API 和同一份 NDJSON 记录**，区别只在读哪些字段。

**前提**：已按[开始之前](#1-打开-usage-metrics-策略链路-b-必需)开启 usage metrics 策略。

### UI：Insights 下的三个看板

```
Enterprise → Insights → 侧边栏
```

| 看板 | 回答什么 |
|------|---------|
| **Copilot usage** | 28 天趋势：活跃用户、功能 / 模型 / 语言 / IDE 分布 |
| **Code generation** | 过去 28 天 AI 改动的总行数、Agent 贡献占比、每日增删行数、用户发起 vs Agent 发起（可按模型、语言拆分） |
| **Copilot impact** | 采纳阶段（Adoption Cohort）分布及其与 PR 产出的关系 |

看板支持直接 **导出 NDJSON**，可以配合 Copilot Chat 做分析。

> Org 层级也有同样的看板（2025-12-12 起提供），但官方 how-to 只给出了企业级导航路径。Org 管理员找不到入口时，用 API 最稳妥。

### API：Usage Metrics Reports（12 个端点）

这是一套 **"先取下载链接，再下载 NDJSON 文件"** 的两段式 API。REST 响应里 **不包含明细数据**，只包含限时签名 URL。

**企业级（6 个）**

```http
GET /enterprises/{enterprise}/copilot/metrics/reports/enterprise-1-day?day=YYYY-MM-DD
GET /enterprises/{enterprise}/copilot/metrics/reports/enterprise-28-day/latest
GET /enterprises/{enterprise}/copilot/metrics/reports/users-1-day?day=YYYY-MM-DD
GET /enterprises/{enterprise}/copilot/metrics/reports/users-28-day/latest
GET /enterprises/{enterprise}/copilot/metrics/reports/repos-1-day?day=YYYY-MM-DD
GET /enterprises/{enterprise}/copilot/metrics/reports/user-teams-1-day?day=YYYY-MM-DD
```

**组织级（6 个，形状完全一致）**

```http
GET /orgs/{org}/copilot/metrics/reports/organization-1-day?day=YYYY-MM-DD
GET /orgs/{org}/copilot/metrics/reports/organization-28-day/latest
GET /orgs/{org}/copilot/metrics/reports/users-1-day?day=YYYY-MM-DD
GET /orgs/{org}/copilot/metrics/reports/users-28-day/latest
GET /orgs/{org}/copilot/metrics/reports/repos-1-day?day=YYYY-MM-DD
GET /orgs/{org}/copilot/metrics/reports/user-teams-1-day?day=YYYY-MM-DD
```

**怎么选 report 类型：**

| Report | 粒度 | 说明 |
|--------|------|------|
| `enterprise-1-day` / `organization-1-day` | 每天一条聚合记录 | 含活跃用户数、代码生成汇总、`pull_requests`、采纳阶段分布 |
| `enterprise-28-day` / `organization-28-day` | 28 天窗口 | 外层是窗口信息，`day_totals` 数组里是每天的聚合记录 |
| `users-1-day` / `users-28-day` | 每用户一条 | 含 `user_id`、`user_login`、`ai_credits_used`、`used_*` 标志位、`ai_adoption_phase`，以及该用户的功能 / 模型 / 语言 breakdown 和代码行数；**不含** 活跃用户数、`pull_requests` |
| `repos-1-day` | 每仓库一条 | 仅含 PR 生命周期数据；当天无 PR 活动的仓库不出现 |
| `user-teams-1-day` | 每 (用户, 团队) 一条 | 用于自行构造团队级指标 |

- `day` 参数 **必填**（`/latest` 端点除外）。
- 没有 `repos-28-day` 和 `user-teams-28-day`，只有 1-day 版本。
- **没有预聚合的团队级报表**。团队指标需要自己把 `user-teams-1-day` 和 `users-1-day` 按 `user_id` + `day` join 起来，再按 `team_id` 聚合。**不要用 `users-28-day` 去 join 单日的 `user-teams-1-day`**，会把 28 天的活动全部错误归属到 join 当天的团队。

### 返回数据：两段式下载与 NDJSON

**第一步：API 响应是 JSON，只有下载链接**

```json
{
  "download_links": [
    "https://example.com/copilot-usage-report-1.ndjson",
    "https://example.com/copilot-usage-report-2.ndjson"
  ],
  "report_day": "2026-07-01"
}
```

28-day 端点返回：

```json
{
  "download_links": ["https://..."],
  "report_start_day": "2026-07-01",
  "report_end_day": "2026-07-28"
}
```

**第二步：下载 NDJSON 文件**（每行一条 JSON 记录）。

> `download_links` 是 **限时签名 URL**，官方只说"有过期时间"，**没有公布具体时长**。拿到响应后 **立即下载**，不要把 URL 存起来隔天再用。

### 主要字段

下面按"你想回答的问题"分组。字段完整定义见官方[用量指标字段定义](https://docs.github.com/en/copilot/reference/copilot-usage-metrics/copilot-usage-metrics)。

#### 谁在用：活跃用户与 per-user 标志位

**活跃用户计数**（聚合报表）

```
daily_active_users / weekly_active_users / monthly_active_users
monthly_active_chat_users / monthly_active_agent_users
daily|weekly|monthly_active_copilot_cloud_agent_users
daily|weekly|monthly_active_copilot_code_review_users
daily|weekly|monthly_passive_copilot_code_review_users
daily_active_cli_users          # 独立于 IDE 口径
daily_active_copilot_app_users  # 仅企业级报表
```

**per-user 报表的功能使用标志位**

```
used_chat, used_agent, used_cli, used_copilot_app
used_copilot_coding_agent / used_copilot_cloud_agent   # 同一个值，两个名字并存（向后兼容）
used_copilot_code_review_active                        # 主动请求或采纳了 review 建议
used_copilot_code_review_passive                       # 被自动指派 review 但没互动
```

> per-user 报表里还有 `ai_credits_used`（该用户在报告期内消耗的 AI Credits 总数，不按功能 / 模型拆分）。官方明确它 **用于消费趋势分析，不是账单数字**；要对账请用[第三节](#三计费谁什么时间用哪个模型消耗了多少-ai-credits)。

#### 用什么：功能、模型、语言、IDE（breakdown 数组）

| 字段 | 维度 | 说明 |
|------|------|------|
| `totals_by_ide[]` | `ide` | 按 IDE 拆分；per-user 报表还含 `last_known_ide_version`、`last_known_plugin_version` |
| `totals_by_feature[]` | `feature` | 按功能拆分 |
| `totals_by_language_feature[]` | `language`, `feature` | 语言 × 功能 |
| `totals_by_language_model[]` | `language`, `model` | 语言 × 模型（仅 chat，不含补全） |
| `totals_by_model_feature[]` | `model`, `feature` | 模型 × 功能（仅 chat） |

- `feature` 的取值：`code_completion`、`chat_inline`、`chat_panel_ask_mode`、`chat_panel_edit_mode`、`chat_panel_agent_mode`、`chat_panel_plan_mode`、`chat_panel_custom_mode`、`chat_panel_unknown_mode`、`agent_edit`、`copilot_cli`、`copilot_app`、`others`。
- `ide` 的取值示例：`vscode`、`visualstudio`、`intellij`、`eclipse`、`xcode`、`neovim`、`vim`、`emacs`、`zed`。
- `model` 的取值：具体模型 ID（如 `gpt-5.4`、`claude-sonnet-4.6`）、`auto`（自动选模型且未归因到具体模型）、`unknown`、`others`。

每个 breakdown 元素里都带同一组度量：`user_initiated_interaction_count`、`code_generation_activity_count`、`code_acceptance_activity_count`、`loc_*_sum`（定义见下文[代码生成](#生成了多少代码代码生成与接受)）。

#### CLI、Copilot App 与第三方 Agent

**CLI 专属指标（`totals_by_cli` 对象）**

```
totals_by_cli.session_count            # 当天的 CLI 会话数
totals_by_cli.request_count            # 请求数（含 agentic 自动跟进调用）
totals_by_cli.prompt_count             # 用户 prompt 数
totals_by_cli.token_usage.output_tokens_sum
totals_by_cli.token_usage.prompt_tokens_sum
totals_by_cli.token_usage.avg_tokens_per_request
totals_by_cli.last_known_cli_version   # 仅 per-user 报表
```

> **CLI 用量是完全独立统计的**，不计入 `totals_by_ide`、`totals_by_feature`，也不计入 IDE 口径的活跃用户数。统计 CLI 活跃人数请用 `daily_active_cli_users`。
>
> `totals_by_copilot_app` 的结构与 CLI 相同，但 Copilot App 的编码活动 **会** 同时体现在 `totals_by_feature` 等数组里（`feature = copilot_app`）。

**第三方 Agent（`totals_by_3rd_party_agent[]`）**

```
agent_name, agent_id, user_initiated_interaction_count, session_count
```
用 `agent_id` 做分组键（`agent_name` 可能变）；这里的 `user_initiated_interaction_count` 统计的是 **Agent 任务启动次数**，不要和顶层同名字段混算。

#### 生成了多少代码：代码生成与接受

**口径**：所有数值都来自 **IDE 内** 实际增删的代码行——补全、inline chat、chat panel 里应用的代码块、agent / edit 模式直接写入文件的改动都算；IDE 之外的产出（例如 Copilot cloud agent 在 GitHub.com 上提交的 PR）不在这里，看下文的 `pull_requests`。

| 字段 | 含义 |
|------|------|
| `code_generation_activity_count` | Copilot 产出事件数。**包含注释和 docstring**；一个 prompt 产生多个代码块算多次 |
| `code_acceptance_activity_count` | 用户接受的次数。计入 apply to file、insert at cursor、insert into terminal、Copy 按钮；**不计** 操作系统剪贴板复制（Ctrl+C）。每次接受动作只 +1，无论原始 prompt 生成了几个代码块 |
| `loc_suggested_to_add_sum` | Copilot 建议新增的代码行数（补全、inline chat、chat panel 等，**不含 agent edit**） |
| `loc_suggested_to_delete_sum` | 建议删除的行数（**功能规划中，目前基本为 0**） |
| `loc_added_sum` | 实际写入编辑器的新增行数（接受的补全、应用的代码块、agent 和 edit 模式） |
| `loc_deleted_sum` | 从编辑器删除的行数（目前来自 agent edit） |
| `user_initiated_interaction_count` | 显式发给 Copilot 的 prompt 数。**不含** 打开 chat 面板、切换模式、快捷键唤起 UI、改配置 |

- **接受率**：`code_acceptance_activity_count ÷ code_generation_activity_count`（看板和 API 用同一公式，只是四舍五入可能不同）。
- **Agent 贡献占比**：用 `feature = agent_edit` 那一行的 `loc_added_sum` 除以总 `loc_added_sum`。
- ⚠️ `code_generation_activity_count` 和 `user_initiated_interaction_count` **不能直接比较**，一个 prompt 可以产生多次生成。
- ⚠️ `agent_edit` 不参与"建议类"指标，可能不填充 `user_initiated_interaction_count`、`loc_suggested_to_add_sum` 之类的字段。

#### PR 产出：`pull_requests` 对象

出现在聚合报表和 `repos-1-day` 报表中（per-user 报表没有）：

```
total_created, total_reviewed, total_merged, median_minutes_to_merge
total_suggestions, total_applied_suggestions
total_created_by_copilot          # Copilot cloud agent 创建的 PR
total_reviewed_by_copilot         # Copilot code review 评审的 PR
total_merged_created_by_copilot
total_merged_reviewed_by_copilot
median_minutes_to_merge_copilot_authored
median_minutes_to_merge_copilot_reviewed
total_copilot_suggestions, total_copilot_applied_suggestions
copilot_suggestions_by_comment_type[]   # 按 security / bug_risk 等类型拆分
```

#### 用到什么程度：采纳阶段（AI Adoption Phase）

判定依据是 **功能级的 engagement**：在滚动 28 天窗口内，某功能 **至少 2 个不同的活跃日** 有活动即算 engaged。从高到低评估，取用户满足的最高阶段。

| 阶段 | Engagement 条件 |
|------|----------------|
| `No Cohort`（看板显示为 Passive users） | 没有 engaged 任何可计阶段的功能。**不等于不活跃**：纯对话式使用（记为 `chat_panel_agent_mode` 等）、从聊天回复里复制代码，都不产生下面的信号 |
| `Phase 1: Code first` | engaged `code_completion` 或 `agent_edit`（`feature` 维度值） |
| `Phase 2: Agent first` | engaged **恰好一个** GitHub 侧 agent surface：Copilot CLI（`copilot_cli`）、Copilot cloud agent（`used_copilot_cloud_agent`）、Copilot code review（`used_copilot_code_review_active` 或 `_passive`，主动被动合并算一个 surface） |
| `Phase 3: Multi-agent` | engaged Copilot App（`copilot_app`），或 ≥2 个上述 agent surface |

- `used_chat`、`used_agent`、`code_acceptance_activity_count` **都不参与判定**：在 agent 模式里只提问、不让 Copilot 改文件，记的是 `chat_panel_agent_mode` 而不是 `agent_edit`，用户会一直留在 No Cohort。
- 到达 Phase 2 / 3 不要求先满足 Phase 1。
- 阶段每天基于滚动 28 天窗口重算，**用户的阶段会天天变**，这是预期行为。

### 限制：滞后 3 天、只留 1 年、小团队被排除、telemetry 关闭

- **数据最长滞后 3 个完整 UTC 日**。定时任务请拉 **T-4 天** 的 `day`，否则经常拿到空报表或不完整数据。
- **1-day 报表从 2025-10-10 起提供，只能取最近 1 年的数据**；组织级报表从 2025-12-12 起提供。要保留更久请自己入仓。
- **当天席位用户少于 5 人的团队不会出现在 `user-teams-1-day`**。把团队数据加总会小于企业 / 组织总数，差额就是只属于小团队的用户。
- **用户在 IDE 里关闭 telemetry** 后，IDE 明细（per-IDE、per-feature、LoC）全部缺失，但服务端遥测仍可能把该用户计入活跃用户数，出现"有活跃用户但没有明细"的现象。
- **Org 与 Enterprise 两级数字不能直接比较**：组织级指标按 **组织成员关系** 归属（用户在哪些 Org 就算进哪些 Org，与席位在哪个 Org 无关），同一用户会出现在多个 Org 看板里，但在企业级只算一次。

---

## 三、计费：谁、什么时间、用哪个模型消耗了多少 AI Credits

**用途**：成本分摊、预算告警、对账。**这是唯一的账单口径数据源。** AI Credits 与预算的计费逻辑见 [预算与计费说明](budget-config.md)，本节只讲怎么把数据取出来。

> **Premium Requests 已是遗留口径。** 2026-06-01 起 Copilot Business / Enterprise 改按 AI Credits 计费，Premium Requests 不再产生新数据。`.../settings/billing/premium_request/usage` 端点和 `report_type = premium_request` 仍可查 6 月 1 日之前的历史（保留 24 个月），本文不再展开。

### UI：AI usage 视图

```
Enterprise → Billing and licensing → Usage → AI usage
```

- **AI usage** 视图：Copilot AI Credits 专用——总消耗、重度用户、模型花费分布、各 Org 采纳情况；支持按维度筛选、分组、选时间范围，图表右上角 **⋯** 可直接下载图表数据
- 同页面的 **Metered usage** 视图是所有计费产品（Actions、Copilot……）的总账；账单概览在 `Billing and licensing → Overview`

> ⚠️ **Org owner 在 AI usage 视图里看不到 per-user 数据**，只有 Enterprise owner / Billing manager 可以按用户筛选。Org 层级要看每用户消耗，必须下载报告（见 [3.2](#32-明细ai-usage-report-csv)）。

### 3.0 两种数据：汇总 vs 明细

计费数据有两种形态。**汇总回答「多少」，明细回答「谁在什么时候用哪个模型花了多少、烧了多少 token」。** 先按你要的维度对号：

| 你要的维度 | ① **汇总**：JSON 即时查询 | ② **明细**：AI usage report CSV |
|---|---|---|
| **谁**（用户） | ❌ 没有用户列。只能用 `user=` 参数 **一次查一个人** | ✅ `username` 列 |
| **什么时间** | ❌ 没有日期列。`year` / `month` / `day` 参数决定统计区间，结果是该区间的总和 | ✅ `date` 列，按天一行 |
| **哪个模型** | ✅ `model` 字段（每个模型一条） | ✅ `model` 列 |
| **AI Credits 消耗** | ✅ `netQuantity`（credits）、`netAmount`（美元） | ✅ `quantity`（credits）、`net_amount`（美元） |
| **token 情况** | ❌ 没有 | ✅ `input` / `output` / `cache_read` / `cache_write` |
| 组织 / cost center | 用 `organization=` / `cost_center_id=` 参数过滤（企业级） | ✅ `organization` / `cost_center_name` 列 |
| 怎么拿 | 一次 `GET`，同步返回 | UI 点 **Get usage report** 收邮件；或 API `POST` 建任务 → 轮询 → 下载 |
| 时间范围 | 最细到天，可查过去 24 个月 | 一次最长 31 天 |
| 企业 / 组织 | 两级都有 | UI 两级都有；**导出 API 仅企业级** |
| 适合 | 看板数字、预算告警、快速查某人某月总量 | 成本分摊、逐用户账单、BI 入仓 |

**选型**：
- 「这个月一共花了多少」「某个人 / 某个模型在这个月/某天总共花了多少」→ ①
- 「全公司每个人、每次Agent任务分别花了多少，token 多少，用的什么模型」→ ②

### 3.1 汇总：JSON 即时查询

**端点**（组织级路径用 `/organizations/{org}/`，**不是** `/orgs/{org}/`）：

```http
# 企业级
GET /enterprises/{enterprise}/settings/billing/ai_credit/usage
    ?year=&month=&day=&organization=&user=&model=&product=&cost_center_id=

# 组织级（没有 organization 和 cost_center_id 参数）
GET /organizations/{org}/settings/billing/ai_credit/usage
    ?year=&month=&day=&user=&model=&product=
```

例：查某人 7 月的 AI Credits 消耗

```
GET /enterprises/{enterprise}/settings/billing/ai_credit/usage?year=2026&month=7&user=zhangsan
```

**响应**：`usageItems` 按 SKU × 模型各一条，没有用户和日期

```json
{
  "timePeriod": { "year": 2026, "month": 7 },
  "enterprise": "contoso",
  "usageItems": [
    {
      "product": "Copilot",
      "sku": "Copilot AI Credits",
      "model": "GPT-5",
      "unitType": "credits",
      "pricePerUnit": 0.01,
      "grossQuantity": 100,
      "grossAmount": 1,
      "discountQuantity": 0,
      "discountAmount": 0,
      "netQuantity": 100,
      "netAmount": 1
    }
  ]
}
```

**怎么读金额**：
- 1 AI Credit = **$0.01**（`pricePerUnit`）。
- `netAmount = grossAmount − discountAmount`：`grossAmount` 是按标价算的金额，`discountAmount` 是 GitHub 侧给的**折扣金额**（如套餐内含用量）。多数客户通过 Azure 计费，折扣在 Azure 侧体现，所以这里通常 `discountAmount = 0`、`netAmount = grossAmount`。

**限制与坑**：
1. 路径是 **`ai_credit`（单数）**，不是 `ai_credits`。
2. 组织级用 **`/organizations/{org}/`**，不是 `/orgs/{org}/`（后者下只剩一个无关的历史端点 `advanced-security`）。
3. **没有 `hour` 参数**，最细到 `day`；结果里也没有日期列，要做时间序列只能按天循环查，或改用 ②。
4. 只能查 **过去 24 个月**；不带时间参数时默认返回当年。

> 想看 **整个企业所有计费产品的总账**（Actions、Copilot 等一起），用同一形状的 `GET /enterprises/{enterprise}/settings/billing/usage/summary?year=&month=&day=&organization=&repository=&product=&sku=&cost_center_id=`（组织级 `/organizations/{org}/settings/billing/usage/summary`），响应里没有 `model`。

### 3.2 明细：AI usage report CSV

这是唯一能同时拿到 **用户 × 日期 × 模型 × AI Credits × token** 的数据。一行明细长这样（列名为官方定义，数值为示意）：

| `date` | `username` | `model` | `quantity` | `unit_type` | `net_amount` | `input` | `output` | `cache_read` | `cache_write` |
|---|---|---|---|---|---|---|---|---|---|
| 2026-07-15 | zhangsan | claude-sonnet-4.6 | 12.5 | credits | 0.125 | 18234 | 2210 | 9500 | 1200 |
| 2026-07-15 | zhangsan | gpt-5.4 | 3.2 | credits | 0.032 | 6100 | 840 | 0 | 0 |
| 2026-07-15 | lisi | claude-sonnet-4.6 | 41.0 | credits | 0.410 | 72000 | 9800 | 30500 | 4100 |

**全部列**（按 `date` + `model` + `username` 聚合，同一组合一行）：

| 分组 | 列 | 说明 |
|---|---|---|
| 谁 / 何时 / 什么 | `date` | 用量发生的日期（UTC） |
| | `username` | 归属用户 |
| | `model` | 模型 ID（如 `claude-sonnet-4.6`） |
| | `product` / `sku` / `unit_type` | 固定为 Copilot / AI Credits SKU / `credits` |
| 归属 | `organization` / `repository` / `cost_center_name` | 归属组织、仓库、cost center（如适用） |
| 金额 | `quantity` | 消耗的 AI Credits 数 |
| | `applied_cost_per_quantity` | 单价（$0.01 / credit） |
| | `gross_amount` / `discount_amount` / `net_amount` | 标价金额 / 折扣金额 / 应付金额（`net = gross − discount`） |
| Token | `input` / `output` / `cache_read` / `cache_write` | 该模型当天的输入、输出、缓存读、缓存写 token 数 |

**拿法 A：UI 邮件下载**（企业级、组织级都可以）

1. `Enterprise → Billing and licensing → Usage → AI usage`
2. 页面顶部点 **Get usage report** → 选日期范围（最长 31 天）→ **Email me the report**
3. 报告发到账号主邮箱，**下载链接 24 小时后过期**；**同一账号同时只能有一个待生成的报告**

**拿法 B：API 异步导出**（**仅企业级**；Org 管理员要拿逐用户 CSV 只能走拿法 A）

```http
POST /enterprises/{enterprise}/settings/billing/reports
{
  "report_type": "ai_credit",
  "start_date": "2026-07-01",
  "end_date": "2026-07-31",        // 可省略，默认今天（UTC）
  "send_email": false              // 可省略，默认 false
}

GET /enterprises/{enterprise}/settings/billing/reports               # 列出所有导出任务
GET /enterprises/{enterprise}/settings/billing/reports/{report_id}   # 查状态、取下载地址
```

轮询 `status` 直到 `completed`，再从 `download_urls` 下载 CSV：

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "report_type": "ai_credit",
  "start_date": "2026-07-01",
  "end_date": "2026-07-31",
  "status": "completed",
  "download_urls": ["https://github.com/enterprises/contoso/metered_exports/1234"],
  "created_at": "2026-08-01T10:30:00Z",
  "actor": "monalisa"
}
```

> **其他两种报告**：`report_type = summarized`（UI 名 Summarized usage report，最长 1 年）和 `detailed`（Detailed usage report，最长 31 天，多 `username`、`workflow_path` 列）是 **Metered usage** 页面上覆盖所有计费产品的总账报告，列定义与上表相同但 **没有 `model` 和 token 列**。只关心 Copilot 的话用 `ai_credit` 即可。

---

## 常见问题

| 现象 | 原因 | 详见 |
|------|------|------|
| 拿用量指标去对账单，对不上 | 链路 B 是遥测口径（含 per-user 的 `ai_credits_used`），链路 C 是账单口径，不能互相对账；账单只看第三节 | [速览](#速览) |
| 昨天的用量指标是空的 | 最长滞后 3 个完整 UTC 日，拉 T-4 才稳 | [二 · 限制](#限制滞后-3-天只留-1-年小团队被排除telemetry-关闭) |
| 有活跃用户但没有 IDE 明细 | 用户在 IDE 里关掉了 telemetry | [二 · 限制](#限制滞后-3-天只留-1-年小团队被排除telemetry-关闭) |
| 找不到某个团队 / 团队加总小于总数 | 当天席位用户少于 5 人的团队被 `user-teams` 报表排除 | [二 · 限制](#限制滞后-3-天只留-1-年小团队被排除telemetry-关闭) |
| Org 看板加总大于 Enterprise 总数 | 组织级按成员关系归属，一个用户会出现在多个 Org；企业级只算一次 | [二 · 限制](#限制滞后-3-天只留-1-年小团队被排除telemetry-关闭) |
| CLI 用量对不上 IDE 数字 | CLI 独立统计，不计入 IDE 口径的活跃用户和 feature 明细 | [二 · CLI](#clicopilot-app-与第三方-agent) |
| `download_links` 打不开 | 限时签名 URL，拿到后要立刻下载 | [二 · 返回数据](#返回数据两段式下载与-ndjson) |
| 用户的采纳阶段天天在变 | 基于滚动 28 天窗口每天重算，预期行为 | [二 · 采纳阶段](#用到什么程度采纳阶段ai-adoption-phase) |
| 用户天天用 Chat 却一直是 Passive user | 阶段只看 `code_completion` / `agent_edit` 等功能信号，纯对话不算 | [二 · 采纳阶段](#用到什么程度采纳阶段ai-adoption-phase) |
| Fine-grained PAT 报 403 | 计费端点和企业级席位端点只支持 Classic PAT | [开始之前 · Token](#3-token用-classic-pat) |
| 组织计费端点 404 | 路径是 `/organizations/{org}/settings/billing/...`，不是 `/orgs/{org}/` | [3.1](#31-汇总json-即时查询) |
| 汇总查询里没有用户名 / 日期 / token | JSON 即时查询是聚合结果；逐用户逐日逐模型 + token 只有 AI usage report CSV 有 | [3.0](#30-两种数据汇总-vs-明细) |
| 想查 90 天前谁没用过 Copilot | `last_activity_at` 只保留 90 天，必须自己每天存快照 | [一 · 限制](#限制90-天保留期与-24-小时延迟) |

---

## 补充：其他可能用得上的端点

```http
# Copilot Agent 会话活动记录（public preview，仅 EMU 企业 owner 可用，游标分页，每页最多 25 条）
GET /enterprises/{enterprise}/copilot/usage-records?phrase=&per_page=25&after=&before=&order=desc

# 席位分配 / 回收
POST   /orgs/{org}/copilot/billing/selected_users              # 分配
DELETE /orgs/{org}/copilot/billing/selected_users              # 回收（设置 pending_cancellation_date）
POST   /orgs/{org}/copilot/billing/selected_teams
DELETE /orgs/{org}/copilot/billing/selected_teams
POST   /enterprises/{enterprise}/copilot/billing/selected_users             # 企业级直接分配
DELETE /enterprises/{enterprise}/copilot/billing/selected_users
POST   /enterprises/{enterprise}/copilot/billing/selected_enterprise_teams
DELETE /enterprises/{enterprise}/copilot/billing/selected_enterprise_teams

# 按 cost center 查用量明细（JSON，天 × 组织 × 仓库；仅 enhanced billing platform；默认只返回未归属 cost center 的用量）
GET /enterprises/{enterprise}/settings/billing/usage?year=&month=&day=&cost_center_id=

# Cost Center（成本分摊，注意 ai_credit_pool_enabled 属性）
GET/POST /enterprises/{enterprise}/settings/billing/cost-centers

# 预算（budget_scope 支持 enterprise / organization / cost_center / user 等）
GET/POST /enterprises/{enterprise}/settings/billing/budgets
```

**审计日志**（不是用量数据，但排查问题很有用）：
- 搜索 `action:copilot` 查所有 Copilot 相关事件，例如 `action:copilot.cfb_seat_assignment_created`
- 搜索 `actor:Copilot` 查 Agent 的活动，保留 **180 天**
- 需要更长保留期请配置 audit log streaming 到自己的 SIEM

**GraphQL**：Copilot 的席位、计费和用量指标 **没有 GraphQL 接口**，全部是 REST-only。

---

## 官方文档索引

| 主题 | 链接 |
|------|------|
| Copilot 席位管理 API | https://docs.github.com/en/rest/copilot/copilot-user-management |
| Copilot 用量指标 API | https://docs.github.com/en/rest/copilot/copilot-usage-metrics |
| 用量指标概念（看板、归属规则） | https://docs.github.com/en/copilot/concepts/copilot-usage-metrics/copilot-metrics |
| 用量指标字段定义 | https://docs.github.com/en/copilot/reference/copilot-usage-metrics/copilot-usage-metrics |
| 用量指标示例 Schema | https://docs.github.com/en/copilot/reference/copilot-usage-metrics/example-schema |
| 团队级指标构造方法 | https://docs.github.com/en/copilot/reference/copilot-usage-metrics/team-level-metrics |
| 各数据源差异与对账 | https://docs.github.com/en/copilot/reference/copilot-usage-metrics/reconciling-usage-metrics |
| 活动报告字段定义 | https://docs.github.com/en/copilot/reference/metrics-data |
| 下载活动报告 | https://docs.github.com/en/copilot/how-tos/administer-copilot/download-activity-report |
| 查看用量看板 | https://docs.github.com/en/copilot/how-tos/administer-copilot/view-usage-and-adoption |
| 企业策略管理 | https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-enterprise/manage-enterprise-policies |
| AI Credits 计费概念 | https://docs.github.com/en/copilot/concepts/billing/usage-based-billing-for-organizations-and-enterprises |
| 企业计费 / 用量 API | https://docs.github.com/en/rest/billing/usage |
| 用量报告导出 API | https://docs.github.com/en/rest/billing/usage-reports |
| Cost Center / Budget API | https://docs.github.com/en/rest/billing/cost-centers ・ https://docs.github.com/en/rest/billing/budgets |
| 用 REST API 自动化用量报告 | https://docs.github.com/en/billing/tutorials/automate-usage-reporting |
| 计费报告字段参考 | https://docs.github.com/en/billing/reference/billing-reports |
| 查看用量与许可证 | https://docs.github.com/en/billing/how-tos/products/view-productlicense-use |

---

## 相关文档

- [GitHub Copilot 开通手册](ghcp-startup-guide.md)
- [Copilot 预算与计费说明](budget-config.md)
- [GH EMU 配置过程](ghcp-emu-config.md)
