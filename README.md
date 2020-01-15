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

## Bootstrap host and install microk8s
ansible-playbook -u ubuntu playbooks/deploy-host.yml -vv
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
ansible-playbook playbooks/configure-cluster.yml -vv
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


# Staging environment

## Cluster Access

* Download and install
  [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) (please
  ensure you install v1.16 or later; v1.17 is current as of January, 2020)

* Populate credentials in your ``~/.kube/config`` by asking another developer.

* Verify access to the cluster:

```sh
    $ kubectl cluster-info
    Kubernetes master is running at https://...
    CoreDNS is running at https://...
    Metrics-server is running at https://...
```

   To further debug and diagnose cluster problems, use ``kubectl cluster-info dump``.

* Set the namespace in your ``kubectl`` context to ``ratom-staging``:

```sh
    kubectl config set-context --current --namespace=ratom-staging
```

## Interacting with Pods

You can interact with running pods via ``kubectl``, for example:

```sh
    # list running pods
    $ kubectl get pods
    NAME                       READY   STATUS    RESTARTS   AGE
    api-55c4fbb789-b8m2v       1/1     Running   0          13m
    api-55c4fbb789-zmksn       1/1     Running   0          13m
    frontend-687d4b9bf-9xcfz   1/1     Running   0          15m
    frontend-687d4b9bf-pnqkw   1/1     Running   0          15m

    # tail logs for the api
    $ kubectl logs -f deployment/api
    # <snip>
    [pid: 15|app: 0|req: 10/14] 10.52.1.7 () {58 vars in 1375 bytes} [Fri Nov  8 11:19:57 2019] GET /admin/ratom/message/ => generated 28852 bytes in 129 msecs (HTTP/1.1 200) 10 headers in 513 bytes (1 switches on core 2)
    [pid: 14|app: 0|req: 5/15] 10.52.1.7 () {60 vars in 1271 bytes} [Fri Nov  8 11:20:32 2019] POST /graphql => generated 240 bytes in 30 msecs (HTTP/1.1 200) 8 headers in 400 bytes (1 switches on core 1)

    # start a shell in a pod, where you can run management commands, etc.
    $ kubectl exec -it api-55c4fbb789-b8m2v bash
    root@api-55c4fbb789-b8m2v:/code#

    # copy a file to a pod
    $ kubectl cp /path/to/source api-55c4fbb789-b8m2v:/path/to/dest
```
