# Container Orchestrator at Spanish SKA Regional Centre

  * [Requirements](#requirements)
  * [Kubernetes](#kubernetes)
    + [Kubernetes deployment based on Ansible - KubeSpray](#kubernetes-deployment-based-on-ansible---kubespray)
    + [Kubernetes deployment based on Ansible](#kubernetes-deployment-based-on-ansible)
    + [Manual installation of Kubernetes](#manual-installation-of-kubernetes)
  * [Rancher](#rancher)
    + [Development option: One node - Default Rancher-generated Self-signed Certificate and CA Certificate](#development-option--one-node---default-rancher-generated-self-signed-certificate-and-ca-certificate)
    + [Rancher of Kubernetes](#rancher-of-kubernetes)
    + [Add a new Kubernetes Cluster to Rancher](#add-a-new-kubernetes-cluster-to-rancher)
    + [Manage federated cluster with Rancher and kubernetes](#manage-federated-cluster-cli)


## Requirements

- Two nodes/VMs with access to Internet and internal/external IP. One VM will act as Master (node1) and the another as worker(node2).
 - 4 CPUs and 32 GB of RAM, 100GB of storage for each VM and Ubuntu 20.04 installed.
- One VM/node where you have access to both VMs using you SSH public key (seed node).
 - You have to copy you public key in both VMs for the same user (in this tutorial: ubuntu is the user). 

## Kubernetes

### Kubernetes deployment based on Ansible - KubeSpray 

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

### Kubernetes deployment based on Ansible

First, go to your Seed node.

Clone the repo in the seed node

```
git clone https://github.com/manuparra/orchestrator-at-spsrc.git
```
Go to kubernetes-ansible folder, and edit the inventories with your data
```
vi inventories/list
```
Launch the playbook
```
ansible-playbook -i inventories/list Kubernetes.yml
```
The installation consists of the following points:
* Install dockers
* Disable SElinux
* Disable firewalld
* Install kubernetes

Once the installation is finished, you have to follow the next steps:

1.- On the master node you have to execute:
```
kubeadm init --apiserver-advertise-address=<IP-master-node> --pod-network-cidr=10.244.0.0/16
```
Once the process is finished, you can see the ``discovery token``.
For example:
```
kubeadm join IP-master-node:6443 --token 31eeyr.uxvpz3b0teajkaki \
        --discovery-token-ca-cert-hash sha256:f2caf4fe6f26dc6cc8188b55ee3c825e5b8d9779a61bfd4eda4b152627e56184
```
You will need this token for the worker node.

Set cluster admin user
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```
Configure Pod Network with Flannel
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

Check deploy status
```
kubectl get pods --all-namespaces
```

2.- In the Worker Node you have to run:
```
kubeadm join <IP-master-node>:6443 --token 31eeyr.uxvpz3b0teajkaki \
--discovery-token-ca-cert-hash sha256:f2caf4fe6f26dc6cc8188b55ee3c825e5b8d9779a61bfd4eda4b152627e56184
```

Check status
```
kubectl get nodes
```


### Manual installation of Kubernetes

Pre-requisites:

- Start installing Docker for all the nodes (workers and master). 
- Remove all the rules within the firewall.
- Disable SELinux.

**Installation**

Disable swap:

```
swapoff -a
```

```
vi /etc/fstab
# disable swap

#/dev/mapper/cl-swap swap swap defaults 0 0 
```

Apply forwarding: 

```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```

Then: ```sysctl --system```


Add a Kubernetes repo:
```
cat <<'EOF' > /etc/yum.repos.d/kubernetes.repo

[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-$basearch
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

Then:


```
yum -y install kubeadm kubelet kubectl 
```

And

```
systemctl enable kubelet 
```

Initialise the master node:


```
kubeadm init --apiserver-advertise-address=192.168.250.100 --pod-network-cidr=10.244.0.0/16 
```
*Note: Change pod network if needed. And check if you Firewall is not blocking it*

Then enable kubectl from the user:

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Command ``kubeadmin init ... `` will give you an output like the next:


```
kubeadm join 192.168.250.100:6443 --token 31eeyr.uxvpz3b0teajkaki \
        --discovery-token-ca-cert-hash sha256:f2caf4fe6f26dc6cc8188b55ee3c825e5b8d9779a61bfd4eda4b152627e56184
```

Execute it on all the workers nodes and then go to [Testing this deployment](#testing-this-deployment), in order to check if the platform is working.

**Next step is to configure StorageClass by using NFS**

1. Install NFS service on an external storage resource (folder ``/data/nfs1``).
2. Install NFS client on each node (*all workers and master*).

Then go to the Master Node and execute:

*You have to install HELM for Kubernetes: https://helm.sh/docs/intro/install/*

```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
```

Then install StorageClass provisioner:

```
helm install nfs-subdir-external-provisioner \
nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
--set nfs.server=192.168.250.50 \
--set nfs.path=/data/nfs1 \
--set storageClass.onDelete=true
```

Then check if storageclass is installed:

```
kubectl get storageclass nfs-client
```

In this point you can use ``nfs-client`` as StorageClass for you deployments in Kubernetes.

For example:


```
...
 storage:
    dynamic:
      storageClass: nfs-client
    extraVolumes:
...
```

### Testing this deployment 

Create the next file `run-my-nginx.yml`:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80

```

And now the service ``nginx-svc.yaml``:

```
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 80
    protocol: TCP
    name: http
  selector:
    run: my-nginx

```

Then execute:

```
kubectl apply -f run-my-nginx.yml
```

and wait until this command be finished: 

```
kubectl get pods -l run=my-nginx -o wide
```

And now start the service:

```
kubectl apply -f nginx-svc.yaml
```

Check the status:

```
kubectl get pods -l run=my-nginx -o wide
```

And check the port enabled to access externally:

```
kubectl get svc my-nginx -o yaml | grep nodePort -C 5
```

Use your CURL tool or a browser to access http://<localhost or IP>:31641: 

```
curl http://<localhost or IP>:31641
```


## Rancher

Rancher has different solutions to deploy this platforms (Dev / Production)

### Development option: One node - Default Rancher-generated Self-signed Certificate and CA Certificate

If you are installing Rancher in a development or testing environment where identity verification isn't a concern, install Rancher using the self-signed certificate that it generates. This installation option omits the hassle of generating a certificate yourself.

Log into your host, and run the command below:

```
docker run -d --restart=unless-stopped -p 80:80 -p 443:443 --privileged  rancher/rancher:latest
```

or with your CAs:

```
docker run -d --restart=unless-stopped  -p 80:80 -p 443:443   -v /<CERT_DIRECTORY>/<FULL_CHAIN.pem>:/etc/rancher/ssl/cert.pem   -v /<CERT_DIRECTORY>/<PRIVATE_KEY.pem>:/etc/rancher/ssl/key.pem   -v /<CERT_DIRECTORY>/<CA_CERTS.pem>:/etc/rancher/ssl/cacerts.pem   --privileged   rancher/rancher:latest
```

The Rancher backup operator can be used to migrate Rancher from the single Docker container install to an installation on a high-availability Kubernetes cluster. For details, refer to the documentation on migrating Rancher to a new cluster.


### Rancher of Kubernetes

**Add the Helm Chart Repository**

Use helm repo add command to add the Helm chart repository that contains charts to install Rancher. For more information about the repository choices and which is best for your use case, see Choosing a Version of Rancher.

Latest: Recommended for trying out the newest features

```
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
```

Stable: Recommended for production environments

```
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
```

**Create a Namespace for Rancher**

We'll need to define a Kubernetes namespace where the resources created by the Chart should be installed. This should always be cattle-system:

```
kubectl create namespace cattle-system
```

**Choose your SSL Configuration**

The Rancher management server is designed to be secure by default and requires SSL/TLS configuration.

- Let's Encrypt: The Let's Encrypt option also uses cert-manager. However, in this case, cert-manager is combined with a special Issuer for Let's Encrypt that performs all actions (including request and validation) necessary for getting a Let's Encrypt issued cert. This configuration uses HTTP validation (HTTP-01), so the load balancer must have a public DNS record and be accessible from the internet.
- Your own certificate: This option allows you to bring your own public- or private-CA signed certificate. Rancher will use that certificate to secure websocket and HTTPS traffic. In this case, you must upload this certificate (and associated key) as PEM-encoded files with the name tls.crt and tls.key. If you are using a private CA, you must also upload that certificate. This is due to the fact that this private CA may not be trusted by your nodes. Rancher will take that CA certificate, and generate a checksum from it, which the various Rancher components will use to validate their connection to Rancher.

```
Rancher Generated Certificates (Default)	ingress.tls.source=rancher	yes
Let’s Encrypt	ingress.tls.source=letsEncrypt	yes
Certificates from Files	ingress.tls.source=secret	no
```

**Install Rancher with Helm and Your Chosen Certificate Option**

The exact command to install Rancher differs depending on the certificate configuration. However, irrespective of the certificate configuration, the name of the Rancher installation in the cattle-system namespace should always be rancher.

This option uses cert-manager to automatically request and renew Let's Encrypt certificates. This is a free service that provides you with a valid certificate as Let's Encrypt is a trusted CA.
note

In the following command,

>    hostname is set to the public DNS record,
>    Set the bootstrapPassword to something unique for the admin user.
>    ingress.tls.source is set to letsEncrypt
>    letsEncrypt.email is set to the email address used for communication about your certificate (for example, expiry notices)
>    Set letsEncrypt.ingress.class to whatever your ingress controller is, e.g., traefik, nginx, haproxy, etc.


```
helm install rancher rancher-<CHART_REPO>/rancher \
  --namespace cattle-system \
  --set hostname=rancher.my.org \
  --set bootstrapPassword=admin \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=me@example.org \
  --set letsEncrypt.ingress.class=nginx
```

If you are installing an alpha version, Helm requires adding the --devel option to the install command:

*Note: change to the stable version!*

```
helm install rancher rancher-alpha/rancher --devel
```

Wait for Rancher to be rolled out:

```
kubectl -n cattle-system rollout status deploy/rancher
Waiting for deployment "rancher" rollout to finish: 0 of 3 updated replicas are available...
deployment "rancher" successfully rolled out
```

**Verify that the Rancher Server is Successfully Deployed**

After adding the secrets, check if Rancher was rolled out successfully:

```
kubectl -n cattle-system rollout status deploy/rancher
Waiting for deployment "rancher" rollout to finish: 0 of 3 updated replicas are available...
deployment "rancher" successfully rolled out
```

**Test**

In a web browser, go to the DNS name that forwards traffic to your load balancer.

### Add a new Kubernetes Cluster to Rancher
 
Go to ``Import Cluster``:
 
![imagen](https://user-images.githubusercontent.com/7033451/195204549-2fd7df46-8161-42a9-bddf-5dc334c92a72.png)

Add a Cluster Name and Description, and finally click "Create".
 
Then:
 
Run the kubectl command below on an existing Kubernetes cluster running a supported Kubernetes version to import it into Rancher:

```
kubectl apply -f https://161.111.167.189:18019/v3/import/dtkltbwtcjzzrx4jtfc4d87sl54wc5lgw6xlx8v7gz7j4ncl6l967h_c-m-4hm5dj9p.yaml
```
 
If you get a "certificate signed by unknown authority" error, your Rancher installation has a self-signed or untrusted SSL certificate. Run the command below instead to bypass the certificate verification:

```
curl --insecure -sfL https://161.111.167.189:18019/v3/import/dtkltbwtcjzzrx4jtfc4d87sl54wc5lgw6xlx8v7gz7j4ncl6l967h_c-m-4hm5dj9p.yaml | kubectl apply -f -
```

If you get permission errors creating some of the resources, your user may not have the cluster-admin role. Use this command to apply it:

```
kubectl create clusterrolebinding cluster-admin-binding --clusterrole cluster-admin --user <your username from your kubeconfig> 
```

After this command you will see:
 
![imagen](https://user-images.githubusercontent.com/7033451/195205044-a0f51fc0-3483-4493-a7ed-a5b4dd42b31d.png)

SPSRC Kubernetes Cluster is enabled and ready to use.
 

### Manage federated cluster CLI
 
The Rancher CLI (Command Line Interface) is a unified tool that you can use to interact with Rancher. With this tool, you can operate Rancher using a command line rather than the GUI.

 https://docs.ranchermanager.rancher.io/reference-guides/cli-with-rancher/rancher-cli
 
**Project Selection**

Before you can perform any commands, you must select a Rancher project to perform those commands against. To select a project to work on, use the command ``./rancher context switch``. When you enter this command, a list of available projects displays. Enter a number to choose your project.

Example: ``./rancher context switch`` Output

Ensure you can run ``rancher kubectl get pods`` successfully.
 
**Commands**

The following commands are available for use in Rancher CLI.
 
- apps, [app]	Performs operations on catalog applications (i.e., individual Helm charts) or Rancher charts.
- catalog	Performs operations on catalogs.
- clusters, [cluster]	Performs operations on your clusters.
- context	Switches between Rancher projects. For an example, see Project Selection.
- inspect [OPTIONS] [RESOURCEID RESOURCENAME]	Displays details about Kubernetes resources or Rancher resources (i.e.: projects and workloads). Specify resources by name or ID.
- kubectl	Runs kubectl commands.
- login, [l]	Logs into a Rancher Server. For an example, see CLI Authentication.
- namespaces, [namespace]	Performs operations on namespaces.
- nodes, [node]	Performs operations on nodes.
- projects, [project]	Performs operations on projects.
- ps	Displays workloads in a project.
- settings, [setting]	Shows the current settings for your Rancher Server.
- ssh	Connects to one of your cluster nodes using the SSH protocol.
 
