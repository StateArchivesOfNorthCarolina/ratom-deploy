# Azure Instructions

This document provides instructions for provisioning a RATOM environment in
Azure.

## Provision

### Install the Azure CLI

First install the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest).

On a Mac, you can run:

```
brew update && brew install azure-cli
```

To sign in, use ``az login`` command.

If the CLI can open your default browser, it will do so and load an Azure sign-in page.


### (Optional) Set default subscription

You can configure the default subscription, used by the following ``az create`` requests, using:

```
az account set --subscription NAME_OR_ID
```


### Create Azure Kubernetes Service (AKS) Cluster

Create a RATOM resource group using [az group create](https://docs.microsoft.com/en-us/cli/azure/group?view=azure-cli-latest#az-group-create):

```sh
az group create \
    --name ratom-group \
    --location eastus
```

Create a new managed Azure Kubernetes Service (AKS) cluster using [az aks create](https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest#az-aks-create):

```
az aks create \
    --resource-group ratom-group \
    --name caktus-ratom \
    --location eastus \
    --node-count 2 \
    --node-vm-size Standard_D2s_v3 \
    --enable-addons monitoring \
    --kubernetes-version 1.15.7
```

## Azure Database for PostgreSQL

Create an Azure Database for PostgreSQL server using [az postgres server create](https://docs.microsoft.com/en-us/cli/azure/postgres/server?view=azure-cli-latest#az-postgres-server-create):

```
az postgres server create \
    --name ratomdb \
    --admin-user ratom \
    --admin-password <PASSWORD> \
    --sku-name B_Gen5_1 \
    --resource-group ratom-group \
    --version 11 \
    --location eastus
```

### Add Firewall Rules

```
az postgres server firewall-rule create -g testgroup -s testsvr -n allowip --start-ip-address 107.46.14.221 --end-ip-address 107.46.14.221
```


## Configure Cluster

To configure ``kubectl`` to connect to your Kubernetes cluster, use ``az aks get-credentials``:

```
az aks get-credentials --resource-group ratom-group --name caktus-ratom
```

Verify the connecting to your cluster:

```
kubectl get nodes
```
