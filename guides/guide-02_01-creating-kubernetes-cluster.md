# 02-01: Creating a Kubernetes Cluster on Proxmox 

## Prerequisites

- Proxmox.
- Ubuntu Cloud-Init Template

## Introduction

We'll be creating Kubernetes cluster bootstrapped with `kubeadm` with Flannel as the overlay network.
You can find the ADRs about [kubeadm](../adrs/adr-001-kubeadm.md) for reasoning on why we are using it.

## Step 1: Create VMs from Ubuntu Cloud-Init Template

I will be creating a 2 node Kubernetes cluster with one control-plane node and one worker node.
So I'll be cloning the template twice.

Right-click the template, and select "Clone".

```
Control plane node 
- VM ID : 150
- name : k8s-master-1

Worker node
- VM ID : 151
- name : k8s-worker-1
```

> Full Clone
> A full clone VM is a complete copy and is fully independent from the original VM or VM Template, but it requires the same disk space as the original.

- Optionally set `Start at boot` option to yes for both VMs if you want them to boot whenever proxmox starts up, also consider adding a delay to startup of the worker node, it's not required, but I'll be doing it, just to make sure the control-plane is up before the worker node

- Setup DHCP reservation for instances on your router, so their IP addresses cannot change

### Verify the MAC address and product_uuid are unique for every node

- You can get the MAC address of the network interfaces using the command `ip link` or `ifconfig -a`
- The product_uuid can be checked by using the command `sudo cat /sys/class/dmi/id/product_uuid`


## Step 2: Installing containerd

This Kubernetes cluster will utilize the containerd runtime. To set that up, we’ll first need to install the required package:

```bash
sudo apt install containerd
```

### Create the initial configuration:

```bash
sudo mkdir /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

### Configuring the systemd cgroup driver

To use the `systemd` cgroup driver in `/etc/containerd/config.toml` with `runc`, set the following config based on your Containerd version

Containerd versions 2.x:
```
[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc]
  ...
  [plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]
    SystemdCgroup = true
```

The `systemd` cgroup driver is recommended if you use `cgroup` v2.

If you apply this change, make sure to restart containerd:

```bash
sudo systemctl restart containerd
```

## Step 3: Disable swap

Kubernetes requires swap to be disabled primarily to guarantee predictable performance and resource isolation. By default, the kubelet agent will fail to start if it detects that swap is enabled on a host system.

To turn off swap, we can run the following command:

```bash
sudo swapoff -a
```

Next, edit the `/etc/fstab` file and comment out the line that corresponds to swap (if present). This will help ensure swap doesn’t end up getting turned back on after reboot.

## Step 4: Enable bridging and br_netfilter

This is required so kube-proxy and our networking plugins (CNIs) can properly route, filter, and modify container traffic traversing between different pods or the host.

### Enable Modules Instantly

Load the br_netfilter module into the active kernel right now:

```bash
sudo modprobe br_netfilter
```

### Force Load on Ubuntu Boot

Ubuntu reads files in `/etc/modules-load.d/` to load drivers during startup. Save the module configuration there:

```bash
echo "br_netfilter" | sudo tee /etc/modules-load.d/k8s.conf
```

### Apply Sysctl Settings for iptables

Ubuntu relies on sysctl to pass bridge traffic through your firewall rules. Apply the network bridge and IPv4 forwarding settings:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

### Reload System Parameters

Apply the configurations without restarting your Ubuntu server:

```bash
sudo sysctl --system
```

### Verify the Status

Ensure Ubuntu successfully activated the rules. Run this command:

```bash
sysctl net.bridge.bridge-nf-call-iptables net.ipv4.ip_forward
```

Expected Output:
```text
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```

## Step 5: Reboot your servers

Reboot each of your instances to ensure all of our changes so far are in place:

```bash
sudo reboot
```

## Step 6: Installing kubeadm, kubelet and kubectl

You will install these packages on all of your machines:

- `kubeadm`: the command to bootstrap the cluster.
- `kubelet`: the component that runs on all of the machines in your cluster and does things like starting pods and containers.
- `kubectl`: the command line util to talk to your cluster.

These instructions are for Kubernetes v1.36.

- Update the `apt` package index and install packages needed to use the Kubernetes `apt` repository:

```bash
sudo apt-get update

sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

- Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:

> If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command
> `sudo mkdir -p -m 755 /etc/apt/keyrings`

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

- Add the appropriate Kubernetes apt repository.

```bash
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

- Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version:

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

- (Optional) Enable the kubelet service before running kubeadm:

```bash
sudo systemctl enable --now kubelet
```

## Step 7: Control-Plane Node only: 

### Initialize the Kubernetes cluster

We can initialize the Kubernetes cluster now. Be sure to customize the first IP address `control-plane-endpoint` to the IP of the control plane node (Don't change the second IP, that's for the pod subnet) and also change the name to match the name of your control-plane node.

```bash
sudo kubeadm init --control-plane-endpoint=192.168.0.150 --node-name k8s-master-1 --pod-network-cidr=10.244.0.0/16
```

After the initialization finishes, you should see at least four commands printed within the output.

Copy the join commands for the worker node somewhere safe, we'll use it to join the workers to the cluster later.

### Setup our user account to manage the cluster

Three commands will be shown in the output from the previous command, and these commands will give our user account access to manage our cluster, without needing to use the root account to do so.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Install an Overlay Network

If you check right now the coredns pods would be stuck in pending state

```bash
kubectl get pods -A
```

They need an overlay network to figure out the IPs of the pods and function properly. They will work as expected after the flannel overlay network is installed. 

The following command will install the Flannel overlay network 

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Platforms like Kubernetes assume that each container (pod) has a unique, routable IP inside the cluster. The advantage of this model is that it removes the port mapping complexities that come from sharing a single host IP.

Flannel is responsible for providing a layer 3 IPv4 network between multiple nodes in a cluster. Flannel does not control how containers are networked to the host, only how the traffic is transported between hosts.

### Adding Nodes

The join command, which we received from the output once we initialized the cluster, can be ran on your node instances now to get them joined to the cluster. 

The following command will help you monitor which nodes have been added to the controller (it can take several minutes for them to appear):

```bash
kubectl get nodes
```

If for some reason the join command has expired, the following command will provide you with a new one:

```bash
kubeadm token create --print-join-command
```

## Step 8: Join Worker Nodes to the cluster

Run the join command from the previous step on each worker node as root.
Since I only have one worker I'll have to run the command only on there.

```bash
sudo kubeadm join 192.168.0.150:6443 --token <your-token> --discovery-token-ca-cert-hash <sha256-cert-hash>
```

## Step 9: Deploying a container within our cluster

Create the file `pod.yml` with the following contents:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx-container
    image: nginx:latest
    ports:
    - containerPort: 80
      name: "nginx-http"
```

Apply the file with the following command:

```bash
kubectl apply -f pod.yml
```

You can check the status of this deployment with the following command:

```bash
kubectl get pods
```

## Step 10: Creating a NodePort Service

Setting up a NodePort service is one of the methods we can use to be able to access the container from outside the pod network. 
To set it up, first create the following file as `svc.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      nodePort: 30080
      targetPort: nginx-http
  selector:
    app: nginx
```

You can apply that file with the following command:

```bash
kubectl apply -f svc.yml
```

To check the status of the service deployment, you can use the following command:

```bash
kubectl get service
```

You can now access the service from your local network using the IP of either the control-plane or the worker node with the post number (e.g. http://192.168.0.151:30080)

Yay!! We have a working kubernetes cluster now :)

I'll be doing a lot a things in this cluster, so keep an eye out for all the updates, for now I have plans to automate this entire process of creating the cluster with Terraform and Ansible, I'll also be replacing kube-proxy and the flannel overlay network with Cilium CNI later and install bunch of infra components on the cluster.

## References

- [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd)
- [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Build an Awesome Kubernetes Cluster using Proxmox](https://www.learnlinux.tv/how-to-build-an-awesome-kubernetes-cluster-using-proxmox-virtual-environment/)
- [Flannel Docs](https://github.com/flannel-io/flannel#deploying-flannel-manually)