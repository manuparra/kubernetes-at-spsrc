# Container Orchestrator at Spanish SKA Regional Centre

## Requirements

- Two nodes/VMs with access to Internet and internal/external IP. One VM will act as Master (node1) and the another as worker(node2).
 - 4 CPUs and 32 GB of RAM, 100GB of storage for each VM and Ubuntu 20.04 installed.
- One VM/node where you have access to both VMs using you SSH public key (seed node).
 - You have to copy you public key in both VMs for the same user (in this tutorial: ubuntu is the user). 

## Kubernetes deployment based on Ansible - KubeSpray 

First, go to your Seed node. 

Clone this repo in one VM that can access to the rest of the VMs where we will deploy our Kubernetes cluster
```
git clone https://github.com/kubernetes-sigs/kubespray
```
Then access to the folder:

```
cd kubespray
```

Copy ``inventory/sample`` as ``inventory/mycluster``

```
cp -rfp inventory/sample inventory/mycluster
```


Add the next env variables:

```
VENVDIR=kubespray-venv
KUBESPRAYDIR=kubespray
ANSIBLE_VERSION=2.11
```

Create un Virtual Env in Python to manage this deployment of tools:

```
virtualenv  --python=$(which python3) $VENVDIR
pip install -U -r requirements-2.11.txt 
```


Add the VMs or nodes where kubernetes will be installed:

```
declare -a IPS=(161.189.165.100,161.189.165.101)
CONFIG_FILE=inventory/mycluster/hosts.yaml 
```

Populate deployment files:

```
python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```

Review and change parameters under ``inventory/mycluster/group_vars``
cat inventory/mycluster/group_vars/all/all.yml
cat inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml


Modify ``inventory/mycluster/hosts.yaml`` with the next:

```
all:
  hosts:
    node1:
      ansible_host: 161.189.165.100
      ip: 192.168.250.72
      access_ip: 192.168.250.72
      ansible_ssh_port: 22
      ansible_user: ubuntu
    node2:
      ansible_host: 161.189.165.101
      ip: 192.168.250.91
      access_ip: 192.168.250.91
      ansible_ssh_port: 22
      ansible_user: ubuntu
  children:
    kube_control_plane:
      hosts:
        node1:
    kube_node:
      hosts:
        node2:
    etcd:
      hosts:
        node1:
    k8s_cluster:
      children:
        kube_control_plane:
          node1:
        kube_node:
          node2:
    calico_rr:
      hosts: {}
```

And then finally type the next:

```
ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml --private-key=~/.ssh/id_rsa
```




