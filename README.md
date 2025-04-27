# AGIC (Application Gateway Ingress Controller)

Azure Application Gateway Ingress Controller (AGIC) のアドオンを使用したAKS環境の構築手順です。
アドオンベースの実装により、AGICの管理がシンプルになり、Azureによって自動的に更新が行われます。

## 前提条件

- Azure CLIがインストールされていること
- Azureサブスクリプションへのアクセス権があること
- kubectlがインストールされていること

## 環境変数の設定

```bash
# サブスクリプションID
SUBSCRIPTION_ID='your-subscription-id'

# リソース名の設定
AKS_NAME='your-aks-name'
RESOURCE_GROUP='your-resource-group'
LOCATION='japaneast'
```

## 1. Azureへのログインと設定

```bash
# Azureへログイン
az login

# サブスクリプションの設定
az account set --subscription $SUBSCRIPTION_ID
```

## 2. リソースグループとAKSクラスターの作成

```bash
# リソースグループの作成
az group create --name $RESOURCE_GROUP --location $LOCATION

# AGICアドオン付きAKSクラスターの作成
az aks create \
  -n $AKS_NAME \
  -g $RESOURCE_GROUP \
  --network-plugin azure \
  --enable-managed-identity \
  -a ingress-appgw \  # AGICアドオンを有効化
  --appgw-name myApplicationGateway \  # Application Gatewayの名前
  --appgw-subnet-cidr "10.225.0.0/16" \  # Application Gateway用サブネット
  --generate-ssh-keys

# Azure Load Balancerを使用したAKSクラスターの作成
az aks create \
  -n $AKS_NAME \
  -g $RESOURCE_GROUP \
  --location $LOCATION \
  --network-plugin azure \
  --enable-managed-identity \
  --load-balancer-sku standard \
  --generate-ssh-keys
```




## 3. AGICアドオンの権限設定

```bash
# Application Gateway IDの取得
appGatewayId=$(az aks show -n $AKS_NAME -g $RESOURCE_GROUP -o tsv --query "addonProfiles.ingressApplicationGateway.config.effectiveApplicationGatewayId")

# Application Gateway サブネットIDの取得
appGatewaySubnetId=$(az network application-gateway show --ids $appGatewayId -o tsv --query "gatewayIPConfigurations[0].subnet.id")

# AGICアドオンIDの取得
agicAddonIdentity=$(az aks show -n $AKS_NAME -g $RESOURCE_GROUP -o tsv --query "addonProfiles.ingressApplicationGateway.identity.clientId")

# AGICアドオンにNetwork Contributor権限を付与
az role assignment create \
  --assignee $agicAddonIdentity \
  --scope $appGatewaySubnetId \
  --role "Network Contributor"
```

## 4. クラスターの認証情報の取得

```bash
az aks get-credentials -n $AKS_NAME -g $RESOURCE_GROUP
```

## アドオンベース実装の利点

- マネージドソリューションとして提供
- Azureによる自動更新と保守
- AKSクラスターとの統合が容易
- 追加のPodやリソースの手動管理が不要
- Application Gatewayのライフサイクル管理の簡素化

## 注意点

- Application Gatewayの作成には10-15分程度かかることがあります
- サブネットCIDRは既存のネットワークと重複しないように設定してください
- AGICアドオンは、作成時に指定するか、既存のAKSクラスターに後から追加することも可能です
- Application GatewayとAKSクラスターは同じVNet内に存在する必要があります