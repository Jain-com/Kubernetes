## In this i am going to perform version upgrade of kubernetes cluster from 1.29 to 1.30 kubeadm

# First Check if you have version lock
sudo dnf versionlock list
sudo dnf versionlock delete kubeadm

# Change to your desired version 1.29 to 1.30
sudo vi /etc/yum.repos.d/kubernetes.repo
sudo dnf clean all 

# Upgrading kubeadm (both nodes)
# Replace x with patch number eg 14
sudo dnf install -y kubeadm-'1.30.x-*' --disableexcludes=kubernetes

kubeadm version
sudo kubeadm upgrade plan

sudo kubeadm upgrade apply v1.30.14

# now we have to drain it
 kubectl drain master 
 kubectl drain master --ignore-daemonsets

kubectl get nodes


# To Check all running pods
 kubectl get pods -A -o wide

 # upgrade kubelet and kubectl
 sudo dnf install -y kubelet-'1.30.14-*' kubectl-'1.30.14-*' --disableexcludes=kubernetes

sudo dnf versionlock list
sudo dnf versionlock delete kubelet
sudo dnf versionlock delete kubectl
sudo dnf list --showduplicates kubelet kubectl --disableexcludes=all
sudo dnf install -y kubelet-'1.30.14-*' kubectl-'1.30.14-*' --disableexcludes=kubernetes
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```text
[ec2-user@ip-10-0-0-10 manifests]$ kubectl get nodes
NAME                        STATUS                     ROLES           AGE   VERSION
ip-10-0-0-12.ec2.internal   Ready                      <none>          34h   v1.29.6
ip-10-0-0-4.ec2.internal    Ready                      <none>          34h   v1.29.6
master                      Ready,SchedulingDisabled   control-plane   2d    v1.30.14
```
kubectl version
kubectl uncordon master
```text
[ec2-user@ip-10-0-0-10 manifests]$ kubectl get nodes
NAME                        STATUS   ROLES           AGE   VERSION
ip-10-0-0-12.ec2.internal   Ready    <none>          34h   v1.29.6
ip-10-0-0-4.ec2.internal    Ready    <none>          34h   v1.29.6
master                      Ready    control-plane   2d    v1.30.14
```

====================
# Working on worker node upgrade now

 sudo dnf versionlock list
 sudo dnf versionlock delete kubeadm
 sudo dnf versionlock delete kubelet
 sudo dnf versionlock delete kubectl

 # Change the version number
 sudo vi /etc/yum.repos.d/kubernetes.repo
 sudo dnf install -y kubeadm-'1.30.14-*' --disableexcludes=kubernetes

 sudo kubeadm upgrade node
 kubectl drain ip-10-0-0-12.ec2.internal --ignore-daemonsets

 # upgrade kubelet and kubectl
sudo dnf install -y kubelet-'1.30.14-*' kubectl-'1.30.14-*' --disableexcludes=kubernetes
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl uncordon ip-10-0-0-12.ec2.internal

# Now both master and worker we have completed the minor upgrade
<img width="632" height="96" alt="image" src="https://github.com/user-attachments/assets/f28a6523-bc7b-44b1-bfa5-f69c610727b4" />



 

