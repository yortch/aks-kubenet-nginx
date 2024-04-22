# AKS cluster with Kubenet networking, NGINX ingress controller and Application Gateway
Instructions for setting up AKS cluster with the following configuration:
* Kubenet networking
* NGINX ingress controller
* Azure Application Gateway

## Prerequisites
* [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/)
* [Helm](https://helm.sh/docs/intro/install/)

## Create AKS cluster
Instructions based on: https://github.com/MicrosoftDocs/azure-docs/blob/main/articles/aks/configure-kubenet.md

### Create a virtual network and subnet

1. Create a resource group using the `az group create` command.

    ```azurecli-interactive
    RG=aks-kubenet-nginx
    REGION=eastus
    az login
    az group create --name $RG --location $REGION
    ```

1. If you don't have an existing virtual network and subnet to use, create these network resources using the `az network vnet create` command. The following example command creates a virtual network named *aks-vnet* with the address prefix of *192.168.0.0/16* and a subnet named *myAKSSubnet* with the address prefix *192.168.1.0/24*:

    ```azurecli-interactive
    VNET=aks-vnet
    SUBNET=aks-subnet
    az network vnet create \
        --resource-group $RG \
        --name $VNET \
        --address-prefixes 192.168.0.0/16 \
        --subnet-name $SUBNET \
        --subnet-prefix 192.168.1.0/24 \
        --location $REGION
    ```

1. Get the subnet resource ID using the `az network vnet subnet show` command and store it as a variable named `SUBNET_ID` for later use.

    ```azurecli-interactive
    SUBNET_ID=$(az network vnet subnet show --resource-group $RG --vnet-name $VNET --name $SUBNET --query id -o tsv)
    ```

### Create an AKS cluster in the virtual network
First create Azure Container Registry so that it can be attach to the AKS cluster

```
REGISTRY_NAME=aksnginxacr
az acr create --name $REGISTRY_NAME --resource-group $RG --sku basic --location eastus
```

Create an AKS cluster using the `az aks create` command.

    ```azurecli-interactive
    CLUSTER=aks-cluster
    az aks create \
        --resource-group $RG \
        --name $CLUSTER \
        --network-plugin kubenet \
        --vnet-subnet-id $SUBNET_ID \
        --generate-ssh-keys \
        --attach-acr $REGISTRY_NAME
    ```

## Create an unmanaged ingress controller
Instructions for installing and configuring NGINX Ingress controller are based on: https://github.com/MicrosoftDocs/azure-docs/blob/main/articles/aks/ingress-basic.md

Get the connect command to AKS cluster from the Azure Portal:

``
az aks get-credentials --resource-group $RG --name $CLUSTER --overwrite-existing
``

### Import the images used by the Helm chart into your ACR

To control image versions, you'll want to import them into your own Azure Container Registry. The NGINX ingress controller Helm chart relies on three container images. Use `az acr import` to import those images into your ACR.

```azurecli
SOURCE_REGISTRY=registry.k8s.io
CONTROLLER_IMAGE=ingress-nginx/controller
CONTROLLER_TAG=v1.8.1
PATCH_IMAGE=ingress-nginx/kube-webhook-certgen
PATCH_TAG=v20230407
DEFAULTBACKEND_IMAGE=defaultbackend-amd64
DEFAULTBACKEND_TAG=1.5

az acr import --name $REGISTRY_NAME --resource-group $RG --source $SOURCE_REGISTRY/$CONTROLLER_IMAGE:$CONTROLLER_TAG --image $CONTROLLER_IMAGE:$CONTROLLER_TAG
az acr import --name $REGISTRY_NAME --resource-group $RG --source $SOURCE_REGISTRY/$PATCH_IMAGE:$PATCH_TAG --image $PATCH_IMAGE:$PATCH_TAG
az acr import --name $REGISTRY_NAME --resource-group $RG --source $SOURCE_REGISTRY/$DEFAULTBACKEND_IMAGE:$DEFAULTBACKEND_TAG --image $DEFAULTBACKEND_IMAGE:$DEFAULTBACKEND_TAG
```

Get ACR Login Server url:
```
 ACR_LOGIN_SERVER=$(az acr list -g $RG --query "[].{acrLoginServer:loginServer}" --output tsv)
```

Add the ingress-nginx repository and use Helm to deploy an NGINX ingress controller. For load balancer IP select an unused IP within the AKS Service CIDR (to be confirmed)

```console
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
 
helm install ingress-nginx ingress-nginx/ingress-nginx \
    --version 4.7.1 \
    --namespace ingress-basic \
    --create-namespace \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.image.registry=$ACR_LOGIN_SERVER \
    --set controller.image.image=$CONTROLLER_IMAGE \
    --set controller.image.tag=$CONTROLLER_TAG \
    --set controller.image.digest="" \
    --set controller.admissionWebhooks.patch.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.service.loadBalancerIP=10.0.222.22 \
    --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-internal"=true \
    --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz \
    --set controller.admissionWebhooks.patch.image.registry=$ACR_LOGIN_SERVER \
    --set controller.admissionWebhooks.patch.image.image=$PATCH_IMAGE \
    --set controller.admissionWebhooks.patch.image.tag=$PATCH_TAG \
    --set controller.admissionWebhooks.patch.image.digest="" \
    --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux \
    --set defaultBackend.image.registry=$ACR_LOGIN_SERVER \
    --set defaultBackend.image.image=$DEFAULTBACKEND_IMAGE \
    --set defaultBackend.image.tag=$DEFAULTBACKEND_TAG \
    --set defaultBackend.image.digest="" 
```

If the command is successful, it should output the command below which can be used to validate an External IP which will be in pending state:
```
kubectl --namespace ingress-basic get services -o wide -w ingress-nginx-controller
```
