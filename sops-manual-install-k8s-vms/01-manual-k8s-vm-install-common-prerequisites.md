# SOP: Manual Kubernetes Installation on VMs - 01. Common Prerequisites and OS Setup

## Objective
This procedure outlines the initial setup phase for both Control Plane and Worker nodes required for a Kubernetes cluster, covering the Operating System installation, network settings, system requirements, and the container runtime installation.

## 0. Prerequisite: Check the IPs
If utilizing a standard enterprise Linux DHCP server (`isc-dhcp-server`), verify the expected MAC assignments and live leases by SSHing into the dedicated DHCP server (not the Kubernetes nodes).
1. Check the DHCP Reservations (expected fixed IPs):
   ```bash
   cat /etc/dhcp/dhcpd.conf | grep -A 3 "host"
   ```
2. Check the Active DHCP Leases (dynamic assignments):
   ```bash
   cat /var/lib/dhcp/dhcpd.leases
   ```

## 1. Operating System Installation (Debian 12)
1. Download the Debian 12 "netinst" ISO.
2. In the hypervisor (e.g., ESXi), create a VM with the following hardware profile:
   - **OS version:** Debian GNU/Linux 12 (64-bit)
   - **Compute:** 4 vCPUs, 16GB RAM
   - **Storage:** 50GB Hard Disk, Thin Provisioned
   - **Network:** Connect the Network Adapter strictly to the designated Kubernetes VLAN (e.g., VLAN 30).
3. Mount the ISO and boot the installation process. Configure the following settings when prompted:
   - **Hostname:** Assign an exact role-based hostname (e.g., `k8s-master-01` or `k8s-worker-01`).
   - **Domain Name:** Assign the internal company domain.
   - **User Setup:** Leave the root password completely blank to disable root login. Create a standard administrative user, which will automatically be granted `sudo` privileges.
   - **Disk Partitioning:** Choose manual partitioning. Create a single `ext4` or `xfs` partition mounted to the root (`/`). **Ensure no swap partition is created.**
   - **Software Selection (tasksel):** Uncheck all desktop environments. Only check `SSH server` and `Standard system utilities`.
4. Complete the installation, reboot, and connect to the server via SSH.

## 2. Hostname Configuration (If required post-installation)
If the hostname was not configured correctly during installation, it must be updated before running `kubeadm init`. For the master node as an example:
1. Run this command to update the hostname dynamically:
   ```bash
   sudo hostnamectl set-hostname k8s-master-01
   ```

## 3. Disable Swap Memory
Kubernetes scheduling relies on strict memory predictability. Swap memory must be disabled completely.
1. Disable swap temporarily for the active session:
   ```bash
   sudo swapoff -a
   ```
2. Disable swap permanently to survive reboots by editing `/etc/fstab` and commenting out any swap entries:
   ```bash
   # /etc/fstab example:
   # /swapfile none swap sw 0 0
   ```
3. Verify that swap space shows 0 using:
   ```bash
   free -h
   ```

## 4. Kernel Modules Configuration
Kernel modules must be loaded to enable overlay filesystem support and bridge netfilter routing.
1. Verify if the modules are already loaded:
   ```bash
   lsmod | grep -e overlay -e br_netfilter
   ```
2. Create a persistent configuration to load modules on boot:
   ```bash
   cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
   overlay
   br_netfilter
   EOF
   ```
3. Load the modules into the current RAM session:
   ```bash
   sudo modprobe overlay
   sudo modprobe br_netfilter
   ```
4. Verify the execution was successful:
   ```bash
   lsmod | grep -e overlay -e br_netfilter
   ```

## 5. Network Routing Rules (sysctl)
Configure the kernel to properly route bridge IPv4 traffic through `iptables` for container network management.
1. Create a persistent configuration file:
   ```bash
   cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-iptables  = 1
   net.ipv4.ip_forward                 = 1
   net.bridge.bridge-nf-call-ip6tables = 1
   EOF
   ```
2. Apply the rules to the running kernel without rebooting:
   ```bash
   sudo sysctl --system
   ```
3. Verify the execution was successful:
   ```bash
   sysctl net.ipv4.ip_forward
   ```

## 6. Container Runtime Setup (containerd)
Install and configure `containerd` as the execution engine for the cluster.
1. Install essential cryptographic tools:
   ```bash
   sudo apt-get update
   sudo apt-get install -y ca-certificates curl gnupg
   ```
2. Download the official Docker GPG key:
   ```bash
   sudo install -m 0755 -d /etc/apt/keyrings
   curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
   sudo chmod a+r /etc/apt/keyrings/docker.gpg
   ```
3. Add the Docker `apt` repository:
   ```bash
   echo \
     "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
     $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
     sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   ```
4. Update the package list and install the runtime:
   ```bash
   sudo apt-get update
   sudo apt-get install -y containerd.io
   ```

## 7. Cgroup Manager Configuration (Systemd Integration)
Configure `containerd` to delegate CPU and RAM limits to `systemd` to prevent resource management conflicts.
1. Generate the default configuration file and overwrite the blank one:
   ```bash
   sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
   ```
2. Modify the file to enable the SystemdCgroup integration:
   ```bash
   sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
   ```
3. Restart the service and enable it on boot:
   ```bash
   sudo systemctl restart containerd
   sudo systemctl enable containerd
   ```
4. Verify the execution was successful:
   ```bash
   sudo systemctl status containerd
   ```

## 8. Network Troubleshooting (DNS Issue Resolution)
If `kubeadm` fails to download images with "no such host" errors due to DNS concurrent lookup conflicts (Go resolver vs hypervisor proxy):
1. Overwrite the Debian network configuration to point directly to a robust public DNS server (e.g., `8.8.8.8`):
   ```bash
   sudo rm /etc/resolv.conf
   echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
   ```
