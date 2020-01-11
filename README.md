# ratom-deploy

## AWS, Ansible, microk8s

Collection of Ansible playbooks and roles to provision and deploy [MicroK8s](https://microk8s.io/) on AWS for the RATOM project.


## Setup

Install Python and Ansible requirements:

```sh
pip install -r requirements.txt
ansible-galaxy install -f -r requirements.yml -p roles/
```

## Provision and Deploy

To deploy the **ratom-dev** Kubernetes cluster, found in ``envs/dev``, run:

```sh
export AWS_PROFILE=ratom

## Provision CloudFormation stack
ansible-playbook playbooks/provision.yml -vv

## Deploy microk8s
ansible-playbook -u ubuntu playbooks/deploy.yml -vv
```


## Managing Kubernetes Clusters


### Setup kubctl

To run ``kubctl`` locally:

- On the host, run ``microk8s.config``
- Copy the YAML to ``~/.kube/config``
- Modify ``clusters.0.cluster.server`` to match the host's IP

Now you should be able to run:

```sh
kubectl cluster-info
```

Optionally, set your default namespace for this context::

```sh
kubectl config set-context --current --namespace=ratom-staging
```


### Install or update cluster

To install cert-manager for Let's Encrypt support, run:

```sh
ansible-playbook playbooks/kube-cluster.yml -vv
```


### Testing that Let's Encrypt is working

1. Find the hostname or IP of your load balancer.
2. Go to CloudFlare and update the DNS record for ``echotest.caktus-built.com`` to
   point to this hostname or IP address (switching the record type if needed).
3. Give the record a minute or two to propagate.
4. Apply the echotest file to the cluster::

       kubectl apply -f roles/kube-cluster/files/echotest.yml

5. Give the certificate a couple minutes to be generated and validated. While waiting,
   you can watch the output of:

       kubectl -n echoserver get pod

   When the ``cm-acme-http-solver`` pod goes away, the certificate should be validated.
   Now, navigate to https://echotest.caktus-built.com and ensure that you have a valid
   certificate. If you don't, you want to follow the [cert-manager troubleshooting](https://docs.cert-manager.io/en/latest/getting-started/troubleshooting.html)
   steps in the documentation. But, be sure to reload a few times, and close the
   browser tab and open a new one to make sure it's really broken, because sometimes
   it takes a few minutes to go through and the browser gets stuck with the
   temporary certificate.

6. You should see the ``*-tls`` secret in the **echoserver** namespace:

       kubectl -n echoserver get secret
       NAME                  TYPE                                  DATA   AGE
       default-token-62pdt   kubernetes.io/service-account-token   3      5m
       echoserver-tls        kubernetes.io/tls                     3      5m

   If not, you may need to re-create the ingress by deleteing and re-applying it.

7. When you're done, delete the echotest resources from the cluster:

       kubectl delete -f roles/kube-cluster/files/echotest.yml


## Deprovision

To destroy the CF stack, run:

```sh
## Deprovision stack
ansible-playbook playbooks/deprovision.yml -vv
```


## Update Python requirements

If ``requirements.in`` is updated, run:

```
pip-compile --upgrade requirements.in
pip-sync requirements.txt
```
