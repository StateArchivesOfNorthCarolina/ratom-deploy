# ratom-deploy

Ansible + AWS + K8s


## Setup

```sh
pip install -r requirements.txt
ansible-galaxy install -f -r requirements.yml -p roles/
```

## Provision and Deploy

```sh
## Provision
ansible-playbook playbooks/provision.yml -vv

## Deploy
ansible-playbook -u ubuntu playbooks/deploy.yml -vv

## Deprovision
ansible-playbook playbooks/deprovision.yml -vv
```


## Update requirements

```
pip-compile --upgrade requirements.in
pip-sync requirements.txt
```
