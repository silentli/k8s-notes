# SOP: Manual Kubernetes Installation on VMs - 03. Worker Node Setup

## Objective
This procedure coordinates the remote worker deployment phase. It involves selectively installing required node agents, authenticating against the existing Control Plane, and structurally validating integration into the active cluster.

*Note: Ensure the steps from **01. Common Prerequisites and OS Setup** have been fully completed on the target machine prior to this.*

## 1. Kubernetes Binaries Installation
**Do not** install `kubectl` on this node. Only install `kubelet` and `kubeadm`.
1. Download the public signing key for the Kubernetes package repositories:
   ```bash
   sudo mkdir -p -m 755 /etc/apt/keyrings
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg --yes
   ```
2. Add the appropriate Kubernetes `apt` repository:
   ```bash
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```
3. Update the package index and install strictly `kubelet` and `kubeadm`:
   ```bash
   sudo apt-get update
   sudo apt-get install -y kubelet kubeadm
   ```
4. Lock the packages to prevent automatic updates:
   ```bash
   sudo apt-mark hold kubelet kubeadm
   ```

## 2. DNS and Control Plane Reachability Verification
Confirm that the worker instance can establish a clear connection pathway to the Control Plane API server.
1. Run `nc` (netcat) from the worker node to see if the connection is open to the Master node (e.g., if Master IP is `192.168.30.10`):
   ```bash
   nc -vz 192.168.30.10 6443
   # or
   nc -vz [IP] [PORT]
   ```
2. **The Fallback (If DNS fails):** If using a hostname that `nc` fails to resolve, append it to the `/etc/hosts` file:
   ```bash
   echo "192.168.30.10 k8s-master-01" | sudo tee -a /etc/hosts
   ```

## 3. The Cryptographic Join Process
Authorize the connection and ingest the worker computing power.
1. **On the MASTER node:** Generate a fresh invitation token.
   ```bash
   kubeadm token create --print-join-command
   ```
2. **On the WORKER node:** Copy the output exactly from the Master node, append `sudo`, and execute it.
   ```bash
   sudo kubeadm join 192.168.30.10:6443 --token abcdef.0123456789abcdef \
           --discovery-token-ca-cert-hash sha256:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
   ```

## 4. Control Plane fully registers node (On the MASTER node)
Perform system inspection steps from the Control Plane view to verify total integration.
1. Check the High-Level Node Status:
   ```bash
   kubectl get nodes
   ```
2. Inspect the Node Conditions (Deep Dive):
   ```bash
   kubectl describe node k8s-worker-01
   ```
3. Verify the System Containers (DaemonSets):
   ```bash
   kubectl get pods -n kube-system --field-selector spec.nodeName=k8s-worker-01
   ```
