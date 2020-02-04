# ratom-deploy

Configure a Kubernetes cluster for the RATOM project.


## Setup

Install Python and Ansible requirements:

```sh
pip install -r requirements.txt
ansible-galaxy install -f -r requirements.yml -p roles/
```

## Choose Hosting Provider

* [Microsoft Azure](docs/azure.md)
* [Amazon Web Services (AWS)](docs/aws.md) (not fully supported)


## Cluster Access

* Download and install
  [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) (please
  ensure you install v1.16 or later; v1.17 is current as of January, 2020)

* Obtain Kubernetes API credentials for the environment.

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
    $ kubectl logs -f -lapp=api
    # <snip>
    [pid: 15|app: 0|req: 10/14] 10.52.1.7 () {58 vars in 1375 bytes} [Fri Nov  8 11:19:57 2019] GET /admin/ratom/message/ => generated 28852 bytes in 129 msecs (HTTP/1.1 200) 10 headers in 513 bytes (1 switches on core 2)
    [pid: 14|app: 0|req: 5/15] 10.52.1.7 () {60 vars in 1271 bytes} [Fri Nov  8 11:20:32 2019] POST /graphql => generated 240 bytes in 30 msecs (HTTP/1.1 200) 8 headers in 400 bytes (1 switches on core 1)

    # start a shell in a pod, where you can run management commands, etc.
    $ kubectl exec -it api-55c4fbb789-b8m2v bash
    root@api-55c4fbb789-b8m2v:/code#

    # copy a file to a pod
    $ kubectl cp /path/to/source api-55c4fbb789-b8m2v:/path/to/dest
```


## Update Python requirements

If ``requirements.in`` is updated, run:

```
pip-compile --upgrade requirements.in
pip-sync requirements.txt
```
