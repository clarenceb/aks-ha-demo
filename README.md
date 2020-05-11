AKS HA Demo
===========

**Note this repo is WIP and has lots of duplicate and hardcoded steps.
Use it for reference purposes only!**

Create Region 1 resources
-------------------------

```sh
source aks-ha-demo.sh        

az group create --name $RESOURCE_GROUP1 --location $REGION1

# VNET and App GW subnet
az network vnet create \
  --name $VNET1_NAME \
  --resource-group $RESOURCE_GROUP1 \
  --location $REGION1 \
  --address-prefix $VNET1_ADDRESS_PREFIX \
  --subnet-name $AG1_NAME \
  --subnet-prefix $AG1_SUBNET_PREFIX

# AKS-blue subnet
az network vnet subnet create \
--name "${VNET1_NAME}-aks-blue" \
--resource-group $RESOURCE_GROUP1 \
--vnet-name $VNET1_NAME   \
--address-prefix ${BLUE1_SUBNET_PREFIX}

vnet1_blue_subnet_id="$(az network vnet subnet show -g $RESOURCE_GROUP1 --vnet-name $VNET1_NAME --name "${VNET1_NAME}-aks-blue" --query id -o tsv)"

# AKS-green subnet
az network vnet subnet create \
--name "${VNET1_NAME}-aks-green" \
--resource-group $RESOURCE_GROUP1 \
--vnet-name $VNET1_NAME   \
--address-prefix $GREEN1_SUBNET_PREFIX

vnet1_green_subnet_id="$(az network vnet subnet show -g $RESOURCE_GROUP1 --vnet-name $VNET1_NAME --name "${VNET1_NAME}-aks-green" --query id -o tsv)"
```

Create AKS clusters - Region 1
------------------------------

```sh
# AKS-blue create cluster
az aks create \
    --resource-group $RESOURCE_GROUP1 \
    --name "aks-blue-$REGION1_SHORTNAME" \
    --network-plugin azure \
    --vnet-subnet-id $vnet1_blue_subnet_id \
    --docker-bridge-address 172.17.0.1/16 \
    --dns-service-ip $BLUE1_DNS_SERVICE_IP \
    --service-cidr $BLUE1_SERVICE_CIDR \
    --generate-ssh-keys \
    -c 1 \
    -k 1.15.7 \
    --tags cluster=blue \
    --nodepool-labels cluster=blue

# AKS-green create cluster (optional - can create this when needed)
az aks create \
    --resource-group $RESOURCE_GROUP1 \
    --name "aks-green-$REGION1_SHORTNAME" \
    --network-plugin azure \
    --vnet-subnet-id $vnet1_green_subnet_id \
    --docker-bridge-address 172.17.0.1/16 \
    --dns-service-ip $GREEN1_DNS_SERVICE_IP \
    --service-cidr $GREEN1_SERVICE_CIDR \
    --generate-ssh-keys \
    -c 1 \
    -k 1.15.7 \
    --tags cluster=green \
    --nodepool-labels cluster=green

# Create Log Analytics workspace and add Container Insights to both clusters
# (you can create separate Log Analytics workspaces rather than sharing one)
az monitor log-analytics workspace create \
  --resource-group $RESOURCE_GROUP1 \
  --workspace "aks-logs-$REGION1_SHORTNAME"

workspace1_id=$(az monitor log-analytics workspace show --resource-group $RESOURCE_GROUP1 --workspace "aks-logs-$REGION1_SHORTNAME" --query id -o tsv)

az aks enable-addons --resource-group $RESOURCE_GROUP1 --name "aks-blue-$REGION1_SHORTNAME" --addons monitoring --workspace-resource-id $workspace1_id
az aks enable-addons --resource-group $RESOURCE_GROUP1 --name "aks-green-$REGION1_SHORTNAME" --addons monitoring --workspace-resource-id $workspace1_id
```

Create Application Gateway - Region 1
-------------------------------------

```sh
az network public-ip create \
  --resource-group $RESOURCE_GROUP1 \
  --name "appgw-pip-$REGION1_SHORTNAME" \
  --allocation-method Static \
  --sku Standard

az network application-gateway create \
  --name "appgw-$REGION1_SHORTNAME" \
  --location $REGION1 \
  --resource-group $RESOURCE_GROUP1 \
  --capacity 2 \
  --sku Standard_v2 \
  --public-ip-address "appgw-pip-$REGION1_SHORTNAME" \
  --vnet-name $VNET1_NAME \
  --subnet $AG1_NAME
```

Setup Application Gateway Ingress Controllers - Region 1
--------------------------------------------------------

```sh
# Here we are sharing the same SP for both blue and green clusters - you could create separate SPs instead.
az ad sp create-for-rbac --name "http://appgw-$REGION1_SHORTNAME-sp" --sdk-auth | base64 -w0 > secretJSON-$REGION1_SHORTNAME.txt
armAuthSecret=$(cat secretJSON-$REGION1_SHORTNAME.txt)
subscriptionId=$(az account show --query id -o tsv)
```

### Blue cluster setup

```sh
az aks get-credentials -n "aks-blue-$REGION1_SHORTNAME" -g $RESOURCE_GROUP1
blue_aks_fqdn=$(az aks show -g $RESOURCE_GROUP1 -n "aks-blue-$REGION1_SHORTNAME" --query fqdn -o tsv)
kubectl apply -f https://raw.githubusercontent.com/Azure/application-gateway-kubernetes-ingress/master/crds/AzureIngressProhibitedTarget.yaml

helm repo add application-gateway-kubernetes-ingress https://appgwingress.blob.core.windows.net/ingress-azure-helm-package/
helm repo update
helm upgrade --install ingress-azure application-gateway-kubernetes-ingress/ingress-azure \
     --namespace default \
     --debug \
     --set appgw.name="appgw-$REGION1_SHORTNAME" \
     --set appgw.resourceGroup=$RESOURCE_GROUP1 \
     --set appgw.subscriptionId=$subscriptionId \
     --set appgw.usePrivateIP=false \
     --set appgw.shared=true \
     --set armAuth.type=servicePrincipal \
     --set armAuth.secretJSON=$armAuthSecret \
     --set rbac.enabled=true \
     --set verbosityLevel=3 \
     --set kubernetes.watchNamespace="" \
     --set aksClusterConfiguration.apiServerAddress=$blue_aks_fqdn

# TODO: Set up DNS zone (mydomain.com)
# TODO: Set up public ip alias record sets for App GW frontend ip (blue-eau.mydomain.com)
# TODO: Add TLS with Let's Encrypt

kubectl get AzureIngressProhibitedTargets -A
kubectl delete AzureIngressProhibitedTarget prohibit-all-targets

# Deploy a sample app to blue cluster
kubectl create ns guestbook-blue
kubectl apply -f prohibit-target-green-eau.yaml -n guestbook-blue
kubectl apply -f guestbook-all-in-one.yaml -n guestbook-blue
kubectl apply -f guestbook-ingress-blue-eau.yaml -n guestbook-blue
```

### Green cluster setup

```sh
az aks get-credentials -n "aks-green-$REGION1_SHORTNAME" -g $RESOURCE_GROUP1
green_aks_fqdn=$(az aks show -g $RESOURCE_GROUP1 -n "aks-green-$REGION1_SHORTNAME" --query fqdn -o tsv)
kubectl apply -f https://raw.githubusercontent.com/Azure/application-gateway-kubernetes-ingress/master/crds/AzureIngressProhibitedTarget.yaml

helm repo add application-gateway-kubernetes-ingress https://appgwingress.blob.core.windows.net/ingress-azure-helm-package/
helm repo update
helm upgrade --install ingress-azure application-gateway-kubernetes-ingress/ingress-azure \
     --namespace default \
     --debug \
     --set appgw.name="appgw-$REGION1_SHORTNAME" \
     --set appgw.resourceGroup=$RESOURCE_GROUP1 \
     --set appgw.subscriptionId=$subscriptionId \
     --set appgw.usePrivateIP=false \
     --set appgw.shared=true \
     --set armAuth.type=servicePrincipal \
     --set armAuth.secretJSON=$armAuthSecret \
     --set rbac.enabled=true \
     --set verbosityLevel=3 \
     --set kubernetes.watchNamespace="" \
     --set aksClusterConfiguration.apiServerAddress=$green_aks_fqdn

# TODO: Set up DNS zone (mydomain.com)
# TODO: Set up public ip alias record sets for App GW frontend ip (green-eau.mydomain.com)
# TODO: Add TLS with Let's Encrypt

kubectl get AzureIngressProhibitedTargets -A
kubectl delete AzureIngressProhibitedTarget prohibit-all-targets

# Deploy a sample app to green cluster
kubectl create ns guestbook-green
kubectl apply -f prohibit-target-blue-eau.yaml -n guestbook-green
kubectl apply -f guestbook-all-in-one.yaml -n guestbook-green
kubectl apply -f guestbook-ingress-green-eau.yaml -n guestbook-green
```

Create Region 2 resources
-------------------------

```sh
az group create --name $RESOURCE_GROUP2 --location $REGION2

# VNET and App GW subnet
az network vnet create \
  --name $VNET2_NAME \
  --resource-group $RESOURCE_GROUP2 \
  --location $REGION2 \
  --address-prefix $VNET2_ADDRESS_PREFIX \
  --subnet-name $AG2_NAME \
  --subnet-prefix $AG2_SUBNET_PREFIX

# AKS-blue subnet
az network vnet subnet create \
--name "${VNET2_NAME}-aks-blue" \
--resource-group $RESOURCE_GROUP2 \
--vnet-name $VNET2_NAME   \
--address-prefix ${BLUE2_SUBNET_PREFIX}

vnet2_blue_subnet_id="$(az network vnet subnet show -g $RESOURCE_GROUP2 --vnet-name $VNET2_NAME --name "${VNET2_NAME}-aks-blue" --query id -o tsv)"

# AKS-green subnet
az network vnet subnet create \
--name "${VNET2_NAME}-aks-green" \
--resource-group $RESOURCE_GROUP2 \
--vnet-name $VNET2_NAME   \
--address-prefix $GREEN2_SUBNET_PREFIX

vnet2_green_subnet_id="$(az network vnet subnet show -g $RESOURCE_GROUP2 --vnet-name $VNET2_NAME --name "${VNET2_NAME}-aks-green" --query id -o tsv)"
```

Create AKS clusters - Region 2
------------------------------

```sh
# AKS-blue create cluster
az aks create \
    --resource-group $RESOURCE_GROUP2 \
    --name "aks-blue-$REGION2_SHORTNAME" \
    --network-plugin azure \
    --vnet-subnet-id $vnet2_blue_subnet_id \
    --docker-bridge-address 172.17.0.1/16 \
    --dns-service-ip $BLUE2_DNS_SERVICE_IP \
    --service-cidr $BLUE2_SERVICE_CIDR \
    --generate-ssh-keys \
    -c 1 \
    -k 1.15.7 \
    --tags cluster=blue \
    --nodepool-labels cluster=blue

# AKS-green create cluster (optional - can create this when needed)
az aks create \
    --resource-group $RESOURCE_GROUP2 \
    --name "aks-green-$REGION2_SHORTNAME" \
    --network-plugin azure \
    --vnet-subnet-id $vnet2_green_subnet_id \
    --docker-bridge-address 172.17.0.1/16 \
    --dns-service-ip $GREEN2_DNS_SERVICE_IP \
    --service-cidr $GREEN2_SERVICE_CIDR \
    --generate-ssh-keys \
    -c 1 \
    -k 1.15.7 \
    --tags cluster=green \
    --nodepool-labels cluster=green

# Create Log Analytics workspace and add Container Insights to both clusters
# (you can create separate Log Analytics workspaces rather than sharing one)
az monitor log-analytics workspace create \
  --resource-group $RESOURCE_GROUP2 \
  --workspace "aks-logs-$REGION2_SHORTNAME"

workspace2_id=$(az monitor log-analytics workspace show --resource-group $RESOURCE_GROUP2 --workspace "aks-logs-$REGION2_SHORTNAME" --query id -o tsv)

az aks enable-addons --resource-group $RESOURCE_GROUP2 --name "aks-blue-$REGION2_SHORTNAME" --addons monitoring --workspace-resource-id $workspace2_id
az aks enable-addons --resource-group $RESOURCE_GROUP2 --name "aks-green-$REGION2_SHORTNAME" --addons monitoring --workspace-resource-id $workspace2_id
```

Create Application Gateway - Region 2
-------------------------------------

```sh
az network public-ip create \
  --resource-group $RESOURCE_GROUP2 \
  --name "appgw-pip-$REGION2_SHORTNAME" \
  --allocation-method Static \
  --sku Standard

az network application-gateway create \
  --name "appgw-$REGION2_SHORTNAME" \
  --location $REGION2 \
  --resource-group $RESOURCE_GROUP2 \
  --capacity 2 \
  --sku Standard_v2 \
  --public-ip-address "appgw-pip-$REGION2_SHORTNAME" \
  --vnet-name $VNET2_NAME \
  --subnet $AG2_NAME
```

Setup Application Gateway Ingress Controllers - Region 2
--------------------------------------------------------

```sh
# Here we are sharing the same SP for both blue and green clusters - you could create separate SPs instead.
az ad sp create-for-rbac --name "http://appgw-$REGION2_SHORTNAME-sp" --sdk-auth | base64 -w0 > secretJSON-$REGION2_SHORTNAME.txt
armAuthSecret=$(cat secretJSON-$REGION2_SHORTNAME.txt)
subscriptionId=$(az account show --query id -o tsv)
```

### Blue cluster setup

```sh
az aks get-credentials -n "aks-blue-$REGION2_SHORTNAME" -g $RESOURCE_GROUP2
blue_aks_fqdn=$(az aks show -g $RESOURCE_GROUP2 -n "aks-blue-$REGION2_SHORTNAME" --query fqdn -o tsv)
kubectl apply -f https://raw.githubusercontent.com/Azure/application-gateway-kubernetes-ingress/master/crds/AzureIngressProhibitedTarget.yaml

helm repo add application-gateway-kubernetes-ingress https://appgwingress.blob.core.windows.net/ingress-azure-helm-package/
helm repo update
helm upgrade --install ingress-azure application-gateway-kubernetes-ingress/ingress-azure \
     --namespace default \
     --debug \
     --set appgw.name="appgw-$REGION2_SHORTNAME" \
     --set appgw.resourceGroup=$RESOURCE_GROUP2 \
     --set appgw.subscriptionId=$subscriptionId \
     --set appgw.usePrivateIP=false \
     --set appgw.shared=true \
     --set armAuth.type=servicePrincipal \
     --set armAuth.secretJSON=$armAuthSecret \
     --set rbac.enabled=true \
     --set verbosityLevel=3 \
     --set kubernetes.watchNamespace="" \
     --set aksClusterConfiguration.apiServerAddress=$blue_aks_fqdn

# TODO: Set up DNS zone (mydomain.com)
# TODO: Set up public ip alias record sets for App GW frontend ip (blue-eau.mydomain.com)
# TODO: Add TLS with Let's Encrypt

kubectl get AzureIngressProhibitedTargets -A
kubectl delete AzureIngressProhibitedTarget prohibit-all-targets

# Deploy a sample app to blue cluster
kubectl create ns guestbook-blue
kubectl apply -f prohibit-target-green-ase.yaml -n guestbook-blue
kubectl apply -f guestbook-all-in-one.yaml -n guestbook-blue
kubectl apply -f guestbook-ingress-blue-ase.yaml -n guestbook-blue
```

### Green cluster setup

```sh
az aks get-credentials -n "aks-green-$REGION2_SHORTNAME" -g $RESOURCE_GROUP2
green_aks_fqdn=$(az aks show -g $RESOURCE_GROUP2 -n "aks-green-$REGION2_SHORTNAME" --query fqdn -o tsv)
kubectl apply -f https://raw.githubusercontent.com/Azure/application-gateway-kubernetes-ingress/master/crds/AzureIngressProhibitedTarget.yaml

helm repo add application-gateway-kubernetes-ingress https://appgwingress.blob.core.windows.net/ingress-azure-helm-package/
helm repo update
helm upgrade --install ingress-azure application-gateway-kubernetes-ingress/ingress-azure \
     --namespace default \
     --debug \
     --set appgw.name="appgw-$REGION2_SHORTNAME" \
     --set appgw.resourceGroup=$RESOURCE_GROUP2 \
     --set appgw.subscriptionId=$subscriptionId \
     --set appgw.usePrivateIP=false \
     --set appgw.shared=true \
     --set armAuth.type=servicePrincipal \
     --set armAuth.secretJSON=$armAuthSecret \
     --set rbac.enabled=true \
     --set verbosityLevel=3 \
     --set kubernetes.watchNamespace="" \
     --set aksClusterConfiguration.apiServerAddress=$green_aks_fqdn

# TODO: Set up DNS zone (mydomain.com)
# TODO: Set up public ip alias record sets for App GW frontend ip (green-eau.mydomain.com)
# TODO: Add TLS with Let's Encrypt

kubectl get AzureIngressProhibitedTargets -A
kubectl delete AzureIngressProhibitedTarget prohibit-all-targets

# Deploy a sample app to green cluster
kubectl create ns guestbook-green
kubectl apply -f prohibit-target-blue-ase.yaml -n guestbook-green
kubectl apply -f guestbook-all-in-one.yaml -n guestbook-green
kubectl apply -f guestbook-ingress-green-ase.yaml -n guestbook-green
```

## TODO AFD:
- Create AFD and backend pools
- Deploy workloads with health and readiness probes (show blue/green labels in page response)
- Cut-over blue/green via enabling/diabling backends in region pool
- Failover to other region by manipulating ingress rule to create an non-200 response
- Use k6 to generate traffic and log responses (distribution of blue/green and region, and success/failure)
- Restrict App GW to AFD
- Introduce caching (only do this is there are cache busting URIs for static assets)
