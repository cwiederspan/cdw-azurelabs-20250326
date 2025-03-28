# Azure AKS Labs

A repo for working through the Azure Labs content - https://azure-samples.github.io/aks-labs/docs/intro

> NOTE: Use East US 2 as it currently supports 3 availability zones

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

# One-time Fix: I needed to set this before using the --ssh-access flag
az feature register --namespace "Microsoft.ContainerService" --name "DisableSSHPreview"
az feature show --namespace "Microsoft.ContainerService" --name "DisableSSHPreview"

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

az aks get-credentials \
--resource-group ${RG_NAME} \
--name ${AKS_NAME} \
--overwrite-existing

az aks nodepool add \
--resource-group ${RG_NAME} \
--cluster-name ${AKS_NAME} \
--mode User \
--name userpool \
--node-count 1 \
--node-vm-size Standard_DS2_v2 \
--zones 1 2 3 \
--ssh-access disabled       # Added to remain secure

# Taint the system node pool so user workloads aren't scheduled here
az aks nodepool update \
--resource-group ${RG_NAME} \
--cluster-name ${AKS_NAME} \
--name systempool \
--node-taints CriticalAddonsOnly=true:NoSchedule

# Export the env values from the deployment results (from above)
while IFS= read -r line; \
do echo "exporting $line"; \
export $line=$(az deployment group show -g ${RG_NAME} -n ${DEPLOY_NAME} --query "properties.outputs.${line}.value" -o tsv); \
done < <(az deployment group show -g $RG_NAME -n ${DEPLOY_NAME} --query "keys(properties.outputs)" -o tsv)

# Wire up the observability for the cluster
az aks update \
--resource-group ${RG_NAME} \
--name ${AKS_NAME} \
--enable-azure-monitor-metrics \
--azure-monitor-workspace-resource-id ${metricsWorkspaceId} \       # Fixed to match proper variable name
--grafana-resource-id ${grafanaDashboardId} \                       # Fixed to match proper variable name
--no-wait

az aks enable-addons \
--resource-group ${RG_NAME} \
--name ${AKS_NAME} \
--addon monitoring \
--workspace-resource-id ${logWorkspaceId} \                         # Fixed to match proper variable name
--no-wait

# Deploy the application
kubectl create namespace pets

kubectl apply -f https://raw.githubusercontent.com/Azure-Samples/aks-store-demo/refs/heads/main/aks-store-quickstart.yaml -n pets

kubectl get all -n pets

kubectl get svc store-front -n pets

# Move on to the Istio lab
az aks mesh enable \
  --resource-group ${RG_NAME} \
  --name ${AKS_NAME}

kubectl get pods -n aks-istio-system

# Install Pets like above if necessary

az aks show --resource-group ${RG_NAME} --name ${AKS_NAME} --query "serviceMeshProfile.istio.revisions"

kubectl get pods -n pets

kubectl rollout restart deployment order-service -n pets
kubectl rollout restart deployment product-service -n pets
kubectl rollout restart deployment store-front -n pets

kubectl rollout restart statefulset rabbitmq -n pets

kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: curl-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: curl
  template:
    metadata:
      labels:
        app: curl
    spec:
      containers:
      - name: curl
        image: docker.io/curlimages/curl
        command: ["sleep", "3600"]
EOF

# Test without mTLS

CURL_POD_NAME="$(kubectl get pod -l app=curl -o jsonpath="{.items[0].metadata.name}")"

kubectl exec -it ${CURL_POD_NAME} -- curl -IL store-front.pets.svc.cluster.local:80

# Now apply mTLS requirement

kubectl apply -n pets -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: pets-mtls
  namespace: pets
spec:
  mtls:
    mode: STRICT
EOF

kubectl exec -it ${CURL_POD_NAME} -- curl -IL store-front.pets.svc.cluster.local:80



kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: curl-inside
  namespace: pets
spec:
  replicas: 1
  selector:
    matchLabels:
      app: curl
  template:
    metadata:
      labels:
        app: curl
    spec:
      containers:
      - name: curl
        image: curlimages/curl
        command: ["sleep", "3600"]
EOF

CURL_INSIDE_POD="$(kubectl get pod -n pets -l app=curl -o jsonpath="{.items[0].metadata.name}")"

kubectl exec -it ${CURL_INSIDE_POD} -n pets -- curl -IL store-front.pets.svc.cluster.local:80

# Setup Ingress

az aks mesh enable-ingress-gateway  \
  --resource-group $RG_NAME \
  --name $AKS_NAME \
  --ingress-gateway-type external

```
