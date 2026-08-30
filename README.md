# Kubernetes

# Connecting Minikube (Hyper-V on Windows 11) to WSL2 (Ubuntu)

This guide walks through setting up **Minikube** on Windows 11 using the **Hyper-V** driver, while routing `kubectl` commands directly from **WSL2 (Ubuntu)** to bypass network isolation issues between Hyper-V and WSL2.

---

## Prerequisites
- Windows 11 Pro / Enterprise / Education (with Hyper-V enabled)
- WSL2 with Ubuntu installed
- Minikube installed on Windows

---

## Step 1: Start Minikube on Windows

Open **PowerShell as Administrator** and create the Minikube virtual machine:

```powershell
# Initial cluster creation
minikube start --memory=4096 --driver=hyperv

(Note: For all future starts, you only need to run minikube start without extra flags).

Verify that kubectl works natively on the Windows host:

PowerShell
kubectl get nodes

Step 2: Configure WSL2 (Ubuntu) Aliases
Due to default network isolation between the Hyper-V switch and WSL2, direct IP communication can time out. To resolve this, configure your WSL2 Ubuntu shell to call the host powershell.exe wrapper seamlessly.

Open your WSL2 Ubuntu terminal and run:

# Clean up any existing broken aliases in ~/.bashrc
sed -i '/alias kubectl=/d' ~/.bashrc
sed -i '/alias minikube=/d' ~/.bashrc

# Add permanent working aliases to ~/.bashrc
echo 'alias kubectl="/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe kubectl"' >> ~/.bashrc

# Reload your shell profile
source ~/.bashrc

Step 3: Verify Connection in WSL2 Ubuntu
Run the following command inside your Ubuntu terminal:

Bash
kubectl get nodes

Expected Output:
Plaintext
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1m    v1.35.1
================================================

Start the Cluster (PowerShell):

PowerShell
minikube start
Work inside Linux (WSL2 Ubuntu):
Open Ubuntu and execute your Kubernetes manifests directly:

Bash
kubectl get nodes
kubectl apply -f deployment.yaml
Stop the Cluster (PowerShell):

PowerShell
minikube stop
