---
title: ASP.NET Core 中的 Azure Key Vault 配置提供程序
author: guardrex
description: 了解如何使用 Azure Key Vault 配置提供程序通过在运行时加载的名称/值对来配置应用。
monikerRange: '>= aspnetcore-2.1'
ms.author: riande
ms.custom: mvc
ms.date: 11/14/2019
uid: security/key-vault-configuration
ms.openlocfilehash: e0e55d40734e0cb6e3e1afe1c708ec47c6f43054
ms.sourcegitcommit: f91d322f790123d41ec3271fa084ae20ed9f89a6
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 11/18/2019
ms.locfileid: "74155172"
---
# <a name="azure-key-vault-configuration-provider-in-aspnet-core"></a>ASP.NET Core 中的 Azure Key Vault 配置提供程序

作者： [Luke Latham](https://github.com/guardrex)和[Andrew Stanton](https://github.com/anurse)

本文档说明如何使用[Microsoft Azure Key Vault](https://azure.microsoft.com/services/key-vault/)配置提供程序从 Azure Key Vault 机密加载应用配置值。 Azure Key Vault 是一项基于云的服务，可帮助保护应用程序和服务使用的加密密钥和机密。 将 Azure Key Vault 用于 ASP.NET Core 应用的常见方案包括：

* 控制对敏感配置数据的访问。
* 在存储配置数据时满足 FIPS 140-2 级别2验证的硬件安全模块（HSM）的要求。

[查看或下载示例代码](https://github.com/aspnet/AspNetCore.Docs/tree/master/aspnetcore/security/key-vault-configuration/samples)（[如何下载](xref:index#how-to-download-a-sample)）

## <a name="packages"></a>package

将包引用添加到[AzureKeyVault](https://www.nuget.org/packages/Microsoft.Extensions.Configuration.AzureKeyVault/)包。

## <a name="sample-app"></a>示例应用

该示例应用在*Program.cs*文件顶部的 `#define` 语句确定的两种模式中运行：

* `Certificate` &ndash; 演示了如何使用 Azure Key Vault 的客户端 ID 和 x.509 证书来访问存储在 Azure Key Vault 中的机密。 可以从部署到 Azure App Service 的任何位置运行此版本的示例，也可以从部署到 ASP.NET Core 应用程序的任何主机上运行。
* `Managed` &ndash; 演示了如何使用[Azure 资源的托管标识](/azure/active-directory/managed-identities-azure-resources/overview)对应用进行身份验证，以使用 Azure AD 身份验证 Azure Key Vault，而不会在应用的代码或配置中存储凭据。 使用托管标识进行身份验证时，不需要 Azure AD 的应用程序 ID 和密码（客户端机密）。 必须将示例的 `Managed` 版本部署到 Azure。 按照[使用 Azure 资源的托管标识](#use-managed-identities-for-azure-resources)部分中的指导进行操作。

有关如何使用预处理器指令（`#define`）配置示例应用的详细信息，请参阅 <xref:index#preprocessor-directives-in-sample-code>。

## <a name="secret-storage-in-the-development-environment"></a>开发环境中的机密存储

使用[机密管理器工具](xref:security/app-secrets)在本地设置机密。 当在开发环境中的本地计算机上运行示例应用时，会从本地密钥管理器存储中加载密码。

机密管理器工具需要应用的项目文件中的 `<UserSecretsId>` 属性。 将属性值（`{GUID}`）设置为任何唯一 GUID：

```xml
<PropertyGroup>
  <UserSecretsId>{GUID}</UserSecretsId>
</PropertyGroup>
```

机密以名称-值对的形式创建。 分层值（配置节）使用 `:` （冒号）作为[ASP.NET Core 配置](xref:fundamentals/configuration/index)项名称中的分隔符。

机密管理器是从打开到项目[内容根目录](xref:fundamentals/index#content-root)的命令 shell 使用的，其中 `{SECRET NAME}` 是名称，`{SECRET VALUE}` 是值：

```dotnetcli
dotnet user-secrets set "{SECRET NAME}" "{SECRET VALUE}"
```

从项目的[内容根](xref:fundamentals/index#content-root)在命令外壳中执行以下命令，为示例应用设置机密：

```dotnetcli
dotnet user-secrets set "SecretName" "secret_value_1_dev"
dotnet user-secrets set "Section:SecretName" "secret_value_2_dev"
```

如果这些机密存储在[生产环境中的机密存储 Azure Key Vault 中，并具有 Azure Key Vault](#secret-storage-in-the-production-environment-with-azure-key-vault)部分，则 `_dev` 后缀将更改为 `_prod`。 后缀在应用的输出中提供视觉提示，指示配置值的源。

## <a name="secret-storage-in-the-production-environment-with-azure-key-vault"></a>Azure Key Vault 中的生产环境中的机密存储

本[快速入门：使用 Azure CLI 主题的 Azure Key Vault 设置和检索机密](/azure/key-vault/quick-create-cli)的说明，请参阅此处，以创建 Azure Key Vault 和存储示例应用使用的机密。 有关更多详细信息，请参阅主题。

1. 使用[Azure 门户](https://portal.azure.com/)中的以下方法之一打开 Azure Cloud shell：

   * 选择代码块右上角的 "**试用**"。 在文本框中使用搜索字符串 "Azure CLI"。
   * 在浏览器中打开 Cloud Shell，并提供 "**启动 Cloud Shell** " 按钮。
   * 选择 Azure 门户右上角菜单中的 " **Cloud Shell** " 按钮。

   有关详细信息，请参阅[Azure 命令行接口（CLI）](/cli/azure/)和[Azure Cloud Shell 概述](/azure/cloud-shell/overview)。

1. 如果尚未通过身份验证，请在 `az login` 命令中登录。

1. 使用以下命令创建资源组，其中 `{RESOURCE GROUP NAME}` 是新资源组的资源组名称，`{LOCATION}` 是 Azure 区域（datacenter）：

   ```azure-cli
   az group create --name "{RESOURCE GROUP NAME}" --location {LOCATION}
   ```

1. 使用以下命令在资源组中创建密钥保管库，其中 `{KEY VAULT NAME}` 是新密钥保管库的名称，`{LOCATION}` 是 Azure 区域（datacenter）：

   ```azure-cli
   az keyvault create --name "{KEY VAULT NAME}" --resource-group "{RESOURCE GROUP NAME}" --location {LOCATION}
   ```

1. 在密钥保管库中，以名称-值对的形式创建机密。

   Azure Key Vault 机密名称仅限字母数字字符和短划线。 分层值（配置节）使用 `--` （两个短划线）作为分隔符。 冒号（通常用于在[ASP.NET Core 配置](xref:fundamentals/configuration/index)中的子项中分隔部分）在 key vault 机密名称中不允许使用。 因此，当机密加载到应用的配置中时，将使用两个短划线并为冒号交换。

   以下机密用于示例应用。 这些值包括一个 `_prod` 后缀，以将其与从用户机密的开发环境中加载的 `_dev` 后缀值区分开来。 将 `{KEY VAULT NAME}` 替换为在上一步中创建的密钥保管库的名称：

   ```azure-cli
   az keyvault secret set --vault-name "{KEY VAULT NAME}" --name "SecretName" --value "secret_value_1_prod"
   az keyvault secret set --vault-name "{KEY VAULT NAME}" --name "Section--SecretName" --value "secret_value_2_prod"
   ```

## <a name="use-application-id-and-x509-certificate-for-non-azure-hosted-apps"></a>为非 Azure 托管的应用使用应用程序 ID 和 x.509 证书

配置 Azure AD、Azure Key Vault 和应用，以便在**Azure 外托管应用时**使用 Azure Active Directory 应用程序 ID 和 x.509 证书对密钥保管库进行身份验证。 有关详细信息，请参阅[关于密钥、机密和证书](/azure/key-vault/about-keys-secrets-and-certificates)。

> [!NOTE]
> 尽管在 Azure 中托管的应用程序支持使用应用程序 ID 和 x.509 证书，但在 Azure 中托管应用程序时，我们建议使用[Azure 资源的托管标识](#use-managed-identities-for-azure-resources)。 托管标识不需要在应用或开发环境中存储证书。

当*Program.cs*文件顶部的 `#define` 语句设置为 `Certificate` 时，示例应用使用应用程序 ID 和 x.509 证书。

1. 创建 PKCS # 12 存档（ *.pfx*）证书。 用于创建证书的选项包括 Windows 和[OpenSSL](https://www.openssl.org/)[上的 MakeCert](/windows/desktop/seccrypto/makecert) 。
1. 将证书安装到当前用户的个人证书存储中。 将密钥标记为可导出是可选的。 记下证书的指纹，此过程将在此过程中使用。
1. 将 PKCS # 12 存档（ *.pfx*）证书导出为 DER 编码的证书（ *.cer*）。
1. 向 Azure AD （**应用注册**）注册应用。
1. 将 DER 编码的证书（ *.cer*）上载到 Azure AD：
   1. 在 Azure AD 中选择应用。
   1. 导航到 "**证书" & "机密**"。
   1. 选择 "**上传证书**"，上传包含公钥的证书。 *.Cer*、 *pem*或 *.crt*证书是可接受的。
1. 将密钥保管库名称、应用程序 ID 和证书指纹存储在应用的*appsettings*文件中。
1. 导航到 Azure 门户中的**密钥保管库**。
1. 选择在[生产环境中的机密存储中](#secret-storage-in-the-production-environment-with-azure-key-vault)创建的密钥保管库，其中 Azure Key Vault "部分。
1. 选择 "**访问策略**"。
1. 选择 "**添加访问策略**"。
1. 打开**机密权限**，并为应用提供**Get**和**List**权限。
1. 选择 "**选择主体**"，并按名称选择注册的应用。 选择 "**选择**" 按钮。
1. 选择“确定”。
1. 选择“保存”。
1. 部署应用程序。

`Certificate` 示例应用从 `IConfigurationRoot` 中获取其配置值，名称与机密名称相同：

* 非分层值： `SecretName` 的值是通过 `config["SecretName"]`获取的。
* 分层值（节）：使用 `:` （冒号）表示法或 `GetSection` 扩展方法。 使用以下任一方法来获取配置值：
  * `config["Section:SecretName"]`
  * `config.GetSection("Section")["SecretName"]`

X.509 证书由操作系统管理。 应用使用*appsettings*文件提供的值调用 <xref:Microsoft.Extensions.Configuration.AzureKeyVaultConfigurationExtensions.AddAzureKeyVault*>：

::: moniker range=">= aspnetcore-3.0"

[!code-csharp[](key-vault-configuration/samples/3.x/SampleApp/Program.cs?name=snippet1&highlight=20-23)]

::: moniker-end

::: moniker range="< aspnetcore-3.0"

[!code-csharp[](key-vault-configuration/samples/2.x/SampleApp/Program.cs?name=snippet1&highlight=20-23)]

::: moniker-end

示例值：

* 密钥保管库名称： `contosovault`
* 应用程序 ID： `627e911e-43cc-61d4-992e-12db9c81b413`
* 证书指纹： `fe14593dd66b2406c5269d742d04b6e1ab03adb1`

appsettings.json：

::: moniker range=">= aspnetcore-3.0"

[!code-json[](key-vault-configuration/samples/3.x/SampleApp/appsettings.json?highlight=10-12)]

::: moniker-end

::: moniker range="< aspnetcore-3.0"

[!code-json[](key-vault-configuration/samples/2.x/SampleApp/appsettings.json?highlight=10-12)]

::: moniker-end

运行应用时，网页将显示加载的机密值。 在开发环境中，用 `_dev` 后缀加载机密值。 在生产环境中，值以 `_prod` 后缀进行加载。

## <a name="use-managed-identities-for-azure-resources"></a>使用 Azure 资源的托管标识

**部署到 azure 的应用**可以利用[Azure 资源的托管标识](/azure/active-directory/managed-identities-azure-resources/overview)，这允许应用使用 Azure AD 身份验证而无需凭据（应用程序 ID 和密码/客户端密码）进行身份验证 Azure Key Vault存储在应用中。

当*Program.cs*文件顶部的 `#define` 语句设置为 `Managed` 时，示例应用使用 Azure 资源的托管标识。

在应用的*appsettings*文件中输入保管库名称。 设置为 `Managed` 版本时，示例应用不需要应用程序 ID 和密码（客户端密码），因此可以忽略这些配置条目。 应用将部署到 Azure，Azure 将对应用进行身份验证，以便仅使用存储在*appsettings*文件中的保管库名称访问 Azure Key Vault。

将示例应用部署到 Azure App Service。

在创建服务时，会自动向 Azure AD 注册部署到 Azure App Service 的应用。 从部署中获取对象 ID 以在下面的命令中使用。 对象 ID 显示在应用服务的 "**标识**" 面板上的 "Azure 门户中。

使用 Azure CLI 和应用程序的对象 ID，为应用程序提供 `list` 和 `get` 权限，以访问密钥保管库：

```azure-cli
az keyvault set-policy --name '{KEY VAULT NAME}' --object-id {OBJECT ID} --secret-permissions get list
```

使用 Azure CLI、PowerShell 或 Azure 门户**重启应用**。

示例应用：

* 创建不带连接字符串的 `AzureServiceTokenProvider` 类的实例。 如果未提供连接字符串，提供程序将尝试从 Azure 资源的托管标识获取访问令牌。
* 使用 `AzureServiceTokenProvider` 实例令牌回调创建新 <xref:Microsoft.Azure.KeyVault.KeyVaultClient>。
* <xref:Microsoft.Azure.KeyVault.KeyVaultClient> 实例与加载所有机密值的 <xref:Microsoft.Extensions.Configuration.AzureKeyVault.IKeyVaultSecretManager> 的默认实现一起使用，并将键名称中的双破折号（`--`）替换为冒号（`:`）。

::: moniker range=">= aspnetcore-3.0"

[!code-csharp[](key-vault-configuration/samples/3.x/SampleApp/Program.cs?name=snippet2&highlight=13-21)]

::: moniker-end

::: moniker range="< aspnetcore-3.0"

[!code-csharp[](key-vault-configuration/samples/2.x/SampleApp/Program.cs?name=snippet2&highlight=13-21)]

::: moniker-end

Key vault 名称示例值： `contosovault`
    
appsettings.json：

```json
{
  "KeyVaultName": "Key Vault Name"
}
```

运行应用时，网页将显示加载的机密值。 在开发环境中，机密值具有 `_dev` 后缀，因为它们是由用户机密提供的。 在生产环境中，值使用 `_prod` 后缀加载，因为它们由 Azure Key Vault 提供。

如果收到 `Access denied` 错误，请确认已向应用程序注册了 Azure AD 并提供了对密钥保管库的访问权限。 确认已在 Azure 中重新启动该服务。

有关将提供程序与托管标识和 Azure DevOps 管道一起使用的信息，请参阅使用[托管服务标识创建与 VM 的 Azure 资源管理器服务连接](/azure/devops/pipelines/library/connect-to-azure#create-an-azure-resource-manager-service-connection-to-a-vm-with-a-managed-service-identity)。

::: moniker range=">= aspnetcore-3.0"

## <a name="configuration-options"></a>配置选项

<xref:Microsoft.Extensions.Configuration.AzureKeyVaultConfigurationExtensions.AddAzureKeyVault*> 可以接受 <xref:Microsoft.Extensions.Configuration.AzureKeyVault.AzureKeyVaultConfigurationOptions>：

```csharp
config.AddAzureKeyVault(
    new AzureKeyVaultConfigurationOptions()
    {
        ...
    });
```

| Property         | 描述 |
| ---------------- | ----------- |
| `Client`         | 用于检索值的 <xref:Microsoft.Azure.KeyVault.KeyVaultClient>。 |
| `Manager`        | 用于控制机密加载的 <xref:Microsoft.Extensions.Configuration.AzureKeyVault.IKeyVaultSecretManager> 实例。 |
| `ReloadInterval` | `Timespan` 在轮询密钥保管库进行更改时等待。 默认值为 `null` （配置不会重新加载）。 |
| `Vault`          | 密钥保管库 URI。 |

::: moniker-end

## <a name="use-a-key-name-prefix"></a>使用密钥名称前缀

<xref:Microsoft.Extensions.Configuration.AzureKeyVaultConfigurationExtensions.AddAzureKeyVault*> 提供一个重载，该重载接受 <xref:Microsoft.Extensions.Configuration.AzureKeyVault.IKeyVaultSecretManager>的实现，这允许你控制密钥保管库机密如何转换为配置密钥。 例如，你可以实现接口，以便基于在应用启动时提供的前缀值加载机密值。 例如，你可以根据应用程序的版本加载机密。

> [!WARNING]
> 不要对 key vault 机密使用前缀，将多个应用的机密放入同一密钥保管库，或将环境机密（例如，*开发*与*生产*机密）放入同一保管库中。 建议不同的应用和开发/生产环境使用不同的密钥保管库，以便隔离应用环境以获得最高级别的安全性。

在下面的示例中，在密钥保管库中建立了机密（并使用开发环境的机密管理器工具）进行 `5000-AppSecret` （密钥保管库机密名称中不允许使用句点）。 此机密代表应用程序版本5.0.0.0 的应用程序机密。 对于其他版本的应用，5.1.0.0，会将机密添加到密钥保管库（并使用机密管理器工具）进行 `5100-AppSecret`。 每个应用版本会将其版本控制的机密值作为 `AppSecret` 加载到其配置中，并在加载机密时将版本去除。

使用自定义 <xref:Microsoft.Extensions.Configuration.AzureKeyVault.IKeyVaultSecretManager>调用 <xref:Microsoft.Extensions.Configuration.AzureKeyVaultConfigurationExtensions.AddAzureKeyVault*>：

[!code-csharp[](key-vault-configuration/samples_snapshot/Program.cs)]

<xref:Microsoft.Extensions.Configuration.AzureKeyVault.IKeyVaultSecretManager> 实现将对机密的版本前缀做出反应，以将适当的机密加载到配置中：

* `Load` 在其名称以前缀开头时加载密码。 未加载其他机密。
* `GetKey`：
  * 从机密名称中删除前缀。
  * 用 `KeyDelimiter`替换任意名称中的两个短划线，这是在配置中使用的分隔符（通常为冒号）。 Azure Key Vault 不允许在机密名称中使用冒号。

[!code-csharp[](key-vault-configuration/samples_snapshot/Startup.cs)]

`Load` 方法由提供程序算法调用，该算法会循环访问保管库机密，查找具有版本前缀的文件。 在 `Load` 找到版本前缀后，该算法使用 `GetKey` 方法返回密钥名称的配置名称。 它从机密名称中去除版本前缀，并返回密钥名称的其余部分，以便加载到应用的配置名称-值对中。

实现此方法时：

1. 应用的项目文件中指定的应用版本。 在以下示例中，应用的版本设置为 `5.0.0.0`：

   ```xml
   <PropertyGroup>
     <Version>5.0.0.0</Version>
   </PropertyGroup>
   ```

1. 确认应用的项目文件中存在 `<UserSecretsId>` 属性，其中 `{GUID}` 是用户提供的 GUID：

   ```xml
   <PropertyGroup>
     <UserSecretsId>{GUID}</UserSecretsId>
   </PropertyGroup>
   ```

   用[机密管理器工具](xref:security/app-secrets)本地保存以下机密：

   ```dotnetcli
   dotnet user-secrets set "5000-AppSecret" "5.0.0.0_secret_value_dev"
   dotnet user-secrets set "5100-AppSecret" "5.1.0.0_secret_value_dev"
   ```

1. 使用以下 Azure CLI 命令在 Azure Key Vault 中保存机密：

   ```azure-cli
   az keyvault secret set --vault-name "{KEY VAULT NAME}" --name "5000-AppSecret" --value "5.0.0.0_secret_value_prod"
   az keyvault secret set --vault-name "{KEY VAULT NAME}" --name "5100-AppSecret" --value "5.1.0.0_secret_value_prod"
   ```

1. 当应用运行时，将加载密钥保管库机密。 `5000-AppSecret` 的字符串机密与应用程序的项目文件（`5.0.0.0`）中指定的应用程序版本相匹配。

1. 将从键名称中去除版本 `5000` （带有短划线）。 在整个应用程序中，读取带有密钥的配置 `AppSecret` 加载机密值。

1. 如果在项目文件中将应用程序的版本更改为 `5.1.0.0` 并再次运行该应用程序，则返回的机密值将 `5.1.0.0_secret_value_dev` 在开发环境中，并 `5.1.0.0_secret_value_prod` 生产环境中。

> [!NOTE]
> 你还可以提供自己的 <xref:Microsoft.Azure.KeyVault.KeyVaultClient> 实现以 <xref:Microsoft.Extensions.Configuration.AzureKeyVaultConfigurationExtensions.AddAzureKeyVault*>。 自定义客户端允许跨应用共享客户端的单个实例。

## <a name="bind-an-array-to-a-class"></a>将数组绑定至类

提供程序能够将配置值读入数组，以便绑定到 POCO 数组。

从允许键包含冒号（`:`）分隔符的配置源中进行读取时，将使用数字键段来区分组成数组的键（`:0:`、`:1:` 。 `:{n}:`) 格式模式中出现的位置匹配。 有关详细信息，请参阅[配置：将数组绑定到类](xref:fundamentals/configuration/index#bind-an-array-to-a-class)。

Azure Key Vault 密钥不能使用冒号作为分隔符。 本主题中所述的方法使用双短划线（`--`）作为层次结构值（节）的分隔符。 数组键存储在具有双短划线和数值段（`--0--`、`--1--` &hellip; `--{n}--`） Azure Key Vault 中。

检查 JSON 文件提供的以下[Serilog](https://serilog.net/)日志记录提供程序配置。 `WriteTo` 数组中定义了两个对象文本，该对象反映了两个 Serilog*接收器*，它们描述了日志记录输出的目标：

```json
"Serilog": {
  "WriteTo": [
    {
      "Name": "AzureTableStorage",
      "Args": {
        "storageTableName": "logs",
        "connectionString": "DefaultEnd...ountKey=Eby8...GMGw=="
      }
    },
    {
      "Name": "AzureDocumentDB",
      "Args": {
        "endpointUrl": "https://contoso.documents.azure.com:443",
        "authorizationKey": "Eby8...GMGw=="
      }
    }
  ]
}
```

使用双短划线（`--`）表示法和数值段 Azure Key Vault 前面的 JSON 文件中所示的配置：

| 项 | “值” |
| --- | ----- |
| `Serilog--WriteTo--0--Name` | `AzureTableStorage` |
| `Serilog--WriteTo--0--Args--storageTableName` | `logs` |
| `Serilog--WriteTo--0--Args--connectionString` | `DefaultEnd...ountKey=Eby8...GMGw==` |
| `Serilog--WriteTo--1--Name` | `AzureDocumentDB` |
| `Serilog--WriteTo--1--Args--endpointUrl` | `https://contoso.documents.azure.com:443` |
| `Serilog--WriteTo--1--Args--authorizationKey` | `Eby8...GMGw==` |

## <a name="reload-secrets"></a>重载机密

在调用 `IConfigurationRoot.Reload()` 之前会缓存机密。 在执行 `Reload` 之前，应用程序不会考虑到密钥保管库中已过期、已禁用和更新的机密。

```csharp
Configuration.Reload();
```

## <a name="disabled-and-expired-secrets"></a>禁用和过期的机密

禁用和过期的机密会引发 <xref:Microsoft.Azure.KeyVault.Models.KeyVaultErrorException>。 若要防止应用程序引发，请使用不同的配置提供程序提供配置，或更新禁用或过期的机密。

## <a name="troubleshoot"></a>疑难解答

当应用无法使用访问接口加载配置时，会将错误消息写入[ASP.NET Core 日志记录基础结构](xref:fundamentals/logging/index)。 以下条件将阻止加载配置：

* Azure Active Directory 中未正确配置应用或证书。
* Azure Key Vault 中不存在密钥保管库。
* 应用无权访问密钥保管库。
* 访问策略不包括 `Get` 和 `List` 权限。
* 在密钥保管库中，配置数据（名称-值对）被错误地命名、缺失、禁用或过期。
* 应用具有错误的密钥保管库名称（`KeyVaultName`）、Azure AD 应用程序 Id （`AzureADApplicationId`）或 Azure AD 证书指纹（`AzureADCertThumbprint`）。
* 在应用程序中，配置项（名称）的值不正确。
* 向密钥保管库添加应用的访问策略时，策略已创建，但未在**访问策略**UI 中选择 "**保存**" 按钮。

## <a name="additional-resources"></a>其他资源

* <xref:fundamentals/configuration/index>
* [Microsoft Azure： Key Vault](https://azure.microsoft.com/services/key-vault/)
* [Microsoft Azure： Key Vault 文档](/azure/key-vault/)
* [如何为 Azure Key Vault 生成和传输受 HSM 保护的密钥](/azure/key-vault/key-vault-hsm-protected-keys)
* [KeyVaultClient 类](/dotnet/api/microsoft.azure.keyvault.keyvaultclient)
* [快速入门：使用 .NET web 应用设置和检索 Azure Key Vault 中的机密](/azure/key-vault/quick-create-net)
* [教程：如何在 .NET 中使用 Azure Windows 虚拟机 Azure Key Vault](/azure/key-vault/tutorial-net-windows-virtual-machine)
