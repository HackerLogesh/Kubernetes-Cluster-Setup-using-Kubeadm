Kubernetes Cluster Setup using Kubeadm
This guide will walk you through setting up a Kubernetes cluster using kubeadm on Linux systems (Ubuntu/CentOS).

Prerequisites
At least 2 machines (1 master, 1+ worker nodes) - can be VMs or physical servers

Each machine should have:

2+ GB RAM (4+ recommended for production)

2+ CPU cores

20+ GB disk space

Unique hostname, MAC address, and product_uuid for each node

Swap disabled

Root or sudo privileges on all machines

Full network connectivity between all machines

Step 1: Prepare All Nodes
1.1 Update system packages
bash
# For Ubuntu/Debian
sudo apt update && sudo apt upgrade -y

# For CentOS/RHEL
sudo yum update -y
1.2 Disable swap
bash
sudo swapoff -a
# To make it permanent, comment out swap lines in /etc/fstab
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
1.3 Set hostnames (do this on each node)
bash
# On master node
sudo hostnamectl set-hostname k8s-master

# On worker nodes
sudo hostnamectl set-hostname k8s-worker1
1.4 Add hosts entries (optional but recommended)
Edit /etc/hosts on all nodes:

bash
sudo nano /etc/hosts
Add entries like:

192.168.1.100 k8s-master
192.168.1.101 k8s-worker1
192.168.1.102 k8s-worker2
1.5 Enable kernel modules and sysctl params
bash
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
Step 2: Install Container Runtime (containerd)
2.1 Install containerd
bash
# For Ubuntu/Debian
sudo apt install -y containerd

# For CentOS/RHEL
sudo yum install -y containerd
2.2 Configure containerd
bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
2.3 Enable and start containerd
bash
sudo systemctl restart containerd
sudo systemctl enable containerd
Step 3: Install Kubernetes Components
3.1 Add Kubernetes repository
bash
# For Ubuntu/Debian
sudo apt install -y apt-transport-https ca-certificates curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# For CentOS/RHEL
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
3.2 Install kubeadm, kubelet and kubectl
bash
# For Ubuntu/Debian
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# For CentOS/RHEL
sudo yum install -y kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
Step 4: Initialize the Master Node
4.1 Initialize the cluster
Run this only on the master node:

bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
This will output a kubeadm join command - save this for later when adding worker nodes.

4.2 Set up kubeconfig for regular user
bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
4.3 Install a Pod Network Add-on (Flannel)
bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
Step 5: Join Worker Nodes
On each worker node, run the kubeadm join command that was output during the master initialization:

bash
sudo kubeadm join <master-ip>:<port> --token <token> --discovery-token-ca-cert-hash <hash>
Step 6: Verify the Cluster
On the master node, run:

bash
kubectl get nodes
You should see all nodes listed (master and workers). Initially they'll show "NotReady" status until the networking is fully configured (usually within a minute).

Step 7: (Optional) Remove Taint from Master Node
By default, the master node won't schedule pods. To allow scheduling on the master (for single-node clusters or testing):

bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
Post-Installation Steps
Install metrics server (for kubectl top commands):

bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
Install dashboard (optional):

bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
Create dashboard user:

bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF

cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
Get dashboard token:

bash
kubectl -n kubernetes-dashboard create token admin-user
Access dashboard:

bash
kubectl proxy
Then access at: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

Troubleshooting
If nodes don't join properly, check:

Network connectivity between nodes

Firewall rules (ports 6443, 10250, etc.)

Time synchronization (install NTP if needed)

To reset a node if something goes wrong:

bash
sudo kubeadm reset
Check component status:

bash
kubectl get componentstatuses
View pod status:

bash
kubectl get pods --all-namespaces
This completes the basic Kubernetes cluster setup using kubeadm. You can now start deploying applications to your cluster!
