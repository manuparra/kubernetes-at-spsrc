# Container Orchestrator at Spanish SKA Regional Centre

  * [Requirements](#requirements)
  * [Kubernetes](#kubernetes)
    + [Kubernetes deployment based on Ansible - KubeSpray](#kubernetes-deployment-based-on-ansible---kubespray)
    + [Kubernetes deployment based on Ansible](#kubernetes-deployment-based-on-ansible)
    + [Manual installation of Kubernetes](#manual-installation-of-kubernetes)
  * [Rancher](#rancher)
    + [Development option: One node - Default Rancher-generated Self-signed Certificate and CA Certificate](#development-option--one-node---default-rancher-generated-self-signed-certificate-and-ca-certificate)
    + [Rancher of Kubernetes](#rancher-of-kubernetes)



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


```
swapoff -a
```


```
vi /etc/fstab
# disable swap

#/dev/mapper/cl-swap swap swap defaults 0 0 
```


```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```

Then: ```sysctl --system```


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

Execute it on all the workers nodes.





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



