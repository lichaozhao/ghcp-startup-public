## GitHub Copilot（GHCP）的计费 SKU 全景（截至 2026-10-30）

根据最新官方材料来看，目前GHCP的计费变化对现存客户没有影响。对计划新开通GHCP的客户也没有影响。仅仅是多了客户可以选择的一种新的计费模式。后续是否会有变化请关注官方公告。

### 基本概念

| 概念 | 说明 |
|---|---|
| **GH Ent** | GitHub Enterprise 的简称，是 GitHub 企业级 SaaS 服务的基础租户。 |
| **GHEC** | GitHub Enterprise Cloud 的简称，是 GitHub 企业级 SaaS 服务的一个 SKU，费用为 $21/月/用户，包含代码仓库等功能。 |
| **CfB** | GitHub Copilot for Business 的简称，独立 SKU，可在系统中查到对应的 SKU 信息。列表价 $19/月/用户，属于最低消费。 |
| **CfE** | GitHub Copilot for Enterprise 的简称，独立 SKU，$39/月/用户。需先采购 GH Ent 底座（$21/月/用户），实际成本为 $60/月/用户。 |
| **Paid AICs** | AIC = AI Credit, 1个AIC的价格是$0.01。Paid AICs是超出最低消费部分的实际 Token 使用费用。 |

### 企业开通 GHCP 的方式

**第一步：开通 GH Ent （用户自助开通）**

GH Ent 相当于 GitHub 为企业提供的独立 SaaS 租户，企业可在其中统一管理该租户下的所有用户账号。

**第二步：选择 GHCP 计费模式（二选一）**

- **方式一：直接开通 CfB （用户自助开通）**
  - 最低消费 $19/月/用户。
  - 这 $19 可全部用于支付 Copilot 的 Token 费用，并在 GH Ent 内作为共享池，全员共享额度。

- **方式二：联系 GH Sales，采用新计费模式**
  - **已采购 GHEC 的用户**（坐席费 $21/月/用户，可使用代码仓库等功能）：Copilot 不再有最低消费，按实际 Token 用量计费。
  - **未采购 GHEC 的用户**：需支付 $10/月/用户的坐席费（仅含 Copilot Access 功能），Copilot 同样不再有最低消费，按实际 Token 用量计费。

> **注意**
> CfE 需先采购 GH Ent 底座（$21/月/用户）才能使用。在新模式下，选择CfE $39 最低消费的套餐不再有意义，可以选择按实际 Token 用量计费。