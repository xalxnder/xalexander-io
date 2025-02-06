---
draft: true
title: Installing a 3 Node Kubernetes Cluster
---

## Backstory 
Six months ago, I started a new job as a DevSecOps engineer. During the interview process the team gave me a heads up about what tools they used. I was excited to learn Kubernetes was one of them. I thought to myself, "Cool, I'll be ready. I've done a few tutorials on this." But after starting, I realized I was wrong. I was not ready 😂. Within the first two weeks, my head was spinning. I wasn’t just dealing with single container pods like i had before. No no, I was dealling with full-scale deployments, behind proxies, etc. Long gone were the days of tutorials. I needed to **start building**. I ordered the following beelink. This was going to be my sandbox. I hit the ground running. In this post I'll be walking through how to install a three node kuberentes cluster via Kubeadm.

There are numerous ways to get a kubernetes cluster up and running. Minikube, k3s, and talos to name a few. Why did I chose to go the kubeadm route? Because I'm a glutton for punishment. No, in all seriousness, I'm a all or nothing person. I'f I'm going to install kubernetes, I want to install it how teams in production environments are installing it.



#### Requirements
- 3 VMs 

I’m going to break the actual steps down into two parts. On the control plane node, on every Node

# On Every Node

### Disable Swap

> [!Note] Explanation
> The default behavior of a kubelet is to fail to start if swap memory is detected on a node. This means that swap should either be disabled or tolerated by kubelet.

In `/etc/fstab` comment out the line that declares swap.
```bash
#/dev/mapper/rl-swap     none                    swap    defaults        0 0
```
Confirm by running the `free` command. You should see 0s for the Swap row.
```bash
[xavier@lab-cp ~]$ free -m
               total        used        free      shared  buff/cache   available
Mem:            3562        1725         769          17        1316        1836
Swap:              0           0           0
```
---
### Install Containerd
> [!Note] Explanation
> To run containers in pods, we need a container runtime. I went with containerd

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

sudo dnf install containerd
```

Verify by checking its status
```bash
systemctl status containerd
```

### Install kubelet, kubeadm, and kubectl
```bash
# This overwrites any existing configuration in /etc/yum.repos.d/kubernetes.repo
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

### Configure cgroup Driver

> [!Note] Explanation
> The kubelet and the underlying container runtime need to interface with cgroups(coontrol groups) to enforce resource management for pods and containers which includes cpu/memory requests and limits for containerized workloads. The cgroup (control group) driver is responsible for managing and allocating system resources such as CPU and memory to containers. 
```bash
$ sudo mv /etc/containerd/config.toml /etc/containerd/config.toml.bak
$ containerd config default > config.toml
$ vim config.toml
```
In the config.toml file, set `SystemdCgroup` to true
![](https://miro.medium.com/v2/resize:fit:1400/0*9xY_Av2HZiL6FD_b)

### Add Kuberenetes Modules
`vim /etc/modules-load.d/k8s.conf`

![](https://miro.medium.com/v2/resize:fit:1400/0*P81ijuh8F3tNon9U)
```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

### Network Configuration
> [!Note] Explanation
> When setting up a Kubernetes cluster, we need to adjust certain kernel parameters to ensure that networking functions correctly. These settings control how the Linux kernel handles network traffic, particularly when dealing with bridged network interfaces and packet forwarding.

`vim /etc/sysctl.d/k8s.conf`

```bash
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```


# On The Control Plane 
### Initialize The Cluster
```bash
sudo kubeadm init --apiserver-advertise-address {HOST IP} --pod-network-cidr 10.244.0.0/16
```

If you're successful, you'll get something like :
```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join {HOST IP}:6443 --token {TOKEN} \
	--discovery-token-ca-cert-hash sha256:{HASH}
```

### Set Up KUBECONFIG
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Deploy A 
> [!Note] Explanation
> In order for your pods to communicate with each other, you need to install a Container Network Interface(CNI)