# Azure Instructions

This document provides instructions for provisioning a RATOM environment in
Azure.


## Provision Azure Resources

This section walks through creating the following Azure services using the CLI:
* [Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/)
* [Azure Database for PostgreSQL](https://docs.microsoft.com/en-us/azure/postgresql/)


### Install the Azure CLI

First install the
[Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest).
On a Mac, you can run:

```
brew update && brew install azure-cli
```

To sign in, use:

```sh
az login
```

If the CLI can open your default browser, it will do so and load an Azure
sign-in page.


#### (Optional) Set default subscription

You can configure the default Azure subscription, used by the following
``az create`` requests, using:

```
az account set --subscription NAME_OR_ID
```


### Create Azure Kubernetes Service (AKS) Cluster

Create a RATOM resource group using
[az group create](https://docs.microsoft.com/en-us/cli/azure/group?view=azure-cli-latest#az-group-create):

```sh
export RESOURCE_GROUP=yourgroupname
az group create \
    --name $RESOURCE_GROUP \
    --location eastus
```

Create a new managed Azure Kubernetes Service (AKS) cluster using
[az aks create](https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest#az-aks-create):

```sh
export CLUSTER_NAME=yourclustername
az aks create \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --location eastus \
    --node-count 2 \
    --node-vm-size Standard_D2s_v3 \
    --enable-addons monitoring \
    --kubernetes-version 1.15.7
```


### Azure Database for PostgreSQL

Create an Azure Database for PostgreSQL server using
[az postgres server create](https://docs.microsoft.com/en-us/cli/azure/postgres/server?view=azure-cli-latest#az-postgres-server-create):

```sh
export AZ_PGSERVER=yourservername
export PGUSER=ratom
export PGPASSWORD=<password>
az postgres server create \
    --name $AZ_PGSERVER \
    --admin-user $PGUSER \
    --admin-password $PGPASSWORD \
    --resource-group $RESOURCE_GROUP \
    --sku-name B_Gen5_1 \
    --version 11 \
    --location eastus
```

While this is creating, let's go ahead and configure our cluster.


## Configure Your Kubernetes Cluster

Next, install
[kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/), which we'll
use to manage the Kubernetes cluster.

To configure ``kubectl`` to connect to your Kubernetes cluster, use
``az aks get-credentials``:

```
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME
```

This configures ``kubectl`` credentials to the cluster under the context defined
in ``$CLUSTER_NAME``. Verify the connection to your cluster with:

```
kubectl get nodes
```

You should see a list of nodes.

Next, we'll add the cluster ingress controller and certificate manager using
[caktus.k8s-web-cluster](https://github.com/caktus/ansible-role-k8s-web-cluster),
an Ansible role.

Add a new host vars file to ``deploy/host_vars/$CLUSTER_NAME.yaml``. Use the
existing host vars files as a reference. The most important variables are:

* ``k8s_context``: The name of your cluster context found in your
  ``.kube/config`` file. This will likely just be ``$CLUSTER_NAME``.
* ``k8s_letsencrypt_email``: The email address used for Let's Encrypt
  cert-related emails.

Next add $CLUSTER_NAME to ``deploy/inventory`` and add it to the ``[k8s]``
group. Use the existing clusters as a reference.

Install it with:

```sh
ansible-playbook -l $CLUSTER_NAME deploy.yaml -vv
```

### Test Let's Encrypt

During the installation, an Azure public IP address is created for the nginx
ingress controller. It will take a few minutes to be assigned, but eventually
you should see it with this command:

```
kubectl get service -n ingress-nginx
```

Add a DNS record for ``k8s_echotest_hostname`` to point to this IP address. Give
the record a minute or two to propagate.

Now install the echo test server:

```sh
ansible-playbook -l $CLUSTER_NAME echotest.yaml -vv
```

Give the certificate a couple minutes to be generated and validated. While
waiting, you can watch the output of:

       kubectl -n echoserver get pod

When the ``cm-acme-http-solver`` pod goes away, the certificate should be
validated. Now, navigate to ``k8s_echotest_hostname`` and ensure that you have a
valid certificate.

To uninstall echotest, run:

```sh
ansible-playbook -l $CLUSTER_NAME echotest.yaml --extra-vars "k8s_echotest_state=absent" -vv
```


### Add PostgreSQL Firewall Rules

We need to allow connections into the PostgreSQL database server.

Save the **outbound** IP address of the kubernetes cluster to an environment
variable. This is not the same IP address obtained from the ingress controller
above. Find it in the Azure console and add it to your shell's environment:

```sh
export IP_ADDRESS=<ip-address>
```

Add a firewall rule to grant it access:

```sh
az postgres server firewall-rule create \
    --resource-group $RESOURCE_GROUP \
    --server-name $AZ_PGSERVER \
    --name k8s-cluster \
    --start-ip-address $IP_ADDRESS \
    --end-ip-address $IP_ADDRESS
```

You'll likely also want to, at least temporarily, grant access to your local IP
to run a few SQL statements against it. Re-run the the above commands with this
IP as well.

```sh
export IP_ADDRESS=<your-external-ip-address>
```

Add a firewall rule to grant it access:

```sh
az postgres server firewall-rule create \
    --resource-group $RESOURCE_GROUP \
    --server-name $AZ_PGSERVER \
    --name my-ip-address \
    --start-ip-address $IP_ADDRESS \
    --end-ip-address $IP_ADDRESS
```

### Create Project PostgreSQL User and Database

Next we'll create a PostgreSQL user and database for the project. Obtain the FQDN with:

```sh
az postgres server show \
    --resource-group $RESOURCE_GROUP \
    --name $AZ_PGSERVER \
    | grep fullyQualifiedDomainName
```

Set a few more environment variables for ``psql`` to use. Azure requires the
username to be be in <username@hostname> format.

```sh
export PGSSLMODE=require
export PGUSER=$PGUSER@$AZ_PGSERVER  # Azure requirement
export PGHOST=<fqdn>
```

Now you should be able to connect directly to the default ``postgres`` database:

```sh
psql postgres
```

Create a database and user. Adjust these parameters to your needs:

```sql
CREATE DATABASE <dbname>;
CREATE ROLE <dbuser> WITH LOGIN NOSUPERUSER INHERIT CREATEDB NOCREATEROLE NOREPLICATION PASSWORD '<password>';
GRANT CONNECT ON DATABASE <dbname> TO <dbuser>;
GRANT ALL PRIVILEGES ON DATABASE <dbname> TO <dbuser>;
```

Extensions require admin permissions, so add them now:

```sql
\c <dbname>
CREATE EXTENSION IF NOT EXISTS citext;
```


## Azure Storage Account

Create an Azure storage account using
[az storage account create](https://docs.microsoft.com/en-us/cli/azure/storage/account?view=azure-cli-latest#az-storage-account-create):

```sh
export STORAGE_NAME=yourstoragename
az storage account create \
    --name $STORAGE_NAME \
    --resource-group $RESOURCE_GROUP
```

Next, create a container in the above storage account that we'll use to store PST files using:
[az storage container create](https://docs.microsoft.com/en-us/cli/azure/storage/container?view=azure-cli-latest#az-storage-container-create):

```sh
export CONTAINER_NAME=yourcontainername
az storage container create \
    --account-name $STORAGE_NAME \
    --name $CONTAINER_NAME \
    --public-access off
```


## Deploy RATOM application

Set current namespace:

```sh
kubectl config set-context --current --namespace=ratom-staging
```

From the ``ratom_web`` repo, run:

```sh
cd deployment/
ansible-playbook deploy.yaml -l $CLUSTER_NAME
```

Fro the ``ratom_server`` repo, run:

```sh
cd deployment/
ansible-playbook deploy.yaml -l $CLUSTER_NAME
```
