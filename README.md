# azure-starter

resourcegroup=azs-rg
appName=azs-app

cd arm
azure group create -f storage.json -e storage.parameters.json azs-rg japanwes
azure group deployment list azs-rg

storageaccount=$(azure storage account list -g $resourcegroup --json | jq -r '.[0].name')
keyvault=$(azure keyvault list $resourcegroup --json | jq -r '.[0].name')

pass=$(pwgen -s1 16)
azure keyvault secret set -w $pass $keyvault $appName

pass=$(azure keyvault secret show $keyvault $appName --json | jq -r '.value')
url=$(azure storage account show -g $resourcegroup $storageaccount --json | jq -r '.primaryEndpoints.blob')

azure ad app create -n $appName -m $url -i $url -p $pass -v
appId=$(azure ad app show -i $url --json | jq -r '.[0].appId')

azure ad sp create -a $appId
spId=$(azure ad sp show -n $appId --json | jq -r '.[0].objectId')

subscriptionId=$(azure account show --json | jq -r '.[0].id')
azure role assignment create --objectId $spId -o Contributor -c /subscriptions/$subscriptionId/

tenantId=$(azure account show --json | jq -r '.[0].tenantId')

cd packer
export ARM_CLIENT_ID=$appId
export ARM_CLIENT_SECRET=$pass
export ARM_RESOURCE_GROUP=$resourcegroup
export ARM_STORAGE_ACCOUNT=$storageaccount
export ARM_SUBSCRIPTION_ID=$subscriptionId
export ARM_TENANT_ID=$tenantId
packer build ubuntu.json

