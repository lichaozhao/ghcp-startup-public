
## 管理目的和实现机制

如果管理员希望能够统一控制GH Ent内所有用户的GHCP端侧的配置项，则可以使用 `Enterprise Managed Settings` 功能。这个功能的原理是，GH Ent的管理员在 GH Ent下创建一个 Organization，并在该 Organization 下创建一个名为 `.github-private` 的仓库。然后，管理员可以在该仓库中以GH约定好的文件名称和格式做个性化配置，这些配置会被 GH 自动下发到 GH Ent内的所有用户的客户端。生效时间大约1小时。

这些配置包括很多，比如，用户使用GHCP的默认模型是什么、沙箱机制是否默认开启、GHCP的Metrics信息收集的目标服务器地址、允许/禁止哪些MCP服务等等。

代价是这个Organization下需要至少有一个用户，即，会有 $21 的GH Ent Cloud的费用，即GH Ent的坐席费，不过整个企业只需要这一个坐席即可。

## 配置方式举例

假设一个 GH Ent 下没有统一默认的模型，很多用户都默认选最贵的模型，管理员希望能够统一配置所有用户的默认模型为 `fusion`，以便于对模型不敏感的用户自动选择更合理的模型，以节省成本。其配置过程如下：

1. 在 GH Ent 下创建一个 Organization，管理员需要加入这个org，并作为这个org的默认管理员。
2. 在该 Organization 下创建一个名为 `.github-private` 的私有仓库。
3. 在该仓库中创建一个 `copilot/managed-settings.json` 的文件，并输入以下内容:
   ```json
    {
    "model": "fusion",
    "fusionTier": "high" 
    }
   ```
   配置完之后，用户自己在客户端上配置的默认模型将会被覆盖为 `fusion`。即，每次用户新打开的session默认都是 `fusion` 模式，如果客户不想用fusion，依然可以手动修改模型类型。但是即使客户在客户端配置文件里修改了`默认模型`的设置，也会在下次新打开的session时被企业管理的配置覆盖。

4. GH 会自动将这些配置下发到 GH Ent 内的所有用户的客户端，生效时间大约1小时。

## 更多配置项

除了默认模型和fusionTier之外，管理员还可以在 `.github-private` 仓库中配置其他GHCP相关的设置，而且可以和GH Ent Teams做组合，实现对不同团队的差异化配置。具体参考文档：https://docs.github.com/en/enterprise-cloud@latest/copilot/reference/enterprise-administrators/enterprise-managed-settings