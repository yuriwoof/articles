---
title: "デプロイスクリプト"
emoji: "🔧"
type: "tech"
topics: ["Azure", "Bicep", "Azure CLI", "Azure PowerShell"]
published: false
---

# はじめに

検証環境作る時にテストデータの投入させたい場合などで役立ちそうということでメモ。

デプロイ スクリプトは、テンプレート(Azure Bicep / Azure ARM Template)からAzure CLIもしくAzure PowerShellを実行できる機能。
仕組みとしては、Azure CLIもしくはAzure PowerShellがインストールされたコンテナイメージをAzure Container Instance (以降、ACI)としてデプロイし、そこから指定されたコマンドを実行するというもの。

# 使い方

この例では、Azure ファイル共有を構築して、デプロイ スクリプトでダウンロードしたjsonファイルを対象ファイル共有にアップロードする例。

```bicep
var storageAccountName = 'storage${uniqueString(resourceGroup().id)}'
var storagefileShareName = 'config'
var userAssignedIdentityName = 'configDeployer'
var roleAssignmentName = guid(resourceGroup().id, 'contributor')
var contributorRoleDefinitionId = resourceId('Microsoft.Authorization/roleDefinitions', 'b24988ac-6180-42a0-ab88-20f7382dd24c')
var deploymentScriptName = 'CopyConfigScript'
var location = resourceGroup().location

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  tags: {
    displayName: storageAccountName
  }
  location: location
  kind: 'StorageV2'
  sku: {
    name: 'Standard_LRS'
    tier: 'Standard'
  }
  properties: {
    encryption: {
      services: {
        file: {
          enabled: true
        }
      }
      keySource: 'Microsoft.Storage'
    }
    supportsHttpsTrafficOnly: true
  }

  resource fileService 'fileServices' existing = {
    name: 'default'
  }
}

resource fileShare 'Microsoft.Storage/storageAccounts/fileServices/shares@2023-01-01' = {
  parent: storageAccount::fileService
  name: storagefileShareName
  properties: {
    enabledProtocols: 'SMB'
  }
}

resource userAssignedIdentity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: userAssignedIdentityName
  location: location
}

resource roleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: roleAssignmentName
  properties: {
    roleDefinitionId: contributorRoleDefinitionId
    principalId: userAssignedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
  }
}

resource deploymentScript 'Microsoft.Resources/deploymentScripts@2020-10-01' = {
  name: deploymentScriptName
  location: location
  kind: 'AzurePowerShell'
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${userAssignedIdentity.id}': {}
    }
  }
  properties: {
    azPowerShellVersion: '10.0'
    environmentVariables: [
      {
        name: 'ResourceGroupName'
        value: resourceGroup().name
      }
      {
        name: 'StorageAccountName'
        value: storageAccountName
      }
      {
        name: 'StorageFileShareName'
        value: storagefileShareName
      }
    ]
    scriptContent: '''
      Invoke-RestMethod -Uri 'https://raw.githubusercontent.com/Azure/azure-docs-json-samples/master/mslearn-arm-deploymentscripts-sample/appsettings.json' -Out 'appsettings.json'
      $storageAccount = Get-AzStorageAccount -ResourceGroupName $env:ResourceGroupName -Name $env:StorageAccountName
      Set-AzStorageFileContent -Context $storageAccount.Context -ShareName $env:StorageFileShareName -Source ".\appsettings.json"
      $file = Get-AzStorageFile -Context $storageAccount.Context -ShareName $env:StorageFileShareName
      $DeploymentScriptOutputs = @{}
      $DeploymentScriptOutputs['Uri'] = $file.CloudFile.Uri.AbsoluteUri
    '''
    cleanupPreference: 'OnExpiration'
    retentionInterval: 'P1D'
  }
  dependsOn: [
    roleAssignment
    fileShare
  ]
}

output fileUri string = deploymentScript.properties.outputs.Uri
```

デプロイする。

```shell
$ az group create -n labbicep -l japaneast
$ az deployment group create -g labbicep --template-file ./azure-bicep/deployscript/example-setfile.bicep 
```

Azure CLIの場合はこちら

```bicep
var storageAccountName = 'storage${uniqueString(resourceGroup().id)}'
var storagefileShareName = 'config'
var userAssignedIdentityName = 'configDeployer'
var roleAssignmentName = guid(resourceGroup().id, 'contributor')
var contributorRoleDefinitionId = resourceId('Microsoft.Authorization/roleDefinitions', 'b24988ac-6180-42a0-ab88-20f7382dd24c')
var deploymentScriptName = 'CopyConfigScript'
var location = resourceGroup().location

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  tags: {
    displayName: storageAccountName
  }
  location: location
  kind: 'StorageV2'
  sku: {
    name: 'Standard_LRS'
    tier: 'Standard'
  }
  properties: {
    encryption: {
      services: {
        file: {
          enabled: true
        }
      }
      keySource: 'Microsoft.Storage'
    }
    supportsHttpsTrafficOnly: true
  }

  resource fileService 'fileServices' existing = {
    name: 'default'
  }
}

resource fileShare 'Microsoft.Storage/storageAccounts/fileServices/shares@2023-01-01' = {
  parent: storageAccount::fileService
  name: storagefileShareName
  properties: {
    enabledProtocols: 'SMB'
  }
}

resource userAssignedIdentity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: userAssignedIdentityName
  location: location
}

resource roleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: roleAssignmentName
  properties: {
    roleDefinitionId: contributorRoleDefinitionId
    principalId: userAssignedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
  }
}

resource deploymentScript 'Microsoft.Resources/deploymentScripts@2020-10-01' = {
  name: deploymentScriptName
  location: location
  kind: 'AzureCLI'
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${userAssignedIdentity.id}': {}
    }
  }
  properties: {
    azCliVersion: '2.49.0'
    environmentVariables: [
      {
        name: 'ResourceGroupName'
        value: resourceGroup().name
      }
      {
        name: 'StorageAccountName'
        value: storageAccountName
      }
      {
        name: 'StorageFileShareName'
        value: storagefileShareName
      }
    ]
    scriptContent: '''
      wget 'https://raw.githubusercontent.com/Azure/azure-docs-json-samples/master/mslearn-arm-deploymentscripts-sample/appsettings.json' -o 'appsettings.json'
      az storage file upload --account-name $StorageAccountName --share-name $StorageFileShareName --source "appsettings.json" --only-show-errors
      URL=$(az storage file url --account-name $StorageAccountName --share-name $StorageFileShareName -p "appsettings.json" --only-show-errors)
      echo "{\"url\":"$URL", \"storageaccount\":\"$StorageAccountName\", \"fileshare\":\"$StorageFileShareName\"}" > $AZ_SCRIPTS_OUTPUT_PATH
    '''
    cleanupPreference: 'OnExpiration'
    retentionInterval: 'P1D'
  }
  dependsOn: [
    roleAssignment
    fileShare
  ]
}

output fileUri string = deploymentScript.properties.outputs.url
```

デプロイする。

```shell
$ az group create -n labbicep -l japaneast
$ az deployment group create -g labbicep --template-file ./azure-bicep/deployscript/example-setfilecli.bicep 
```

## 認証

デプロイスクリプトを実行する際には、２種類の権限について考慮する必要がある。

1つ目が"デプロイ プリンシパル"で、デプロイ スクリプトを実行するユーザーやサービスプリンシパルにあたる。
これは、Bicepのデプロイ権限と合わせ、 Azure Container Instanceやストレージ アカウントを作成する際の権限を持つ必要がある

もう一方が、デプロイ スクリプト プリンシパルで、これはデプロイ スクリプト内で Azure CLI/Azure PowerShellを呼ぶ際に用いられる。
変数としてサービス プリンシパルを渡せるけど、ユーザー割り当てマネージドIDを用いた方が簡単。

なお、どちらも省略した場合、１つ目のデプロイ プリンシパルが使われることを期待したが、
そうはならず、認証に失敗してしまう。

```bash
$ az deployment group create -g labbicep --template-file ./azure-bicep/deployscript/example-setblob.bicep 
...(省略)...
{"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/labbicep/providers/Microsoft.Resources/deployments/example-setblob","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/labbicep/providers/Microsoft.Resources/deploymentScripts/CopyConfigScript","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'failed'.","details":[{"code":"DeploymentScriptError","message":"The provided script failed with multiple errors. First error:\r\nMicrosoft.Azure.Commands.Common.Exceptions.AzPSApplicationException: No subscription found in the context.  Please ensure that the credentials you provided are authorized to access an Azure subscription, then run Connect-AzAccount to login.
```

## 出力の受け渡し

PowerShellの場合は、実行するスクリプト内では、連想配列 ```$DeploymentScriptOutputs``` を定義し、
そこに出力に出したい値を入れてあげる。

```powershell
      $DeploymentScriptOutputs = @{}
      $DeploymentScriptOutputs['Uri'] = $file.CloudFile.Uri.AbsoluteUri
```

Bicepの方では、取り出したい出力分 ```output``` で定義し、
Azure CLIからBicepテンプレートを実行した場合、以下のように参照できる。

```bicep
output fileUri string = deploymentScript.properties.outputs.Uri
```

```shell
$ az deployment group show --resource-group labbicep --name 'example-setfile' --query 'properties.outputs.fileUri.value' --output tsv
https://xxx.file.core.windows.net/config/appsettings.json
```

もしくは

```shell
$ az deployment-scripts show --resource-group 'labbicep' --name 'CopyConfigScript' --query 'outputs.Uri' -o tsv
https://xxx.file.core.windows.net/config/appsettings.json
```

Azure CLIの場合にはPowerShellのように連想配列の用意がなく、```$AZ_SCRIPTS_OUTPUT_PATH```にリダイレクトする。
この環境変数は、```/mnt/azscripts/azscriptoutput/scriptoutputs.json```になっていて、これがパースされるみたい。

ドキュメントの例では、配列で確認しているが、そうでなくても取得できた。

[CLI スクリプトからの出力を操作する](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/deployment-script-bicep#work-with-outputs-from-cli-script)

>デプロイ スクリプトの出力は AZ_SCRIPTS_OUTPUT_PATH の場所に保存される必要があり、その出力は有効な JSON 文字列オブジェクトでなければなりません。 
>ファイルの内容は、キーと値のペアとして保存される必要があります。 
>たとえば、文字列の配列は、{ "MyResult": [ "foo", "bar"] } として格納されます。 配列の結果のみ ([ "foo", "bar" ] など) の格納は、無効です。

```bicep
echo "{\"url\":"$URL", \"storageaccount\":\"$StorageAccountName\", \"fileshare\":\"$StorageFileShareName\"}" > $AZ_SCRIPTS_OUTPUT_PATH
```

```bicep
output fileUri string = deploymentScript.properties.outputs.url
```

```shell
$ az deployment group show --resource-group labbicep --name 'example-setfilecli' --query 'properties.outputs.fileUri.value' --output tsv
https://xxx.file.core.windows.net/config/appsettings.json

$ az deployment-scripts show --resource-group 'labbicep' --name 'CopyConfigScript' --query 'outputs.url' -o tsv
https://xxx.file.core.windows.net/config/appsettings.json
```

![Azure ポータルよりデプロイ スクリプトの出力を確認](/images/4baff5ef09c6e3/img.png)

## 制限

現時点の制限としては、仮想ネットワークにデプロイができないため、
例えばプライベート エンドポイントに制限したリソースへのアクセスが行えない。

* [プライベート仮想ネットワークへのアクセス](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/deployment-script-bicep#access-private-virtual-network)


## ローカルでテスト

ACIで参照しているコンテナイメージは公開されているので、デプロイスクリプトを作成するにあたり、ローカルでまずは試してから行う。
ただし、デプロイスクリプトでは利用できる環境変数は用意されていないので、その点注意。

```powershell
PS C:\Users\yurio> docker pull mcr.microsoft.com/azure-cli:2.49.0
PS C:\> docker run -v c:/docker:/data -it mcr.microsoft.com/azure-cli:2.49.0
b17727d89c49:/#
```

* [Docker を使用する](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/templates/deployment-script-template-configure-dev#use-docker)

## 課金

課金に際しては自動生成されるACIとストレージ アカウント (既存のストレージ アカウントを使用する場合、作成されるファイル共有)に対する課金が行われる。
成功時にそれらを消したい場合、```cleanupPreference```は、```OnSuccess```と指定しておくと、トラブルシューティング時に実行されたスクリプトや出力が残るのでこの指定が良いかも。

```bicep
    cleanupPreference: 'OnSuccess'
    retentionInterval: 'P1D'
```

# 参考文献

* [デプロイ スクリプトを使用して Bicep および ARM テンプレートを拡張する](https://learn.microsoft.com/ja-jp/training/modules/extend-resource-manager-template-deployment-scripts/)
* [Bicep でデプロイ スクリプトを使う](https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/deployment-script-bicep#clean-up-deployment-script-resources)
