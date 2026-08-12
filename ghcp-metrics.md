# GitHub Copilot 用量数据获取指南

> 面向新任 GitHub Copilot 管理员：帮你快速判断 **「我想要的数据在哪里、用哪个 API 取、是什么格式」**，以及 **如何集成到自己的管理控制台**。
>
> 本文所有 UI 路径、API 端点、字段名均基于 GitHub REST API 版本 `2026-03-10`。核对时间：2026-08-12。GitHub 的 Copilot 相关 API 变动频繁，落地前建议再次对照官方文档。

## 速览（为“太长不看“准备的汇总和导航）

**一句话**：Copilot 的管理数据分散在 **三套互不相通的 API** 里——席位、用量指标、计费用量。先确定你的问题属于哪一类，再去对应的链路取数；**跨链路的数字不能互相对账**。

### 快速导航

| # | 你想得到答案的问题 | UI 入口 | REST API | 格式 | 入口 |
|---|---------------|---------|----------|------|------|
| 1 | 谁被分配了 Copilot？谁长期未使用？ | Org → Settings → Copilot → Access；Ent → Billing and licensing → Licensing | `/copilot/billing/seats` 系列 | JSON / CSV | [→](#一谁被分配了-copilot谁长期未使用) |
| 2 | 谁在用 Chat、Agent、CLI、哪些模型和语言？ | Ent → Insights → Copilot usage | `/copilot/metrics/reports/*` | JSON + NDJSON | [→](#二谁在使用-chatagentcli模型和编程语言) |
| 3 | Copilot 生成和接受了多少代码？ | Ent → Insights → Code generation | 同上（共用一套 report API） | JSON + NDJSON | [→](#三copilot-生成和接受了多少代码) |
| 4 | 谁、哪个组织、哪个模型消耗了多少 AI Credits / 费用？ | Ent → Billing and licensing → Usage → AI usage | `/settings/billing/ai_credit/usage` 等 | JSON / CSV | [→](#四哪个用户组织模型和产品消耗了多少-ai-credits) |
| 5 | 企业总共买了多少席位、当前花了多少钱？ | Ent → Billing and licensing → Licensing / Overview | `/copilot/billing/seats`、`/settings/billing/usage/summary` | JSON | [→](#五企业整体购买了多少席位当前费用) |

> 缩写：`Ent` = Enterprise 层级，`Org` = Organization 层级。
> 第3项，Copilot 生成的代码仅指IDE里 Code Generation 的功能，不包括Copilot的对话、Agent等功能产生的代码。

### 前置条件及注意事项

1. **开策略**：`Enterprise → AI controls → Copilot → "Copilot usage metrics" = Enabled everywhere`，否则链路 B 全部无数据。
2. **建 Token**：一个 Classic PAT，勾选 `manage_billing:copilot` + `read:enterprise` + `read:org`，可覆盖本文全部端点。**计费端点和企业级席位端点不支持 Fine-grained PAT。**
3. **按上表选链路取数**。
4. **注意时效**：用量指标最长滞后 3 个完整 UTC 日，定时任务请拉 **T-4 天**；`last_activity_at` 只保留 90 天，必须自己存快照。

---

## 目录

**开始之前**
- [前置条件：策略、角色和 Token](#前置条件策略角色和-token) — 不配好这些，后面的 API 全部会报 403 或返回空
  - [1. 必须先打开 usage metrics 策略](#1-必须先打开-usage-metrics-策略)
  - [2. 角色要求](#2-角色要求)
  - [3. Token 类型（这是最大的坑）](#3-token-类型这是最大的坑)
  - [4. 通用请求头](#4-通用请求头)

**五类数据明细**

| 分类 | UI | API | 数据格式 | 其他 |
|------|----|-----|---------|------|
| [一、席位与长期未使用](#一谁被分配了-copilot谁长期未使用) | [Access 页面与活动报告](#uiaccess-页面与活动报告) | [席位查询端点](#api席位查询端点) | [席位 JSON 与活动报告 CSV](#返回数据席位-json-与活动报告-csv) | [90 天保留期与 24 小时延迟](#重要限制90-天保留期与-24-小时延迟) |
| [二、Chat / Agent / CLI / 模型 / 语言](#二谁在使用-chatagentcli模型和编程语言) | [Copilot usage 看板](#uicopilot-usage-看板) | [Usage Metrics Reports（12 个端点）](#apiusage-metrics-reports12-个端点) | [两段式下载与 NDJSON](#返回数据两段式下载与-ndjson) | [前置策略](#前置条件开启-usage-metrics-策略) ・ [主要字段](#主要字段功能模型语言cli-与采纳阶段) |
| [三、代码生成与接受](#三copilot-生成和接受了多少代码) | [Code generation 看板](#uicode-generation-看板) | [与用量指标共用端点](#api与用量指标共用同一套端点) | — | [主要字段](#主要字段生成接受与代码行数) ・ [PR 相关字段](#pr-相关字段pull_requests-对象) |
| [四、AI Credits 消耗与费用](#四哪个用户组织模型和产品消耗了多少-ai-credits) | [AI usage 视图与 CSV 报告](#uiai-usage-视图与-csv-报告) | **先读**：[汇总 vs 明细](#本节速览先搞清楚汇总和明细的区别) · [粒度对照](#各端点能拿到的粒度对照)<br>[① 汇总查询](#api-汇总查询ai-credits-与-premium-requests) · [② 明细导出](#api-明细导出推荐给自动化和成本分摊) | [JSON 与 CSV 报告字段](#返回数据json-与-csv-报告字段) | [什么场景用哪个](#什么场景用哪个) |
| [五、企业席位与费用总览](#五企业整体购买了多少席位当前费用) | [Licensing 与 Overview](#uilicensing-与-overview) | [席位总数与用量汇总](#api席位总数与用量汇总) | [页面汇总与 JSON](#返回数据页面汇总与-json) | — |

> 第3项，Copilot 生成的代码仅指IDE里 Code Generation 的功能，不包括Copilot的对话、Agent等功能产生的代码。

---

## 前置条件：策略、角色和 Token

### 1. 必须先打开 usage metrics 策略

否则链路 B 的所有 Dashboard 和 API 都拿不到数据（返回空或 403）。

```
Enterprise → AI controls（不是 Policies，GitHub 已重命名该导航）
          → 侧边栏 Copilot
          → 找到 "Copilot usage metrics" 策略
          → 设置为 Enabled everywhere
```

> Org 层级的对应入口：`Org → Settings → Copilot → Policies`。

### 2. 角色要求

| 数据 | 需要的角色 |
|------|-----------|
| 企业席位 / 活动报告 | Enterprise owner 或 Billing manager |
| 组织席位 / 活动报告 | Organization owner |
| 企业用量指标 | Enterprise owner、Billing manager，或拥有企业自定义角色权限 **"View Enterprise Copilot Metrics"** 的用户 |
| 组织用量指标 | Organization owner，或拥有 **"View Organization Copilot Metrics"** 权限的用户 |
| 计费用量 | Enterprise / Organization administrator 或 Billing manager |

### 3. Token 类型

**推荐Classic PAT。** 
**建议**：给自建控制台准备一个专用的 **Classic PAT**，勾选 `manage_billing:copilot` + `read:enterprise` + `read:org`，可以覆盖本文全部端点。

### 4. 通用请求头

```bash
curl -L \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2026-03-10" \
  https://api.github.com/...
```

---

## 一、谁被分配了 Copilot，谁长期未使用

**用途**：许可证盘点、回收长期不活跃的席位。

### UI：Access 页面与活动报告
**组织级**

```
Organization → Settings → 侧边栏 Copilot → Access
```
- 页面顶部显示已分配席位数和预估月度费用
- "Access management" 列表可按 **最后使用时间排序**
- 点击 **Get usage report → Get activity report** 下载 CSV

**企业级**

```
Enterprise → Billing and licensing → Licensing → Copilot 旁边点 Get activity report
```

### API：席位查询端点
```http
# 企业：列出所有席位（跨所有 Org 和 Enterprise Team，同一用户只计一次 total_seats）
GET /enterprises/{enterprise}/copilot/billing/seats?page=1&per_page=100

# 企业：查单个用户的席位详情
GET /enterprises/{enterprise}/members/{username}/copilot

# 组织：席位汇总和策略设置
GET /orgs/{org}/copilot/billing

# 组织：列出所有席位
GET /orgs/{org}/copilot/billing/seats?page=1&per_page=100

# 组织：查单个用户的席位详情
GET /orgs/{org}/members/{username}/copilot
```

> ⚠️ 常见错误：`GET /orgs/{org}/copilot/billing/seats/{username}` **不存在**。查单用户请用 `/orgs/{org}/members/{username}/copilot`。

### 返回数据：席位 JSON 与活动报告 CSV
**REST API：JSON**

`GET /enterprises/{enterprise}/copilot/billing/seats` 响应示例：

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

`GET /orgs/{org}/copilot/billing` 返回的是席位汇总 + 策略，**不是费用**：

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
  "plan_type": "business"
}
```

**UI Activity Report：CSV**，字段如下：

| 字段 | 说明 |
|------|------|
| `report_time` | 报告生成的 UTC 时间戳 |
| `login` | GitHub 用户名 |
| `last_authenticated_at` | 最近一次认证的 UTC 时间 |
| `last_activity_at` | 最近一次 Copilot 交互的 UTC 时间 |
| `last_surface_used` | 最近使用的界面：IDE 名称+版本（如 `VS Code 1.89.1`）、GitHub.com 功能名（如 `Copilot Chat`）或 `Unspecified` |

### 重要限制：90 天保留期与 24 小时延迟
- **`last_activity_at` 延迟最长 24 小时**；用户必须在 IDE 中开启 telemetry，IDE 内的活动才会被记录。
- **保留期 90 天**，不可修改。超过 90 天无活动，`last_activity_at` 会被置为 `nil`。所以 **"长期未使用"最多只能识别到 90 天**。
- CSV 报告每 30 分钟自动刷新一次。
- 非 GA 功能（如 Copilot Spaces、Copilot Spark）的使用不计入活动报告。
- 该系列端点标记为 **public preview**，字段可能变化。

---

## 二、谁在使用 Chat、Agent、CLI、模型和编程语言

**用途**：功能采纳率分析、模型治理、IDE 分布、团队对比。

### 前置条件：开启 usage metrics 策略
```
Enterprise → AI controls → Copilot → "Copilot usage metrics" = Enabled everywhere
```

### UI：Copilot usage 看板
```
Enterprise → Insights → 侧边栏 Copilot usage
```

同级还有另外两个看板：
- **Copilot impact**：采纳阶段（Adoption Cohort）分布及其与 PR 产出的关系
- **Code generation**：代码生成/接受明细（见下一节）

Dashboard 支持直接 **导出 NDJSON**，可以配合 Copilot Chat 做分析。

> Org 层级也有 usage metrics 看板（官方参考文档明确说明 dashboard 在 enterprise 和 organization 两级都可用），但官方 how-to 只给出了企业级的导航路径。Org 管理员如果找不到入口，用 API 是最稳妥的方式。

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
| `enterprise-1-day` / `organization-1-day` | 每天一条聚合记录 | 含活跃用户数、`pull_requests`、采纳阶段分布 |
| `enterprise-28-day` / `organization-28-day` | 28 天窗口 | 外层是窗口信息，`day_totals` 数组里是每天的聚合记录 |
| `users-1-day` / `users-28-day` | 每用户一条 | 含 `user_id`、`user_login`、`ai_credits_used`、`used_*` 标志位、`ai_adoption_phase` |
| `repos-1-day` | 每仓库一条 | 仅含 PR 生命周期数据；当天无 PR 活动的仓库不出现 |
| `user-teams-1-day` | 每 (用户, 团队) 一条 | 用于自行构造团队级指标 |

> 没有 `repos-28-day` 和 `user-teams-28-day`，只有 1-day 版本。
> **没有预聚合的团队级报表**。团队指标需要自己把 `user-teams-1-day` 和 `users-1-day` 按 `user_id` + `day` join 起来，再按 `team_id` 聚合。
> 千万不要用 `users-28-day` 去 join 单日的 `user-teams-1-day`，会把 28 天的活动全部错误归属到 join 当天的团队。

**`day` 参数是必填的**（`/latest` 端点除外）。

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

> `download_links` 是 **限时签名 URL**，官方只说"有过期时间"，**没有公布具体时长**。所以请在拿到响应后 **立即下载**，不要把 URL 存起来隔天再用。

### 主要字段：功能、模型、语言、CLI 与采纳阶段
**功能 / 模型 / 语言 / IDE 维度（breakdown 数组）**

| 字段 | 维度 | 说明 |
|------|------|------|
| `totals_by_ide[]` | `ide` | 按 IDE 拆分；per-user 报表还含 `last_known_ide_version`、`last_known_plugin_version` |
| `totals_by_feature[]` | `feature` | 按功能拆分 |
| `totals_by_language_feature[]` | `language`, `feature` | 语言 × 功能 |
| `totals_by_language_model[]` | `language`, `model` | 语言 × 模型（仅 chat，不含补全） |
| `totals_by_model_feature[]` | `model`, `feature` | 模型 × 功能（仅 chat） |

`feature` 的取值：`code_completion`、`chat_inline`、`chat_panel_ask_mode`、`chat_panel_edit_mode`、`chat_panel_agent_mode`、`chat_panel_plan_mode`、`chat_panel_custom_mode`、`chat_panel_unknown_mode`、`agent_edit`、`copilot_cli`、`copilot_app`、`others`。

`ide` 的取值示例：`vscode`、`visualstudio`、`intellij`、`eclipse`、`xcode`、`neovim`、`vim`、`emacs`、`zed`。

`model` 的取值：具体模型 ID（如 `gpt-5.4`、`claude-sonnet-4.6`）、`auto`（自动选模型且未归因到具体模型）、`unknown`、`others`。

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

> **CLI 用量是完全独立统计的**，不计入 `totals_by_ide`、`totals_by_feature`，也不计入 IDE 口径的活跃用户数。想统计 CLI 活跃人数请用 `daily_active_cli_users`。
>
> `totals_by_copilot_app` 的结构与 CLI 相同，但 Copilot App 的编码活动 **会** 同时体现在 `totals_by_feature` 等数组里（`feature = copilot_app`）。

**per-user 报表的功能使用标志位**

```
used_chat, used_agent, used_cli, used_copilot_app
used_copilot_coding_agent / used_copilot_cloud_agent   # 同一个值，两个名字并存（向后兼容）
used_copilot_code_review_active                        # 主动请求或采纳了 review 建议
used_copilot_code_review_passive                       # 被自动指派 review 但没互动
```

**活跃用户计数（聚合报表）**

```
daily_active_users / weekly_active_users / monthly_active_users
monthly_active_chat_users / monthly_active_agent_users
daily|weekly|monthly_active_copilot_cloud_agent_users
daily|weekly|monthly_active_copilot_code_review_users
daily|weekly|monthly_passive_copilot_code_review_users
daily_active_cli_users          # 独立于 IDE 口径
daily_active_copilot_app_users  # 仅企业级报表
```

**第三方 Agent（`totals_by_3rd_party_agent[]`）**

```
agent_name, agent_id, user_initiated_interaction_count, session_count
```
用 `agent_id` 做分组键（`agent_name` 可能变），并且这里的 `user_initiated_interaction_count` 统计的是 **Agent 任务启动次数**，不要和顶层同名字段混算。

**采纳阶段（Adoption Phase）**

分类逻辑：在滚动 28 天窗口内，**至少 2 个活跃日** 满足条件才归入对应阶段。

| 阶段 | 判定条件 |
|------|---------|
| `No Cohort`（看板显示为 Passive users） | 未达到任何阶段的 2 天门槛 |
| `Phase 1: Code first` | ≥2 天满足 `used_chat=true` 或 `code_acceptance_activity_count>0` 或 `used_agent=true` |
| `Phase 2: Agent first` | ≥2 天使用了 **一个** GitHub 侧 agent surface：`used_cli`、`used_copilot_cloud_agent` 或 `used_copilot_code_review_active` |
| `Phase 3: Multi-agent` | ≥2 天使用了 **两个及以上** 上述 surface |

阶段每天基于滚动 28 天窗口重算，**用户的阶段会天天变**，这是预期行为。

---

## 三、Copilot 生成和接受了多少代码

**用途**：衡量代码贡献量、AI 生成占比、Agent vs 人工的比例。只看 **IDE 里 Code Generation 的功能**，不包括 Copilot Chat、Agent 等功能产生的代码。

### UI：Code generation 看板
```
Enterprise → Insights → 侧边栏 Code generation
```

看板指标包括：过去 28 天 AI 改动的总行数、Agent 贡献占比、Agent 平均删除行数、每日增删行数、用户发起 vs Agent 发起的代码改动（并可按模型、按语言拆分）。

### API：与用量指标共用同一套端点
与上一节 **共用同一套 Usage Metrics Reports API**，不需要额外调用别的端点：

```http
GET /enterprises/{enterprise}/copilot/metrics/reports/enterprise-1-day?day=YYYY-MM-DD
GET /enterprises/{enterprise}/copilot/metrics/reports/users-1-day?day=YYYY-MM-DD
GET /orgs/{org}/copilot/metrics/reports/organization-1-day?day=YYYY-MM-DD
```

### 主要字段：生成、接受与代码行数
| 字段 | 含义 |
|------|------|
| `code_generation_activity_count` | Copilot 产出事件数。**包含注释和 docstring**；一个 prompt 产生多个代码块算多次 |
| `code_acceptance_activity_count` | 用户接受的次数。计入 apply to file、insert at cursor、insert into terminal、Copy 按钮；**不计** 操作系统剪贴板复制（Ctrl+C）。每次接受动作只 +1，无论原始 prompt 生成了几个代码块 |
| `loc_suggested_to_add_sum` | Copilot 建议新增的代码行数（补全、inline chat、chat panel 等，**不含 agent edit**） |
| `loc_suggested_to_delete_sum` | 建议删除的行数（**功能规划中，目前基本为 0**） |
| `loc_added_sum` | 实际写入编辑器的新增行数（接受的补全、应用的代码块、agent 和 edit 模式） |
| `loc_deleted_sum` | 从编辑器删除的行数（目前来自 agent edit） |
| `user_initiated_interaction_count` | 显式发给 Copilot 的 prompt 数。**不含** 打开 chat 面板、切换模式、快捷键唤起 UI、改配置 |

**接受率计算**：`code_acceptance_activity_count ÷ code_generation_activity_count`（看板和 API 用的是同一个公式，只是四舍五入可能不同）。

> ⚠️ `code_generation_activity_count` 和 `user_initiated_interaction_count` **不能直接比较**，因为一个 prompt 可以产生多次生成。
>
> ⚠️ `agent_edit` 这个 feature 值不参与"建议类"指标，可能不填充 `user_initiated_interaction_count` 之类的字段。

### PR 相关字段（`pull_requests` 对象）

出现在聚合报表和 `repos-1-day` 报表中：

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

---

## 四、哪个用户、组织、模型和产品消耗了多少 AI Credits

**用途**：成本分摊、预算告警、对账。**这是唯一的账单口径数据源。**

> AI Credits 和预算的计费逻辑请参考本仓库的 [预算与计费说明](budget-config.md)。本节只讲怎么把数据取出来。

### 本节速览：先搞清楚「汇总」和「明细」的区别

这一节是全文最容易取错数的地方。原因是 GitHub 为计费数据提供了 **两种性质完全不同的取数方式**，能拿到的粒度差别很大：

| | **① 汇总查询**（同步） | **② 明细导出**（异步） |
|---|---|---|
| 怎么调 | `GET` 一次拿到结果 | `POST` 建任务 → 轮询状态 → 下载文件 |
| 端点 | `.../ai_credit/usage`<br>`.../premium_request/usage`<br>`.../usage/summary` | `POST .../settings/billing/reports` |
| 返回 | JSON，**已按 SKU / 模型聚合** | CSV 文件，**逐行原始明细** |
| 有没有"谁"这一列 | ❌ **没有**，只能靠 `user=` 参数一个个过滤 | ✅ 有 `username` 列 |
| 有没有 token 明细 | ❌ | ✅ `input` / `output` / `cache_read` / `cache_write` |
| 时间粒度 | 按 `year`/`month`/`day` 参数筛，结果不含日期列 | 有 `date` 列，可做时间序列 |
| 实时性 | 立即返回 | 异步生成，需轮询 |
| 适合 | 看板数字、预算告警、快速查某人某月用量 | 成本分摊、逐用户账单、BI 入仓 |

**一句话选型：**
- 只想知道 **「这个月一共花了多少」「某个人花了多少」** → 用 ① 汇总查询，一次请求搞定
- 要做 **「全公司每个人各花了多少」的分摊表，并且要看Token的明细** → 用 ② 明细导出，别用 ① 去循环拉


### UI：AI usage 视图与 CSV 报告
```
Enterprise → Billing and licensing → Usage
```
- **Metered usage** 视图：所有计费产品的用量
- **AI usage** 视图：Copilot AI Credits 专用视图，可看总消耗、重度用户、模型花费分布、各 Org 采纳情况

支持在页面上按维度筛选、分组、选时间范围。

**下载 CSV 报告：**
1. 在 "Metered usage" 或 "AI usage" 页面顶部点 **Get usage report**
2. 指定报告参数
3. 点 **Email me the report**
4. 收到邮件后点链接下载，**链接 24 小时后过期**

也可以点图表右上角的 **⋯ → 选择格式** 直接下载图表数据。

> ⚠️ **Org owner 在 AI usage 视图里看不到 per-user 数据**，只有 Enterprise owner / Billing manager 可以按用户筛选。Org 层级要看每用户消耗，必须下载报告。

### API ①：汇总查询（AI Credits 与 Premium Requests）
**企业级**

```http
GET /enterprises/{enterprise}/settings/billing/ai_credit/usage
```

支持的查询参数：`year`、`month`、`day`、`organization`、`user`、`model`、`product`、`cost_center_id`，比如：

```
GET /enterprises/{enterprise}/settings/billing/ai_credit/usage?year=2026&month=7&user=zhangsan
```


**组织级**

```http
GET /organizations/{org}/settings/billing/ai_credit/usage
```

支持的查询参数：`year`、`month`、`day`、`user`、`model`、`product`


> ⚠️ 三个高频踩坑点：
> 1. 路径是 **`ai_credit`（单数）**，不是 `ai_credits`。
> 2. 组织级计费端点用的是 **`/organizations/{org}/`**，不是 `/orgs/{org}/`。（`/orgs/{org}/settings/billing/` 只剩下一个无关的历史端点 `advanced-security`。）
> 3. **没有 `hour` 参数**，最细只到 `day`。

### 返回数据：JSON 与 CSV 报告字段
**REST API：JSON**

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

**几个可以从响应里直接读出来的计费常识：**
- 1 AI Credit = **$0.01**（`pricePerUnit: 0.01`）
- `grossAmount`是列表价格，`discountAmount` 是从Github直接支付的折扣后价格，但是大多数客户都是Azure计费，折扣在Azure侧体现，所以这里的discountAmount和grossAmount相同。

**UI 下载报告：CSV**

| 报告类型 | 入口 | 最长周期 | 特点 |
|---------|------|---------|------|
| Summarized usage report | Metered usage | 1 年 | 按 `date`+`sku`+`repository`+`cost_center_name`(+`organization`) 汇总 |
| Detailed usage report | Metered usage | 31 天 | 在汇总报告基础上增加 `username`、`workflow_path` |
| AI usage report | AI usage | 31 天 | 按 `date`+`model`+`username` 汇总，**并拆出 token 明细** |

CSV 字段：`date`、`product`、`sku`、`quantity`、`unit_type`、`applied_cost_per_quantity`、`gross_amount`、`discount_amount`、`net_amount`、`username`、`organization`、`repository`、`workflow_path`、`cost_center_name`。

AI usage report 额外包含：`model`、`input`、`output`、`cache_read`、`cache_write`（token 明细）。

### API ②：明细导出（推荐给自动化和成本分摊）

汇总查询拿不到逐用户明细时，用这套 **异步导出 API**，可以拿到和 UI 下载的 CSV 完全一致的明细报告：

```http
POST /enterprises/{enterprise}/settings/billing/reports
{
  "report_type": "ai_credit",       // detailed | summarized | premium_request | ai_credit
  "start_date": "2026-07-01",
  "end_date": "2026-07-31",
  "send_email": false
}

GET  /enterprises/{enterprise}/settings/billing/reports              # 列出所有导出任务
GET  /enterprises/{enterprise}/settings/billing/reports/{report_id}  # 查状态并取下载地址
```

响应：

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

轮询 `status` 直到 `completed`，再从 `download_urls` 取文件。

---

## 五、企业整体购买了多少席位、当前费用

### UI：Licensing 与 Overview
```
Enterprise → Billing and licensing → Licensing
```
- 页面滚到底部，点 **Copilot** 标签页，可看已分配席位数和当前账期总花费
- "Enterprise Cloud" 区域点 **More details** 看许可证明细，点 **Download CSV report** 下载
- 点 **Manage** 可看许可证变更历史（public preview，每日快照，含操作人和生效日期）

```
Enterprise → Billing and licensing → Overview
```
查看账单概览。

### API：席位总数与用量汇总
```http
# 席位总数（total_seats 已对跨 Org/Team 的重复用户去重）
GET /enterprises/{enterprise}/copilot/billing/seats

# 组织级席位汇总
GET /orgs/{org}/copilot/billing

# 所有计费产品的用量汇总（含 Copilot）
GET /enterprises/{enterprise}/settings/billing/usage/summary
    ?year=&month=&day=&organization=&repository=&product=&sku=&cost_center_id=

# 明细版（注意：仅企业启用了 enhanced billing platform 时可用）
GET /enterprises/{enterprise}/settings/billing/usage
    ?year=&month=&day=&cost_center_id=

# 当前 AI Credits 消耗和费用
GET /enterprises/{enterprise}/settings/billing/ai_credit/usage
```

> `usage/summary` 默认返回当年的数据，且 **只保留过去 24 个月**。

### 返回数据：页面汇总与 JSON
- UI：页面汇总 + CSV 下载
- REST API：JSON

## 常见问题

| 坑 | 说明 |
|----|------|
| 拿用量指标去对账单 | 注意区分 Metrics，AICs Usage（ 汇总 & 明细 ）。 |
| 昨天的数据是空的 | 用量指标最长滞后 **3 个完整 UTC 日**，拉 T-4 才稳。 |
| 数据比预期少很多 | 用户在 IDE 里关掉了 telemetry。此时 IDE 明细（per-IDE、per-feature、LoC）全部缺失，但服务端遥测仍可能把该用户计入活跃用户数，导致"有活跃用户但没有明细"的现象。 |
| 找不到某个团队 | **当天座席用户少于 5 人的团队会被 user-teams 报表排除。** 把团队数据加总起来会小于企业/组织总数，差额就是只属于小团队的用户。 |
| CLI 用量对不上 IDE 数字 | CLI 是独立统计的，不计入 IDE 口径的活跃用户和 feature 明细。 |
| `download_links` 过期 | 是限时签名 URL，官方未公布具体时长。拿到后立刻下载，不要存起来第二天用。 |
| 用户的采纳阶段天天在变 | 阶段基于滚动 28 天窗口每天重算，这是预期行为，不是数据错误。 |
| Fine-grained PAT 报 403 | 计费端点和企业级席位端点 **不支持 Fine-grained PAT**，只能用 Classic PAT。 |
| 组织计费端点 404 | 路径是 `/organizations/{org}/settings/billing/...`，不是 `/orgs/{org}/`。 |
| 想查 90 天前谁没用过 Copilot | `last_activity_at` 只保留 90 天，之后置为 `nil`。必须自己每天存快照。 |
| Org / Enterprise 的 PR 数字不一致 | 官方说明：两级在用户去重和归因时间上有差异，出现差值属正常。 |

---

## 补充：其他可能用得上的端点

```http
# Copilot Agent 会话活动记录（public preview，仅 EMU 企业owner可用，游标分页）
GET /enterprises/{enterprise}/copilot/usage-records?phrase=&per_page=25&after=&before=&order=desc

# 席位分配 / 回收
POST   /orgs/{org}/copilot/billing/selected_users     # 分配
DELETE /orgs/{org}/copilot/billing/selected_users     # 回收（设置 pending_cancellation_date）
POST   /orgs/{org}/copilot/billing/selected_teams
DELETE /orgs/{org}/copilot/billing/selected_teams

# Cost Center（成本分摊，注意 ai_credit_pool_enabled 属性）
GET/POST /enterprises/{enterprise}/settings/billing/cost-centers

# 预算（budget_product_sku 支持 ai_credits 和 premium_requests）
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
| 用量指标字段定义 | https://docs.github.com/en/copilot/reference/copilot-usage-metrics/copilot-usage-metrics |
| 用量指标示例 Schema | https://docs.github.com/en/copilot/reference/copilot-usage-metrics/example-schema |
| 团队级指标构造方法 | https://docs.github.com/en/copilot/reference/copilot-usage-metrics/team-level-metrics |
| 各数据源差异与对账 | https://docs.github.com/en/copilot/reference/copilot-usage-metrics/reconciling-usage-metrics |
| 活动报告字段定义 | https://docs.github.com/en/copilot/reference/metrics-data |
| 下载活动报告 | https://docs.github.com/en/copilot/how-tos/administer-copilot/download-activity-report |
| 查看用量看板 | https://docs.github.com/en/copilot/how-tos/administer-copilot/view-usage-and-adoption |
| 企业策略管理 | https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-enterprise/manage-enterprise-policies |
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
