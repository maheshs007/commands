```markdown
# Creating a Kubernetes Cluster Using Chef with Cilium CNI and kube-proxy Replacement

This guide explains how to create a Kubernetes cluster using Chef, configured to use **Cilium** as the CNI and to replace `kube-proxy` with Cilium's eBPF-based implementation for improved network performance and security.

---

## Prerequisites

1. **Chef Workstation Setup**:
   - Install Chef Workstation: [Chef Workstation Documentation](https://docs.chef.io/workstation/).
   - Configure `knife` to connect to your Chef server.

2. **Chef Server and Nodes**:
   - Ensure Chef Server is installed and accessible.
   - Add all nodes (control plane and workers) to the Chef Server.

3. **Cilium Prerequisites**:
   - Kernel version 4.19+ (5.10+ recommended) for eBPF.
   - Ensure `bpftool` and `iproute2` are installed on all nodes.

4. **Cookbooks**:
   - Create or use a cookbook to:
     - Install Kubernetes packages (`kubeadm`, `kubelet`, `kubectl`).
     - Install Cilium and enable kube-proxy replacement.

---

## Steps to Create the Cluster with Chef and Cilium

### 1. **Prepare the Cookbook**
   - Generate a new Chef cookbook:
     ```bash
     chef generate cookbook k8s-cluster-cilium
     ```
   - Include recipes for:
     - Installing Kubernetes packages.
     - Setting up Cilium as the CNI with kube-proxy replacement.

### 2. **Install Kubernetes and Prerequisites**
   - Add a recipe to install Kubernetes components:
     ```ruby
     package %w(kubeadm kubelet kubectl) do
       action :install
     end

     execute 'disable swap' do
       command 'swapoff -a && sed -i "/swap/d" /etc/fstab'
     end

     execute 'load kernel modules' do
       command 'modprobe br_netfilter && echo "1" > /proc/sys/net/bridge/bridge-nf-call-iptables'
     end
     ```
   - Ensure Docker or another supported runtime is installed and configured.

### 3. **Initialize the Control Plane**
   - Add a recipe to initialize the control plane with `kubeadm`:
     ```ruby
     execute 'initialize control plane' do
       command 'kubeadm init --pod-network-cidr=10.0.0.0/16'
       not_if { ::File.exist?('/etc/kubernetes/admin.conf') }
     end

     execute 'set kubectl config' do
       command <<-EOH
         mkdir -p $HOME/.kube
         cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
         chown $(id -u):$(id -g) $HOME/.kube/config
       EOH
       only_if { ::File.exist?('/etc/kubernetes/admin.conf') }
     end
     ```

### 4. **Install Cilium**
   - Add a recipe to install Cilium with kube-proxy replacement:
     ```ruby
     execute 'install cilium CLI' do
       command 'curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/download/v0.15.8/cilium-linux-amd64 && chmod +x cilium-linux-amd64 && mv cilium-linux-amd64 /usr/local/bin/cilium'
       not_if 'command -v cilium'
     end

     execute 'deploy cilium' do
       command <<-EOH
         cilium install \
         --kube-proxy-replacement=strict \
         --cluster-name=k8s-cluster \
         --operator-replicas=1 \
         --version=v1.14.0
       EOH
       not_if 'kubectl get pods -n kube-system | grep cilium'
     end
     ```

### 5. **Join Worker Nodes**
   - Add a recipe for worker nodes to join the cluster:
     ```ruby
     execute 'join cluster' do
       command 'kubeadm join <control-plane-ip>:<port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>'
       not_if 'kubectl get nodes | grep $(hostname)'
     end
     ```

### 6. **Enable kube-proxy Replacement**
   - Add the following Cilium configuration in the recipe:
     ```ruby
     execute 'enable kube-proxy replacement' do
       command <<-EOH
         cilium config set kube-proxy-replacement strict
         cilium config set enable-bpf-masquerade true
         cilium config set enable-ipv4 true
         cilium config set enable-ipv6 false
       EOH
       only_if 'cilium status'
     end
     ```

---

## Common Errors and Troubleshooting

### 1. **Cilium Pods Failing**
- **Cause:** Kernel or BPF-related issues.
- **Fix:**
  - Check `kubectl logs -n kube-system -l k8s-app=cilium`.
  - Verify kernel supports eBPF with:
    ```bash
    bpftool feature
    ```

---

### 2. **Worker Node Fails to Join**
- **Cause:** Token expired or control plane is unreachable.
- **Fix:**
  - Regenerate the join token:
    ```bash
    kubeadm token create --print-join-command
    ```
  - Verify network connectivity to the control plane.

---

### 3. **Cluster Networking Issues**
- **Cause:** Misconfigured Cilium or missing kube-proxy replacement settings.
- **Fix:**
  - Check Cilium logs for errors:
    ```bash
    cilium status
    ```
  - Reapply Cilium with the correct settings.

---

### 4. **Cilium Fails to Replace kube-proxy**
- **Cause:** Node's kernel or environment does not support required eBPF features.
- **Fix:**
  - Verify eBPF is enabled:
    ```bash
    uname -r
    bpftool feature
    ```
  - Upgrade the kernel if required.

---

### 5. **Chef Run Fails**
- **Cause:** Errors in the cookbook or dependency issues.
- **Fix:**
  - Check Chef logs (`/var/chef/cache/chef-stacktrace.out`).
  - Ensure all dependencies (e.g., `curl`, `iptables`) are installed.

---

## Observations Over 5+ Years with Kubernetes and Chef

1. **eBPF with Cilium**: Using Cilium with kube-proxy replacement significantly improves performance but requires careful validation of kernel and BPF support on all nodes.
2. **Event Monitoring**: Always monitor `kubectl get events` and `cilium status` for real-time insights into cluster health.
3. **Chef Consistency**: Regularly update Chef cookbooks to align with the latest Kubernetes and Cilium versions to avoid using deprecated APIs or features.
4. **Networking Plugins**: Replacing kube-proxy reduces overhead but requires tuning and compatibility checks during upgrades.
5. **Logging**: Enable detailed logging for Cilium and API server for better debugging.

This approach ensures a streamlined Kubernetes cluster setup with improved network performance and observability using Cilium and Chef.
```
