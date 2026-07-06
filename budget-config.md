# 基本计费逻辑和相关名词

**本文大纲：**
- 计费概览
- 官方词汇
- 方便理解而在本文中使用的词汇
- 计费和预算控制详细逻辑
- 计费的例子
- 建议设置

## 计费逻辑
>这里只以 copilot for business（cfb）的用户举例，如果是 copilot for enterprise（cfe），计费逻辑是一样的，只是两者的每月默认额度不同。
- 每个用户每个月需要支付19$ 订阅费用，里面包含 19$ 的额度，即 1900积分。消费超过19$ 的部分按实际模型Token的消耗情况做额外计费。
- 每次模型调用的成本等同于该模型厂商的列表价格（区分 input、output、cache token等）。参考[官方文档](https://docs.github.com/en/copilot/reference/copilot-billing/models-and-pricing)
- 所有用户的AI积分会组成一个大的积分池，所有用户共享。默认情况下，每个成员都可以不限量的使用积分池的数据。极端情况，某个用户有可能在短时间内消耗完整个积分池的积分。
- 2026年的6-8月，这三个月对6月1日之前创建了github enterprise的客户有促销，每个用户在成本不变的前提下，19$ 实际获得额度是3000积分。

## 官方词汇
- AICs：AI Credits，AI积分，19$ 等于1900 AICs。
- Cost Center：成本中心，人或团队的虚拟管理单元
- Budget：预算，需要关联到 成员、Cost Center或者Enterprise上，来设置对应的成员、Cost Center或者Enterprise在当月可以使用的额度。

## 方便理解而在本文中使用的词汇
- 默认池：每个成员有1900 AICs的额度，所有成员的额度加起来组成了一个默认的额度池。
- 增量池：如果默认池不够用，管理员可以通过配置Enterprise的预算来增加额度，这个增加的额度就叫增量池。增量池有enterprise 范围的增量池，也有Cost Center范围的增量池。
  - Enterprise 范围的增量池，在默认池耗尽后，对所有成员生效的增量池
  - Cost Center 范围的增量池，在默认池耗尽后，只对Cost Center下的成员生效的增量池
  - Cost Center 在7月初新增了 ai_credit_pool_enabled 属性，该属性需要用API配置。默认为false，但这个值被配置为true时：
    - 仅允许选择 User 和 Enterprise Team 作为 Cost Center 的成员，不允许选择 Organization 作为 Cost Center 的成员
    - Cost Center 会获得一个可以从 Enterprise 范围内的默认池中取用额度的上限，上限等于 Cost Center 所包含的所有成员的配额的总和
- 个人配额：个人可以从额度池中取用的额度。个人配额有三种：
  - Universal Budget（对所有成员生效），优先级最低
  - Cost Center Budget（对Cost Center下的成员生效），优先级中等
  - User Budget（对指定的单个成员生效），优先级最高

# 计费和预算控制逻辑 
> [官方文档](https://docs.github.com/en/copilot/concepts/billing/usage-based-billing-for-organizations-and-enterprises)：A user's included AI credits are pooled at the billing entity level. 默认情况是一个enterprise对应一个billing entity。

> 一次 Copilot 调用能否继续，主要看两件事：
> 1. 用户自己的使用上限是否还有额度，也就是个人配额；
> 2. 企业或 Cost Center 的额度池是否还有可用额度。

1. 默认池来自成员订阅内包含的 AICs。增加成员才会增加默认池。
2. 用户使用 Copilot 时，优先消耗默认池里的 AICs。
3. 个人配额只是限制单个用户最多能用多少，不会增加默认池或增量池的总额度。
4. 个人配额的生效优先级是：User Budget > Cost Center Budget > Universal Budget。
   - 如果配置了 User Budget，就以 User Budget 为准。
   - 如果 User Budget 已耗尽，即使 Cost Center Budget 或 Universal Budget 还有额度，用户也不能继续使用 Copilot。
5. 默认池不够用时，可以通过预算配置增量池：
   - Enterprise 增量池：默认池耗尽后，可供企业范围内符合条件的成员使用。
   - Cost Center 增量池：默认池耗尽后，只供该 Cost Center 下的成员使用。
6. Cost Center 未配置 `ai_credit_pool_enabled`，或该值为 `false` 时：
   - Cost Center 成员先消耗默认池；
   - 默认池耗尽后，优先消耗该 Cost Center 的增量池；
   - Cost Center 增量池耗尽后，再根据配置判断是否可以使用 Enterprise 增量池；
   - 不属于任何 Cost Center 的成员，在默认池耗尽后，只能使用 Enterprise 增量池。
7. Cost Center 配置 `ai_credit_pool_enabled=true` 时：
   - Cost Center 会获得一个可从企业默认池中取用的额度上限；
   - 这个上限等于该 Cost Center 内所有成员默认套餐配额的总和，比如cost center里有两个人，一个人配额是 10$ ，另一个人配额是 100$ ，则该 Cost Center 的默认池取用额度上限为 38$ ，即这两个人的默认套餐配额总和；
   - Cost Center 成员使用 Copilot 时，仍然消耗企业默认池；
   - 同时，也会扣减该 Cost Center 可从默认池中取用的额度；
   - 当该 Cost Center 的默认池取用额度耗尽后，才会转为消耗 Cost Center 增量池；
   - 如果企业里所有的用户都有Cost Center的归属，则就可以实现每个Cost Center内的用户在 Cost Center的范围内共享自己的默认池额度，而不会影响其他Cost Center可以从默认池额度。
8. 某个成员不能继续使用 Copilot，通常有以下原因：
   - 个人配额已经耗尽；
   - 个人配额未耗尽，但默认池已耗尽，且没有可用增量池；
   - 有增量池，但该成员不在可使用该增量池的范围内；
   - 可用增量池本身也已经耗尽。


## 例子
### 1.未做任何设置的情况
- Enterprise里有两个人张三和李四。每个人 19$ 额度，企业总计默认池有 38$ 的额度。
- 管理员未设置个人配额，也没有设置enterprise范围的预算
- 张三和李四可以无限量使用github copilot，月底会出账单，账单按实际token的消耗情况来计算
  - 如果张三用了 19$ ，李四用了 10$ ，月底账单是 38$
  - 如果张三用了 20$ ，李四用了 10$ ，月底账单是 38$
  - 如果张三用了 30$ ，李四用了 20$ ，月底账单是 50$

### 2.设置个人配额的情况
- Enterprise里有两个人张三和李四。每个人 19$ 额度，企业总计默认池有 38$ 的额度。
  - 管理员设置个人配额为 100$ ，但是没有设置enterprise范围的预算
    - 张三和李四各自可以使用 100$ 的github copilot上限，月底账单按实际token的消耗情况来计算，但是最低是38$
  - 管理员设置张三个人配额为 100$ ，李四个人配额为 10$ ，但是没有设置enterprise范围的预算
    - 张三用满了 100$ ，李四也用满了 10$ ，月底账单是 张三和李四的基础额度 38$ + 张三的增量部分 81$ + 李四的增量部分 0$ = 119$

### 3.设置个人配额和增量池的情况
- Enterprise里有两个人张三和李四。每个人 19$ 额度，企业总计默认池有38$的额度。
  - 管理员设置每个人的个人配额为 100$ ，并且在enterprise scope设置预算为 0$ 
    - 李四休假，张三使用到 38$ 后，默认池耗尽了，张三不能再使用copilot了，月底账单是 38$
    - 张三用了 30$ 后休假，李四休假回来后，李四还可以使用 8$ 的默认池剩余额度，月底账单是 38$
  - 管理员设置个人配额为 100$ ，并且在enterprise scope设置预算为 200$
    - 张三用了 50$ （超了 31$ ），李四用了 30$ （超了 11$ ），则：
      - 默认池剩余 0$；
      - 增量池剩余 200$ - 42$ = 158$；
      - 月底账单是 80$ （ 38$ （基础额度）+ 31$ + 11$ = 80$）；
    - 张三用了 100$ ，李四用了 100$ ，月底账单是 200$ ；增量池剩余 38$ ；

### 4. Cost Center 开启 `ai_credit_pool_enabled=true` 的情况

#### 4.1 只切分默认池，不设置 Cost Center 增量池

- Enterprise 里有 4 个人，每个人 19$ 额度，企业总计默认池有 76$。
- 管理员创建两个 Cost Center：
  - 研发 Cost Center：张三、李四，总计可从默认池取用 38$；
  - 销售 Cost Center：王五、赵六，总计可从默认池取用 38$。
- 两个 Cost Center 都配置了 `ai_credit_pool_enabled=true`。
- 没有设置 Cost Center 增量池，也没有设置 Enterprise 增量池。
- 每个人的个人配额都设置为 100$。

这时：

- 如果张三和李四合计用了 38$，研发 Cost Center 可从默认池取用的额度已经耗尽；
- 即使销售 Cost Center 还没有使用，企业默认池里理论上还有 38$ 没被销售使用，张三和李四也不能继续使用 Copilot；
- 销售 Cost Center 的王五、赵六仍然可以使用自己 Cost Center 对应的 38$ 默认池额度。

这个配置的效果是：默认池被按 Cost Center 隔离，某个 Cost Center 不能消耗其他 Cost Center 的默认池额度。

#### 4.2 Cost Center 默认池取用额度耗尽后，继续使用 Cost Center 增量池

- Enterprise 里有 4 个人，每个人 19$ 额度，企业总计默认池有 76$。
- 研发 Cost Center 有张三、李四，可从默认池取用 38$。
- 研发 Cost Center 配置了 `ai_credit_pool_enabled=true`。
- 管理员给研发 Cost Center 设置了 20$ 的增量池。
- 张三和李四的个人配额都设置为 100$。

这时：

- 张三和李四合计先使用 38$ ，这部分消耗企业默认池，同时也消耗研发 Cost Center 可从默认池取用的 38$ ；
- 超过 38$ 后，继续消耗研发 Cost Center 的 20$ 增量池；
- 张三和李四合计最多可以使用 58$；
- 如果继续使用超过 58$，即使个人配额还没耗尽，也不能继续使用 Copilot。
- 如果张三李四之外的用户没有做 19$ 的限制，而是更多的个人配额（比如每个人 100$ ），则企业默认池可能被这些人提前耗尽，那么张三和李四使用时，就只能从Cost Center的额外池额度里继续使用 Copilot 了。
  
月底账单至少包含 76$ 基础订阅费用；研发 Cost Center 超出默认池的 20$ 会作为额外用量计费。

#### 4.3 个人配额仍然会优先生效

- Enterprise 里有 2 个人，张三和李四，每个人 19$ 额度，企业默认池有 38$。
- 两人都在研发 Cost Center。
- 研发 Cost Center 配置了 `ai_credit_pool_enabled=true`，可从默认池取用 38$。
- 研发 Cost Center 还配置了 50$ 增量池。
- 张三的个人配额是 25$ ，李四的个人配额是 100$ 。

这时：

- 张三最多只能使用 25$，即使研发 Cost Center 的默认池或增量池还有额度，张三也不能超过自己的个人配额；
- 李四可以继续使用研发 Cost Center 剩余的默认池取用额度，以及后续的 Cost Center 增量池；
- 所以用户能否继续使用 Copilot，仍然要先看个人配额，再看 Cost Center 或 Enterprise 的额度池是否还有可用额度。

#### 4.4 所有成员都归属到 Cost Center 时，可以实现默认池按 Cost Center 管理

- Enterprise 里有 4 个人，每个人 19$ 额度，企业默认池总计 76$。
- 所有人都被分配到 Cost Center：
  - Cost Center A：张三、李四，可从默认池取用 38$；
  - Cost Center B：王五、赵六，可从默认池取用 38$。
- 两个 Cost Center 都配置了 `ai_credit_pool_enabled=true`。
- 每个 Cost Center 可以单独设置自己的增量池。

这时：

- Cost Center A 的成员只能使用 Cost Center A 对应的默认池取用额度，以及 Cost Center A 的增量池；
- Cost Center B 的成员只能使用 Cost Center B 对应的默认池取用额度，以及 Cost Center B 的增量池；
- 一个 Cost Center 用量过高，不会直接消耗另一个 Cost Center 的默认池额度；
- 管理员可以更接近“部门预算”的方式来管理 Copilot 用量。


# 建议设置
1. 个人配额设置的总额度不应该小于 **默认池+增量池** 的总额度，否则可能会出现成员还有额度但是无法使用copilot的情况；
2. 如果要实现按部门共享默认池的额度，则需要为每个部门单独创建 cost center ，并且要保证每个人都有一个cost center归属。并且每个cost center都配置了 ai_credit_pool_enabled=true；
3. 6月30日增加了 Cost Center User Level Budget 的功能，方便以 cost center 为范围设置每个成员的个人配额。可以酌情使用。
   
## 配置过程

- 配置 Cost Center （注意 ai_credit_pool_enabled 属性） 的 [API](https://docs.github.com/en/enterprise-cloud@latest/rest/billing/cost-centers?apiVersion=2026-03-10#create-a-new-cost-center)
- 参考[操作手册](https://github.com/nickhou1983/Github-Sop/blob/main/budget/github-enterprise-budget.md#%E6%AD%A5%E9%AA%A4-1%E6%A3%80%E6%9F%A5%E6%98%AF%E5%90%A6%E5%B7%B2%E9%85%8D%E7%BD%AE%E4%BC%81%E4%B8%9A%E7%BA%A7-budget)


