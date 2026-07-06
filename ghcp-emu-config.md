# GH EMU的工作原理

1. 在SSO账号集成方案里，一般用到两个协议：使用SCIM协议同步用户信息；使用SAML协议做SSO验证。
2. GH的EMU也是采取这种方式，具体过程为：
   - 在GH ENT创建时，选择EMU模式
   - EMU模式下，GH Ent会准备好配置做SAML认证的配置页面，并且提供SCIM的API接口
   - IdP侧（比如Azure Entra ID），需要可以配置GH提供的SCIM的API接口，配置时需要使用在GH ENT里生成的 PAT （personal access token）
   - IdP侧还需要启用基于 SAML 的SSO功能，启用时会包含，SAML的登录URL、SAML的Issuer URL、SAML的证书
   - 要在GH的SSO配置页面，填入 IdP的登录URL、SAML的Issuer URL、SAML的证书 这三个值
3. IdP 推荐使用Azure Entra ID。也支持自定义的IdP。

# 具体步骤
1. 登陆GH创建EMU实体
2. 在azure entra id创建 gh ent app，目的是通过这个enterprise app将Entra ID和GH做SSO集成。
3. 配置 ghent app 的 SSO页面。推荐选SAML（非OIDC）。主要配置identifier, Reply URL, Sign-on URL[参考文档](https://learn.microsoft.com/en-us/entra/identity/saas-apps/github-enterprise-managed-user-tutorial)
4. 从ghent app下载证书备用
5. 回到GH EMU里配置 的identity provider的SSO页面，sign on URL填 Microsoft Entra ID的属性，issuer填 Microsoft Entra Identifier，证书复制下载的证书内容
6. 在GH里创建一个PAT，权限scim:enterprise
7. 回到az ad里给gh ent app的provisioning配置 tenant url（gh的ent 地址https://api.github.com/scim/v2/enterprises/{enterprise} ），然后填入PAT，PAT权限scim:enterprise
8. 配置完成，选用户provision即可。