# 6月1日 计费方式变更及带来的影响
> 适用于 GitHub Free Org 用户
> 请务必在6月初完成以下检查和设置，以避免不必要的费用风险

## 计费方式变更
- 6月1日 GitHub Copilot的计费方式已经变更为基于使用量（Token）的计费方式
- 所有用户的额度在组织范围内是共享的
- 默认所有的用户没有限额
- 组织范围也没有限额

更多信息查看[GHCP新计费模式说明](budget-config.md)

## 可能出现的情况
- 用户可以无限量使用造成月底账单很大
- 某个用户短时间内消耗掉组织内所有用户的共享额度


# 建议马上做的操作

需要马上做两个操作：
- **检查并设置组织级预算**
- **检查并设置每个人的配额**

## 检查是否设置了组织级预算的步骤
1. 用管理员访问地址 `https://github.com/organizations/<您的组织名称>/settings/copilot/policies`，进入 Copilot 设置页
2. 检查是否开启了 **"Premium request paid usage"** 选项；如果没有，开启它以启用预算功能
3. 访问 `https://github.com/organizations/<您的组织名称>/settings/billing/budgets`，进入预算设置页
4. 确认预算列表如图：![图](org-bugdet-screenshots/org-budget.png)
5. 如果没有，请点击右上角按钮创建预算
6. 创建预算时的参数
7. 创建结果如图：![图](org-bugdet-screenshots/org-create-budget.png)

## 检查是否设置了每个人的配额的步骤
1. 访问 `https://github.com/organizations/<您的组织名称>/settings/billing/budgets`，进入预算设置页
2. 确认预算列表如图：![图](org-bugdet-screenshots/org-budget.png)
3. 如果没有，请点击右上角按钮创建预算
4. 创建个人配额时的参数
5. 创建结果如图：![图](org-bugdet-screenshots/org-create-budget.png)

# 其他问题

## 管理员如何查看每个人的消耗情况

- 费用查看
- Token查看

## 用户如何查看自己的消耗情况
- 费用查看
- Token查看

## 如何调整某些用户的配额 
UI or API
