# Kubernetes & Minikube on Windows WSL2 (Hyper-V Setup)

This guide documents the setup, fixes, and daily workflow for running standard Linux `kubectl` and `minikube` commands inside a WSL2 Ubuntu environment connected to a Minikube cluster running on Windows via Hyper-V.

---

## 🛠️ One-Time Setup & Configuration

### 1. Configure Native Linux `kubectl` in WSL2
To avoid errors caused by calling `powershell.exe` from Linux subdirectories (`Invalid argument`), use native Linux `kubectl` configured to target the Hyper-V host gateway.

Run these commands inside your **WSL Ubuntu terminal**:

```bash
# Clean up any existing PowerShell aliases
sed -i '/alias kubectl=/d' ~/.bashrc
unalias kubectl 2>/dev/null

# Sync Windows kubeconfig to Linux
mkdir -p ~/.kube
cp -f /mnt/c/Users/admin/.kube/config ~/.kube/config

# Normalize path separators
sed -i 's|\\|/|g' ~/.kube/config
sed -i 's|C:/Users/admin|/mnt/c/Users/admin|g' ~/.kube/config

# Point cluster endpoint to the Windows Host IP
WIN_IP=$(ip route | grep default | awk '{print $3}')
sed -i "s|https://[0-9.]*:8443|https://${WIN_IP}:8443|g" ~/.kube/config
kubectl config set-cluster minikube --insecure-skip-tls-verify=true
2. Configure Local Port Proxy on Windows
To allow WSL2 traffic on port 8443 to reach the Minikube Hyper-V virtual machine:

Open PowerShell as Administrator on Windows.

Run the following command:

PowerShell
netsh interface portproxy add v4tov4 listenport=8443 listenaddress=0.0.0.0 connectport=8443 connectaddress=(minikube ip)
3. Add Custom SSH Shortcut to WSL2 (minikube-ssh)
Standard minikube ssh fails when executed inside WSL subdirectories. Set up a native Linux SSH shortcut using the cluster's RSA key:

Run these commands in your WSL Ubuntu terminal:

Bash
# 1. Copy key to native Linux environment with safe permissions
mkdir -p ~/.ssh
cp /mnt/c/Users/admin/.minikube/machines/minikube/id_rsa ~/.ssh/minikube_id_rsa
chmod 600 ~/.ssh/minikube_id_rsa

# 2. Add function to ~/.bashrc
cat << 'EOF' >> ~/.bashrc

# Custom shortcut to SSH directly into Minikube node from WSL2
minikube-ssh() {
  WIN_IP=$(ip route | grep default | awk '{print $3}')
  ssh -i ~/.ssh/minikube_id_rsa -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null docker@$WIN_IP -p 22
}
EOF

# 3. Reload environment
source ~/.bashrc

🚀 Daily Workflow
1. Starting the Environment
When powering on your machine:

Open PowerShell and start Minikube:

PowerShell
minikube start
Open WSL Ubuntu and verify the cluster status:

Bash
kubectl get nodes
2. Common Operations (Inside WSL2)
Deploy Pods / Services:

Bash
kubectl create -f pods.yml
kubectl get po -o wide
SSH into the Minikube VM:

Bash
minikube-ssh
Test Pods directly inside Node shell:

Bash
# Inside minikube-ssh session
curl http://<POD_IP>
exit
3. Shutting Down Cleanly
Before shutting down your PC, gracefully stop the cluster to prevent state corruption:

Exit the SSH session (if logged in):

Bash
exit
Open PowerShell and stop the VM:

PowerShell
minikube stop
