# ratom-deploy

## AWS, Ansible, microk8s

Collection of Ansible playbooks and roles to provision and deploy [MicroK8s](https://microk8s.io/) on AWS.


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

## Deprovision stack
ansible-playbook playbooks/deprovision.yml -vv
```

## Configure kubctl

- On the host, run ``microk8s.config``
- Copy the YAML to ``~/.kube/config``
- Modify ``clusters.0.cluster.server`` to match the host's IP

Now you should be able to run:

```sh
kubectl cluster-info
```


## Update Python requirements

If ``requirements.in`` is updated, run:

```
pip-compile --upgrade requirements.in
pip-sync requirements.txt
```
