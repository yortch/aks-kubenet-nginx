# AKS cluster with Kubenet networking, NGINX ingress controller and Application Gateway
Instructions for setting up AKS cluster with the following configuration:
* Kubenet networking
* NGINX ingress controller
* Azure Application Gateway

## Prerequisites
* [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/)
* [Helm](https://helm.sh/docs/intro/install/)
* [kubectl](https://kubernetes.io/docs/tasks/tools/)

## Create AKS cluster

### Create a virtual network and subnet

1. Create a resource group using the `az group create` command.

    ```azurecli-interactive
    RG=aks-kubenet-nginx
    REGION=eastus
    az login
    az group create --name $RG --location $REGION
    ```

1. If you don't have an existing virtual network and subnet to use, create these network resources using the `az network vnet create` command. The following example command creates a virtual network named *aks-vnet* with the address prefix of *10.21.0.0/16* and a subnet named *aks-subnet* with the address prefix *10.21.0.0/24*:

    ```azurecli-interactive
    VNET=aks-vnet
    SUBNET=aks-subnet
    az network vnet create \
        --resource-group $RG \
        --name $VNET \
        --subnet-name $SUBNET \
        --address-prefix 10.21.0.0/16 \
        --subnet-prefix 10.21.0.0/24 \
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
    --generate-ssh-keys 
```

## Create an unmanaged ingress controller
Instructions for installing and configuring NGINX Ingress controller are based on: https://github.com/MicrosoftDocs/azure-docs/blob/main/articles/aks/ingress-basic.md

Get the connect command to AKS cluster from the Azure Portal:

```
az aks get-credentials --resource-group $RG --name $CLUSTER --overwrite-existing
```

Get the first IP address available in the subnet to use as the load balancer IP:
```
AKS_LB_IP_ADDRESS=$(az network vnet subnet list-available-ips -g $RG --vnet-name $VNET -n $SUBNET --query
"[0]")
```

Add the ingress-nginx repository and use Helm to deploy an NGINX ingress controller. 

```console
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
    --version 4.7.1 \
    --namespace nginx-ingress \
    --create-namespace \
    --set controller.replicaCount=2 \
    --set controller.service.loadBalancerIP=$AKS_LB_IP_ADDRESS \
    --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-internal"=true \
    --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz 
```

If the command is successful, it should output the command below which can be used to validate an External IP which will be in pending state:

```
kubectl --namespace ingress-basic get services -o wide -w ingress-nginx-controller
```

## Deploy demo applications

To see the ingress controller in action, run two demo applications in your AKS cluster. In this example, you use `kubectl apply` to deploy two instances of a simple *Hello world* application.

Run the two demo applications using `kubectl apply`:

```console
kubectl apply -f aks-helloworld-one.yaml --namespace ingress-basic
kubectl apply -f aks-helloworld-two.yaml --namespace ingress-basic
```

### Create an ingress route

Both applications are now running on your Kubernetes cluster. To route traffic to each application, create a Kubernetes ingress resource. The ingress resource configures the rules that route traffic to one of the two applications.

In the following example, traffic to *EXTERNAL_IP/hello-world-one* is routed to the service named `aks-helloworld-one`. Traffic to *EXTERNAL_IP/hello-world-two* is routed to the `aks-helloworld-two` service. Traffic to *EXTERNAL_IP/static* is routed to the service named `aks-helloworld-one` for static assets.

Create the ingress resource using the `kubectl apply` command.

```
kubectl apply -f hello-world-ingress.yaml --namespace ingress-basic
```

## Create Application Gateway

1. Create Virtual Network and Subnet for Application Gateway using the `az network vnet create` command. 

    ```azurecli-interactive
    APPGW_VNET=appgw-vnet
    APPGW_SUBNET=appgw-subnet
    az network vnet create \
        --resource-group $RG \
        --name $APPGW_VNET \
        --subnet-name $APPGW_SUBNET \
        --address-prefix 10.20.0.0/16 \
        --subnet-prefix 10.20.0.0/24 \
        --location $REGION
    ```

1. Create a public IP and save the IP Address for the Application Gateway

    ```console
    APPGW_PIP=aks-appgw-pip
    az network public-ip create --resource-group $RG --name $APPGW_PIP --sku Standard --allocation-method static
    APPGW_PIP_ADDRESS=$(az network public-ip show --resource-group $RG --name $APPGW_PIP --query ipAddress -o tsv)
    ```

1. Set up bi-directional network peering between the AKS subnet and the Application Gateway virtual network:

    ```command
    # get aks vnet id 
    VNET_ID=$(az network vnet show -n $VNET -g $RG -o tsv --query "id")
    # get gateway vnet id
    APPGW_VNET_ID=$(az network vnet show --name $APPGW_VNET -g $RG -o tsv --query "id")
    az network vnet peering create -n appgw2aks \
        -g $RG --vnet-name $APPGW_VNET \
        --remote-vnet $VNET_ID \
        --allow-vnet-access
    az network vnet peering create -n aks2appgw \
        -g $RG --vnet-name $VNET \
        --remote-vnet $APPGW_VNET_ID \
        --allow-vnet-access
    ```

1. Create Application Gateway:
    ```
    APPGW='appgw-aks' 
    az network application-gateway create --name $APPGW --resource-group $RG --subnet $APPGW_SUBNET --vnet-name $APPGW_VNET --sku Standard_V2 --priority 1001 --servers $AKS_LB_IP_ADDRESS --public-ip-address $APPGW_PIP
    ```

1. Navigate to the browser or use the command below to verify connectivity
    ```
    curl -L http://$APPGW_PIP
    ```


# Reference
* https://github.com/MicrosoftDocs/azure-docs/blob/main/articles/aks/configure-kubenet.md
* https://azure.github.io/application-gateway-kubernetes-ingress/how-tos/networking/
* https://medium.com/@saifeddine.segni94/nginx-ingress-controllers-and-azure-app-gateway-for-azure-kubernetes-service-aks-f18c5be955d0