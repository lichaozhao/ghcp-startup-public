# GitHub Copilot 的管理员指南

本仓库整理 GitHub Copilot 企业开通、计费和预算配置相关资料，适合企业管理员快速参考。

## 内容
- GH 的账号体系[说明](#github-账号体系说明)。
- [GitHub Copilot 开通手册](ghcp-startup-guide.md)：从创建 GitHub Enterprise 到分配 Copilot 权限的完整流程。
- 开通 [GH EMU 的配置过程](ghcp-emu-config.md)：企业管理员如何做企业 SSO 的集成。
- [Coilot 预算与计费说明](budget-config.md)：Copilot AI Credits、预算和个人配额的基本概念和计费逻辑。管理员必读。
- 6月初管理员需要针对新的计费做的[调整](must-do-on-June-1st.md)。
- Copilot Cli作为服务运行的特殊情况[说明](copilot-cli-as-service.md)。

## 适用对象

- GitHub Enterprise 管理员
- Copilot 采购、计费或成本管理负责人
- 需要为团队开通 Copilot 的技术负责人

## GitHub 账号体系说明
> 在 [GitHub Copilot 开通手册](ghcp-startup-guide.md) 中也有说明。

在使用GitHub Copilot时，用户首先需要有一个 GitHub 账号。GitHub 的账号体系分为个人账号和企业账号两种类型，企业账号又分为 Enterprise、Organization 和 Team 等层级。**只有在Enterprise层级下的Orginazation下的用户，才会有 21$ 的GHEC的坐席费用。**

GitHub 的用户管理采用层级结构，从上到下依次为：

| 层级 | 说明 |
|------|------|
| **Enterprise（企业）** | 最顶层管理单元，提供统一的计费、策略管控和安全治理。一个 Enterprise 下可以管理多个 Organization。 |
| **Organization（组织）** | **可选项**，如果创建了Org，则会产生21$的额外费用。企业下的团队协作单元，用于管理代码仓库、团队和成员权限。每个 Organization 可以独立管理 Copilot 策略。 只有Org下的用户才可以开通Copilot for Enterprise。 |
| **Team（团队）** | Enterprise 或者 Organization 下的分组，用于批量管理成员权限和 Copilot 许可证分配。 |
| **User（用户）** | 具体的开发者账号，可以属于多个 Organization 和 Team。 |

**关键角色说明：**
- **Enterprise Owner（企业所有者）**：拥有企业级最高权限，可管理计费、策略、Organization 等。
- **Billing Manager（计费管理员）**：可管理企业计费设置，但无法访问代码仓库。
- **Organization Owner（组织所有者）**：管理组织内成员、仓库和 Copilot 席位分配。

**账号体系说明：**
- 默认情况下，GH的账号都属于个人账号，哪怕是用企业邮箱注册的账号，也属于个人账号。比如，`zhangsan@contoso.com` 的企业邮箱登陆GH，实际上是属于个人账号。
- GH 支持将企业的账号体系和 GH 打通，实现企业SSO账号和GH之间做映射，用户可以使用企业SSO账号登录GH。这个功能叫做 Enterprise Managed Users（EMU）。用户登陆时的用户名为 `zhangsan_contoso` ，并且没有密码的输入选项，GH会引导用户跳转到企业的SSO页面做登陆。并且这种模式下，不支持用户自己注册GH账号，管理员需要在自己的SSO平台将相关用户同步到GH上。



![GitHub 账号体系说明](ent-creation-screenshots/00.github-account-explaination.png)

