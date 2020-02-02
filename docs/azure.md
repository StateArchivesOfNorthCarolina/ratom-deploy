# Azure Instructions

This document provides instructions for provisioning a RATOM environment in
Azure.

## Provision

### Install the Azure CLI

First install the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest). On a Mac, you can run:

```
brew update && brew install azure-cli
```

To sign in, use the ``az login`` command. If the CLI can open your default
browser, it will do so and load an Azure sign-in page.


### (Optional) Set default subscription

You can configure the default Azure subscription, used by the following
``az create`` requests, using:

```
az account set --subscription NAME_OR_ID
```


### Create Azure Kubernetes Service (AKS) Cluster

Create a RATOM resource group using
[az group create](https://docs.microsoft.com/en-us/cli/azure/group?view=azure-cli-latest#az-group-create):

```sh
az group create \
    --name ratom-group \
    --location eastus
```

Create a new managed Azure Kubernetes Service (AKS) cluster using
[az aks create](https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest#az-aks-create):

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

Create an Azure Database for PostgreSQL server using
[az postgres server create](https://docs.microsoft.com/en-us/cli/azure/postgres/server?view=azure-cli-latest#az-postgres-server-create):

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

This configures ``kubectl`` credentials to the cluster under the context
``caktus-ratom``. Verify the connection to your cluster with:

```
kubectl get nodes
```

You should see a list of nodes.

Next, we'll add the cluster ingress controller and certificate manager using
[caktus.k8s-web-cluster](https://github.com/caktus/ansible-role-k8s-web-cluster),
an Ansible role. Review the variables in ``caktus.k8s-web-cluster.yml``. These need to be configured for your cluster.

Install it with:

```sh
ansible-playbook -i envs/caktus-aks playbooks/configure-cluster.yml -vv
```

## Test Let's Encrypt

During the installation, an Azure public IP address is created for the nginx ingress
controller. It will take a few minutes to be assigned, but eventually you should see it with this command:

```
kubectl get service -n ingress-nginx
```

Add a DNS record for ``k8s_echotest_hostname`` to point to this IP address. Give
the record a minute or two to propagate.

Now install the echo test server:

```sh
ansible-playbook -i envs/caktus-aks playbooks/echotest.yml -vv
```

Give the certificate a couple minutes to be generated and validated. While
waiting, you can watch the output of:

       kubectl -n echoserver get pod

When the ``cm-acme-http-solver`` pod goes away, the certificate should be
validated. Now, navigate to ``k8s_echotest_hostname`` and ensure that you have a
valid certificate.

To uninstall echotest, run:

```sh
ansible-playbook -i envs/caktus-aks playbooks/echotest.yml --extra-vars "k8s_echotest_state=absent" -vv
```
