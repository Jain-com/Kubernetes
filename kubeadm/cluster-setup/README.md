# Kubernetes Cluster Setup using kubeadm

## Architecture

1 Control Plane + 2 Worker Nodes

```text
                AWS VPC
                   |
        +----------+----------+
        |                     |
   Control Plane          Worker Nodes
   10.0.0.10              10.0.0.11
                         10.0.0.12

<img width="1053" height="317" alt="image" src="https://github.com/user-attachments/assets/319af3b5-3533-41a8-98e0-077816d92dee" />
Control Plane SG
<img width="1659" height="549" alt="image" src="https://github.com/user-attachments/assets/73497203-52ee-4388-b8dc-9679a5c3b493" />
Worker nodes SG
<img width="1641" height="616" alt="image" src="https://github.com/user-attachments/assets/db900cfb-945d-4789-b5eb-ac0064935a06" />

# Disable swap on all 3 VMs
swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Do on all 3 VM's
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots (3 VM's)
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot (3 VM's)
sudo sysctl --system

# Verify that the br_netfilter, overlay modules are loaded by running the following commands: (3 VM's)
lsmod | grep br_netfilter
lsmod | grep overlay

# Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command: (3 VM's)
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward


# Install container runtime (3 VM's)


curl -LO https://github.com/containerd/containerd/releases/download/v1.7.14/containerd-1.7.14-linux-amd64.tar.gz

sudo tar Cxzvf /usr/local containerd-1.7.14-linux-amd64.tar.gz

curl -LO https://raw.githubusercontent.com/containerd/containerd/main/containerd.service

sudo mkdir -p /usr/local/lib/systemd/system/
sudo mv containerd.service /usr/local/lib/systemd/system/
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

# Check that containerd service is up and running (3 VM's)
systemctl status containerd

# install runc (3 VM's)

curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc

# Install cni plugins (3 VM's)

curl -LO https://github.com/containernetworking/plugins/releases/download/v1.5.0/cni-plugins-linux-amd64-v1.5.0.tgz

sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.0.tgz

# Install kubeadm, kubelet and kubectl (3 VM's)

sudo yum update
sudo yum install -y ca-certificates curl gpg --allowerasing
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
sudo tee /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
sudo yum install -y kubelet-1.29.6-1.1 kubeadm-1.29.6-1.1 kubectl-1.29.6-1.1 --disableexcludes=kubernetes
sudo yum install -y kubelet-1.29.6-150500.1.1 kubeadm-1.29.6-150500.1.1 kubectl-1.29.6-150500.1.1 --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
sudo dnf install -y 'dnf-command(versionlock)'
sudo dnf versionlock add kubelet kubeadm kubectl

kubeadm version
kubelet --version
kubectl version --client

# Configure crictl to work with containerd (3 VM's)
sudo crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock
crictl ps

# Initialize the control plane (control VM)
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=10.0.0.10 --node-name master

 mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubeadm join 10.0.0.10:6443 --token 9i4ysb.c9fcysxegczf8ce6 \
        --discovery-token-ca-cert-hash sha256:716476b8c6637128e7f082ae1c181f1731024df9a7313ead45e125afa3bdfd21  

# Install calico (control VM)
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/v3_projectcalico_org.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml
curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/custom-resources.yaml -O
kubectl apply -f custom-resources.yaml

kubectl get pods -A

# Perform the below steps on both the worker nodes
 #Perform steps 1-8 on both the nodes
 #Run the join command that was generated in step 9 on the Master node, which is similar to the below

sudo kubeadm join <private-ip-of-master-node>:6443 --token xxxxx --discovery-token-ca-cert-hash sha256:xxx

sudo kubeadm join 10.0.0.10:6443 --token 9i4ysb.c9fcysxegczf8ce6 --discovery-token-ca-cert-hash sha256:716476b8c6637128e7f082ae1c181f1731024df9a7313ead45e125afa3bdfd21

#If you forgot to copy the command, you can execute the below command on the master node to generate the join command again

kubeadm token create --print-join-command

Once all steps are done run
kubectl get nodes

<img width="734" height="138" alt="image" src="https://github.com/user-attachments/assets/cd1ea9cb-0ad3-4562-82f9-2f377961d441" />


# run in worker nodes
mkdir -p $HOME/.kube
sudo cp -p /etc/kubernetes/kubelet.conf $HOME/.kube
