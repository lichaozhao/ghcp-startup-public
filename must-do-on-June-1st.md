
# 6月1日必须做的事情
- 如果您是free org的用户，参考：[org-must-do-on-June-1st.md](org-must-do-on-June-1st.md)
- 如果您是enterprise的用户，参考：[ent-must-do-on-June-1st.md](ent-must-do-on-June-1st.md)

# 常见问题：
> 后续会逐渐更新 
## 管理员如何查看每个人的消耗情况
- 费用查看
  入口在Budget设置页，点击universal 预算可以看到：
   ![图](ent-bugdet-screenshots/admin-view.png)

- Token查看，暂时未上线

## 用户如何查看自己的消耗情况
- 方法1，访问地址 https://github.com/settings/copilot/features 
- 方法2，重启vscode之后，在右下角可以查看，类似：
![图](ent-bugdet-screenshots/user-view.png)

## 如何调整某些用户的配额 
1. 还是进入上文提到的Budget设置页
2. 创建budget
   - Budget Type选择**AI credits budget** 
   - Budget Scope选择**Users**，这里选择具体的用户
   - Budget amount 按实际预期设置，注意勾选**Stop usage when user's budget limit is reached**选项，如果不勾选则用户即使达到预算限制也可以正常使用
   - 设置完毕后，应该如图：
    ![图](ent-bugdet-screenshots/user-budget.png)

## 具体计费规则
参考：[GHCP新计费模式说明](budget-config.md)中的计费规则部分