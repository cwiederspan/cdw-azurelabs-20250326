# Azure AKS Labs

A repo for working through the Azure Labs content - https://azure-samples.github.io/aks-labs/docs/intro

```bash

source .env

az login -t $TENANT_ID

USER_ID="$(az ad signed-in-user show --query id -o tsv)"

az extension list --query "[?name=='aks-preview']"

az extension add --name aks-preview

az account list-locations --query "[?metadata.regionType=='Physical' && metadata.supportsAvailabilityZones==true].{Region:name}" -o table

az group create --name ${RG_NAME} --location ${LOCATION}

az deployment group create \
--name ${DEPLOY_NAME} \
--resource-group $RG_NAME \
--template-uri https://raw.githubusercontent.com/azure-samples/aks-labs/refs/heads/main/docs/getting-started/assets/aks-labs-deploy.json \
--parameters userObjectId=${USER_ID} \
--no-wait

K8S_VERSION=$(az aks get-versions -l ${LOCATION} \
--query "reverse(sort_by(values[?isDefault==true].{version: version}, &version)) | [1] " \
-o tsv)

az aks create \
--resource-group ${RG_NAME} \
--name ${AKS_NAME} \
--location ${LOCATION} \
--tier standard \
--kubernetes-version ${K8S_VERSION} \
--os-sku AzureLinux \
--nodepool-name systempool \
--node-count 3 \
--zones 1 2 3 \
--load-balancer-sku standard \
--network-plugin azure \
--network-plugin-mode overlay \
--network-dataplane cilium \
--network-policy cilium \
--ssh-access disabled \
--enable-managed-identity \
--enable-acns \
--generate-ssh-keys

```
