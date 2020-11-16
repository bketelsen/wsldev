---
title: "WSL2+KinD+Multipass: The Swarmenetes"
author: nunix
date: 2020-11-16T17:06:57.641Z
description: Can two Cloud Native technologies mix? Like Swarm and Kubernetes?
  Competitors you say? Allies they are!
tags:
  - wsl2
  - docker
  - swarm
  - kubernetes
  - kind
  - multipass
  - hyper-v
url: /swarmenetes
categories:
  - cloudnative
---
# Introduction

Local Kubernetes (K8s) single nodes cluster are so *before COVID* (read: 2019).
The current "trend" is to create K8s multi-nodes clusters or, even better, High Availability (HA) clusters.

When dealing with K8s in Docker (KinD), both multi-nodes and HA are possible, however it will run only on our computer/laptop. And this is totally fine, I mean, that's the original purpose "why" it has been created.

But what if a (crazy) Corsair wanted to create a multi-nodes cluster in a multi-hosts scenario? doing it **only** with the tools at hand and (literally) no YAML!!!

Starting to wonder how? is it even possible (specially the YAML part)?

The Corsair welcomes you to his new boat: the Swarmenetes.

# Prerequisites

This blog post is intended to be followed, locally, from any OS (reaching Cloud Native nirvana). Still, in order to give you a background, the following technologies are the ones being used in this blog post:

- OS: Windows 10 Professional version 2004 - channel: release (not Insiders, as incredible as it can be)
- WSL OS: [Ubuntu 20.04](https://www.microsoft.com/en-us/p/ubuntu-2004-lts/9n6svws3rx71) from the Windows Store
  - WSL version: 2
- WSL kernel: original version and updated via `wsl --update`
  - Version: 5.4.51-microsoft-standard-WSL2
- Virtualization: 
  - Local driver: Hyper-V
  - Multipass guest OS: Ubuntu 20.04 LTS
- Docker: [installed on WSL2 distro](https://docs.docker.com/engine/install/ubuntu/) 
  - Version: 19.03.13
  - User type: [anonymous](https://www.docker.com/increase-rate-limits)
- [Optional] Terminal: [Windows Terminal](https://www.microsoft.com/en-us/p/windows-terminal/9n0dx20hk701)
  - Version: 1.3.2651.0

## Docker Hub limits: how much will it cost?

TBD

# Building two nodes

Before we build our multi-nodes, multi-hosts KinD cluster, we need ... well ... at least two hosts.

For our first host, we will be using WSL2 with Ubuntu 20.04 as the distro. We will also install Docker, KinD and the Kubernetes tools directly on it.

We will **not** leverage Docker Desktop (see below: lessons learned).

The second host, as seen in the prerequisites, we will be using Canonical Multipass. This will allow us to run the exact same commands, independent of the OS running beneath. And same as for the first node, we will install all the components needed (Docker, KinD, Kubernetes tools)

## Preparing the Swarm leader

The first step is to prepare the main host where, once the cluster will be created, we will run the commands to manage the cluster from there.

If the prerequisites are installed (it won't  be explain in this blog), then we already have an OS updated and Docker installed:

![WSL2 Docker integrated](/images/wsl-docker-version.png)

### [Optional] Docker is not starting?

If the command `sudo service docker status` shows that Docker is not running, it might be due to an issue with iptables.

As described [here](https://github.com/WhitewaterFoundry/Pengwin/issues/485#issuecomment-518028465), we can setup our WSL2 distro to use the legacy iptables, and start again Docker:

```bash
# Source documentation: https://github.com/WhitewaterFoundry/Pengwin/issues/485#issuecomment-518028465
# Update the iptables command
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
# Start the docker daemon
sudo service docker start
# Check if the docker daemon is running
sudo service docker status
```

![WSL2 update iptables command](/images/wsl2-docker-iptables-update.png)

### Install KinD and Kubernetes tools

With Docker installed, we can now install KinD and the Kubernetes tools:

```bash
# Source documentation: https://kind.sigs.k8s.io/docs/user/quick-start/
# Current working directory: $HOME
# Download KinD binary
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.9.0/kind-linux-amd64
# Change the permissions of the binary to be executable
chmod +x ./kind
# Move the binary to a directory contained in the $PATH variable
sudo mv ./kind /usr/local/bin
# Check if KinD has been correctly installed
kind version
```

![WSL2 KinD install](/images/wsl-kind-install.png)

```bash
# Source documentation: https://kubernetes.io/docs/tasks/tools/install-kubectl/
# Download Kubectl binary
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
# Change the permissions of the binary to be executable
chmod +x ./kubectl
# Move the binary to a directory contained in the $PATH variable
sudo mv ./kubectl /usr/local/bin
# Check if kubectl has been correctly installed
kubectl version
```

![image-20201111085726859](/images/wsl-kubectl-install.png)

> *Note: the last command shows a connection refused. This is normal as there is still no Kubernetes server running. So we can consider it as a warning and not error*

 ```bash
# Source documentation: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
# Install (potential) dependencies packages
sudo apt update && sudo apt install -y apt-transport-https curl
 ```

![WSL2 kubeadm dependencies install](/images/wsl-kubeadm-dependencies.png)

```bash
# Add the repository signed key
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
# Add the repository
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

![WSL2 kubeadm repository add](/images/wsl-kubeadm-repository-add.png)

```bash
# Refresh the listing of the repositories
sudo apt update
```

![WSL2 kubeadm repository refresh](/images/wsl-kubeadm-repository-refresh.png)

```bash
# Install kubeadm
sudo apt install -y kubeadm
```

![WSL2 kubeadm install](/images/wsl-kubeadm-install.png)

```bash
# Check if kubeadm has been correctly installed
kubeadm version
```

![image-20201111152038754](/images/wsl-kubeadm-version.png)

Our first host is now ready, so time to prepare the second one.

## Preparing the Swarm co-leader

The second step will be to create and launch a new Virtual Machine (VM), and in order to ensure we are not limited, we will tweak the options:

```bash
# Create a new VM with the following options:
# Command: multipass launch
# Options:
# 	lts: shortname for the OS and its version, which is Ubuntu 20.04 LTS
# 	--name kind1: sets the name of the VM
# 	--cpus 2: number of virtual CPUs assigned to the VM
# 	--mem 4G: amount of RAM assigned to the VM
# 	--disk 20G: creates a virtual Disk with 20Go
multipass.exe launch lts --name kind1 --cpus 2 --mem 4G --disk 20G
# Check if the VM has been correctly created
multipass.exe list
```

![Multipass create new VM](/images/multipass-create-vm.png)

Once the VM is created, we can configure it by either running the commands with `multipass.exe exec` or entering the VM shell with `multipass.exe shell`.

For this blog post, we will enter into the shell as we will need to come back to it later on:

```bash
# Enter into the VM shell by providing the name of the VM
multipass.exe shell kind1
```

![Multipass enter into VM shell](/images/multipass-shell-vm.png)

We can now update and upgrade the system and install Docker, KinD and the Kubernetes tools:

```bash
# Current working directory: $HOME
# Update the repositories
sudo apt update
```

![Multipass guest OS update](/images/multipass-os-update.png)

![Multipass guest OS update part 2](/images/multipass-os-update-2.png)

```bash
# Upgrade the system with the latest packages version
sudo apt upgrade -y
```

![Multipass guest OS upgrade](/images/multipass-os-upgrade.png)

![Multipass guest OS part 2](/images/multipass-os-upgrade-2.png)

```bash
# Source documentation: https://docs.docker.com/engine/install/ubuntu/
# Install dependencies
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

![Multipass Docker dependencies install](/images/multipass-docker-dependencies-install.png)

```bash
# Add Docker repository GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
# Add Docker repository
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

![Multipass Docker repository install](/images/multipass-docker-repository-install.png)

```bash
# Install Docker and all needed components
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

![Multipass Docker install](/images/multipass-docker-install.png)

> *Note: as displayed in the screenshot, Docker was already installed in order to show the results more succintly*

```bash
# Add the user to the docker group
sudo usermod -aG docker $USER
# Activate the change to the Docker group without logout/login
newgrp docker
```

![Multipass Docker add user to group](/images/multipass-docker-group-add-user.png)

```bash
# Check if Docker is running
docker version
```

![Multipass Docker status](/images/multipass-docker-status.png)

```bash
# Source documentation: https://kind.sigs.k8s.io/docs/user/quick-start/
# Download KinD binary
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.9.0/kind-linux-amd64
# Change the permissions of the binary to be executable
chmod +x ./kind
# Move the binary to a directory contained in the $PATH variable
sudo mv ./kind /usr/local/bin
# Check if KinD has been correctly installed
kind version
```

![Multipass Kind install](/images/multipass-kind-install.png)

```bash
# Install the Kubernetes tools from the SNAP repository
sudo snap install kubectl --classic
sudo snap install kubeadm --classic
# Check if the Kubernetes tools have been correctly installed
kubectl version
kubeadm version
```

![Multipass Kubernetes tools install](/images/multipass-k8s-tools-install.png)



At this point, we have everything ready on the *second host*, and before you create our cluster, we need to setup the network in a "very special" way.

# WSL finally meets Hyper-V

Since the launch of WSL2, there was a "small but painful" limitation: it could not see or connect to the Hyper-V networks. This means that the WSL2 distros and the Hyper-V VMs could simply not communicate.

In October this year (2020), [Chris Jeffrey](https://twitter.com/chrisjeffreyuk) suggested a [very nice workaround](https://techcommunity.microsoft.com/t5/itops-talk-blog/windows-subsystem-for-linux-2-addressing-traffic-routing-issues/ba-p/1764074) and that's what we will use in this blog.

But first let's see how the connection is not possible between WSL2 and Hyper-V VMs:

```bash
# Find the Multipass VM ip
# Command: multipass info
# Options:
# 	kind1: name of the VM
multipass.exe info kind1
```

![Multipass VM info](/images/multipass-vm-info.png)

```bash
# Try to ping the Multipass VM
ping 172.22.244.80
# Try to ssh into Multipass VM
ssh -l ubuntu 172.22.244.80
```

![WSL2 ping Hyper-V](/images/wsl2-hyperv-ping.png)

As we can see, no connectivity is currently possible between WSL2 and Hyper-V VMs.

Let us run the command from Chris blog and try again the connectivity tests:

```bash
# Source documentation: https://techcommunity.microsoft.com/t5/itops-talk-blog/windows-subsystem-for-linux-2-addressing-traffic-routing-issues/ba-p/1764074
# Run the powershell command directly from WSL
powershell.exe -c 'Get-NetIPInterface | where {$_.InterfaceAlias -eq "vEthernet (WSL)" -or $_.InterfaceAlias -eq "vEthernet (Default Switch)" } | Set-NetIPInterface -Forwarding Enabled'
# [Optional] if we rebooted Windows 10, we need to find again the Multipass VM IP
multipass.exe info kind1
```

![WSL2 ping Hyper-VM VM successfully](/images/wsl2-hyperv-network-forward.png)

```bash
# Try to ping the Multipass VM
ping 172.29.221.245
# Try to ssh into Multipass VM
ssh -l ubuntu 172.29.221.245
```

![image-20201115134453998](/images/wsl2-hyperv-ping-success.png)

> *Note: the SSH command ends in error as the Multipass VM SSH server is configured to only accept connections with a SSH key*

Success! our WSL2 distro can now connect to the Hyper-V VMs which use the "Default Switch" for their Network virtual adapter.

Now it's time to setup the network that will be used for our KinD multi-nodes cluster.

# A Swarm-y network

Right now, even if WSL2 sees the Hyper-V VMs, both of them have their own Docker network setup, which are private to the host. So what we need, is a common network which our containers will use in order to be able to communicate "easily" between them.

If only we had something to help us making it simple, right? Well we do have this technology at hand since quite a long time (Cloud Native years): [Docker Swarm overlay network](https://docs.docker.com/network/overlay/)!

## The Swarm creation

With the connectivity enabled between WSL2 and Hyper-V, we can create a Docker Swarm cluster between the two nodes.

In order to use the **exact same** network, we need our two nodes to be `manager nodes`. We will also need to ensure we use the IP assigned to the WSL2 host `eth0` interface, as it is the one that Hyper-V sees:

```bash
# Get the eth0 IP address
ip addr show eth0 | grep -m1 inet
# Initialize the swarm on the main node with the eth0 IP address
docker swarm init --advertise-addr=172.28.115.62
```

![WSL2 Docker Swarm init](/images/wsl2-docker-swarm-init.png)

```bash
# Run the join command on the Multipass VM
docker swarm join --token SWMTKN-1-60ozt3r11ea82y1a2z4hvoxdkxjmjax0u91eirbmanb9aypy91-543i683o73p4t76z7jdv5gzs5 172.28.115.143:2377
```

![Multipass VM docker join](/images/wsl2-docker-swarm-join.png)

Both nodes are now part of the Swarm cluster and we can now promote the second node and check their status from the primary `manager node` :

```bash
# Check the current status for both nodes
docker node ls
# Promote the second node as manager
docker node promote kind1
# Check if the status has been correctly updated
docker node ls
```

![WSL2 Swarm nodes list](/images/wsl2-docker-swarm-node-list.png)

The two nodes Swarm cluster is now created. We can create our network.

## A KinD and attachable network

Now that we have our nodes connected, we still need to use a common network. By default, Docker Swarm creates two networks as described in the [official documentation](https://docs.docker.com/network/overlay/) and cited here:

- an overlay network called `ingress`, which handles control and data traffic related to swarm services. When you create a swarm service and do not connect it to a user-defined overlay network, it connects to the `ingress` network by default.
- a bridge network called `docker_gwbridge`, which connects the individual Docker daemon to the other daemons participating in the swarm.

However, both networks are exclusively used for deploying Docker Swarm services or stacks. In our case, we need to attach "standalone containers" to a Swarm network.

Thankfully, we can create our own network and make it attachable by the containers directly:

```bash
# Source documentation: https://docs.docker.com/network/network-tutorial-overlay/#use-an-overlay-network-for-standalone-containers
# Create a new overlay network
# Command: docker network create
# Options:
# 	--driver=overlay: specifies the network driver to be used 
#   --attachable: allows containers to use this network
#   kindnet: name of the network
docker network create --driver=overlay --attachable kindnet
# Check if the network has been created
docker network ls
```

![WSL2 Swarm network create](/images/wsl2-docker-swarm-network-create.png)

```bash
# Check on the second manager node is the network is visible
docker network ls
```

![image-20201115212419227](/images/multipass-docker-swarm-network-check.png)

We have now configured all the components and can create our multi-nodes KinD cluster.

# Swarmenetes: full sail!

The first step will be to create our two KinD clusters and then we will *reset* the one on the second `manager node` and will have it join the KinD cluster on the primary `manager node`.

Once again, this section has been made possible thanks to a great human being: the KinD Super Hero and CNCF Ambassador [Duffie Cooley](https://twitter.com/mauilion). He took time over his schedule to answer silly Corsair's questions and help unlocking the Pandora box.

## A single node to start slowly

Before we create the multi-nodes cluster, let us understand how KinD works and what could be an issue when going multi-nodes. And we will see what workaround/solution will be used to avoid the said issues.

OK, let us decrypt this enigmatic sentence by creating a single node KinD cluster:

```bash
# Create a single node KinD cluster
kind create cluster
```

![WSL2 KinD create cluster](/images/wsl2-kind-cluster-create.png)

```bash
# Check if the cluster is running
kubectl cluster-info --context kind-kind
# Check the node from KinD perspective
kind get nodes
# Check the node from Kubernetes perspective with a detailed output
kubectl get nodes -o wide
# Check which server address is used in the ".kube/config" file
grep server .kube/config
```

![image-20201115175211288](/images/wsl2-kind-cluster-check.png)

Everything is working well, however the `config` file shows that our server connection string is using the local IP (127.0.0.1).

And if we try to generate a `join token`, this will also be the IP used:

```bash
# Create a new join token
kubeadm token create --print-join-command
```

![KinD create join token](/images/wsl2-kind-cluster-join-token.png)

And if we try to run this command in our `worker node` then it will fail as it will try to connect locally.

Well, this actually not the full history. Let us have a deeper look into the container running the K8s cluster.

### KinDly show the config

When we create a K8s cluster with KinD, it generates an "external" config to be used outside the container. If we follow the rabbit (read: the `server connection string` in the `.kube/config` file), we will see the local IP with a random port. This port is actually bound to the standard [K8s defaut API server port 6443](https://kubernetes.io/docs/concepts/security/controlling-access/#api-server-ports-and-ips):

```bash
# Check the running container
docker ps
```

![Docker KinD control plane port bindings](/images/wsl2-kind-cluster-port-check.png)

This configuration, as said before, will not help us, however if we check the "internal" configuration, then we get way more interesting data:

```bash
# Get the server connection string from the "admin.conf" inside the container
docker exec swarmenetes-control-plane grep server /etc/kubernetes/admin.conf
```

![Docker KinD check server connection string](/images/wsl2-kind-cluster-config-internal.png)

Perfect! the `server connection string` inside the container is set against a DNS name and not an IP. So if we go to the very end of this rabbit hole, we should get a `join` command that we will be able to reuse in another node:

```bash
# Check if kubeadm is pre-installed in the container
docker exec swarmenetes-control-plane kubeadm version
# Create a new join token from inside the container
docker exec swarmenetes-control-plane kubeadm token create --print-join-command
```

![image-20201116152850007](/images/wsl2-kind-cluster-join-token-internal.png)

As expected, the `join` command has the DNS name seen in the `server connection string` and we will be able to reuse it.

It's time to put everything together and build our multi-nodes on multi-hosts KinD cluster.

### Cleaning before the grand finale

Let us delete this cluster as it is not using the right network:

```bash
# Check the name of the cluster created
kind get clusters
# Delete the running cluster
kind delete cluster
# Check if the cluster is deleted
kind get clusters
```

![KinD cluster delete](/images/wsl2-kind-cluster-delete.png)

## A multi-nodes on multi-hosts

Finally here and we will directly create a two nodes cluster on our `manager nodes`. And let us have a look on the last remaining part that will help us: attaching to a specific network when creating a KinD cluster.

### KinD network: dancing with Dragons

Before we can create our new cluster, we need to see how we can use our Swarm network we created earlier.

Remember, this network will allow us to create the two clusters on different hosts and still using a unique network to communicate.

One more time, luckily for us, the KinD team already got a [similar request](https://github.com/kubernetes-sigs/kind/issues/273) and [provided](https://github.com/kubernetes-sigs/kind/pull/1538) an [experimental feature](https://github.com/kubernetes-sigs/kind/pull/1538/files) (cf. pkg/cluster/internal/providers/docker/provider.go):

```bash
# Check the Docker networks
docker network ls
# Set the KinD network environment variable to the network name to be used
export KIND_EXPERIMENTAL_DOCKER_NETWORK="kindnet"
```

![KinD network variable set](/images/wsl2-kind-cluster-network-variable.png)

In order to save us time, we already know that we will need to run the exact same commands in our second node, so let us do it now:

```bash
# Check the Docker networks
docker network ls
# Set the KinD network environment variable to the network name to be used
export KIND_EXPERIMENTAL_DOCKER_NETWORK="kindnet"
```

![image-20201116154325547](/images/multipass-kind-cluster-network-variable.png)

### Creating two KinD clusters

We can finally create our KinD clusters using our Swarm network. And for an easier management "later on", we will also set a specific name to each one:

```bash
# Create a KinD cluster on the first node
kind create cluster --name swarmenetes
```

![KinD cluster multi-nodes create on first node](/images/wsl2-kind-cluster-create-multinode.png)

> *Note: as we can see, there is two warnings about using a specific Docker network*

```bash
# Create a KinD cluster on the first node
kind create cluster --name swarmenetes-mate
```

![KinD cluster multi-nodes create on second node](/images/multipass-kind-cluster-create-multinode.png)

```bash
# Check if the cluster is running on the first node
kubectl cluster-info --context kind-swarmenetes
# Check the node from KinD perspective
kind get nodes --name swarmenetes
# Check the node from Kubernetes perspective with a detailed output
kubectl get nodes -o wide
```

![KinD cluster multi-nodes check](/images/wsl2-kind-cluster-multinode-check.png)

```bash
# Check if the cluster is running on the second node
kubectl cluster-info --context kind-swarmenetes-mate
# Check the node from KinD perspective
kind get nodes --name swarmenetes-mate
# Check the node from Kubernetes perspective with a detailed output
kubectl get nodes -o wide
```

![image-20201116160213846](/images/multipass-kind-cluster-multinode-check.png)

Our two clusters are up and running and if we look closely at the *INTERNAL-IP* values, we can see they are both on the same network subnet.

Let us do a quick check to see if it is the same network:

```bash
# Use curl to see if we get a response from the other node
curl -k https://10.0.1.4:6443
```

![Docker curl second node](/images/wsl2-kind-cluster-network-curl.png)

With the confirmation both clusters can communicate, we can reset our second node and join it to our cluster on the primary node.

### What is the address name?

While our clusters can communicate using their respective IP addresses, they do not have "name resolving" to these specific addresses.

And as we saw in the `join`  command that we generated "internally", there was no IP address, just a "DNS name".  This means we will need to had this particular name to be resolved by our second node:

```bash
# Check the name of the container running on the primary node
docker ps
# Get the server connection string from the "admin.conf" inside the container of the primary node
docker exec swarmenetes-control-plane grep server /etc/kubernetes/admin.conf
# Get the primary node IP from the Kubernetes detailed output
kubectl get nodes -o wide
```

![Docker KinD get DNS name and IP](/images/wsl2-kind-cluster-hostname-ip.png)

```bash
# Check the name of the container running on the second node
docker ps
# Add the DNS name and IP address to the /etc/hosts of the second node
docker exec swarmenetes-mate-control-plane bash -c "echo '10.0.1.2        swarmenetes-control-plane' >> /etc/hosts"
```

![image-20201116165954554](/images/multipass-kind-cluster-multinode-hostname-add.png)



### Reset before joining

Before we can run the `join` command against our cluster on the second node, we need to, literally, reset its configuration.

And to ensure the reset is "clean", we will do it from within the container:

```bash
# Check the name of the container running on the second node
docker ps
# Reset the configuration of the cluster on the second node
docker exec swarmenetes-mate-control-plane kubeadm reset -f
```

![image-20201116162950619](/images/multipass-kind-cluster-reset.png)

Now that our cluster is reset, we can get the `join` command from the cluster on the primary node and create our multi-nodes cluster.

### And in the Swarm bind them

As seen during our tests on the single node cluster, we will need to run the `join` command within the containers hosting both `manager nodes`.

On our primary node, we can generate a new `join` command to be used on our second node cluster:

```bash
# Check the name of the container running on the primary node
docker ps
# Create a new join token from inside the container
docker exec swarmenetes-control-plane kubeadm token create --print-join-command
```

![Docker KinD generate join command](/images/wsl2-kind-cluster-join-token-internal-multinode.png)

Then in our second node, we can run the `join` command:

```bash
# Check the name of the container running on the second node
docker ps
# Run the join command
docker exec swarmenetes-mate-control-plane kubeadm join swarmenetes-control-plane:6443 --token w7yfju.klu2fedmcdi6yiqz     --dis
covery-token-ca-cert-hash sha256:6f7b60592aab7e89e3383632cf95364e09b2656721cc77f64d6c8c6420b5f84f --ignore-preflight-errors=all
```

![KinD second node join](/images/multipass-kind-cluster-join-multinode.png)

![KinD second node join](/images/multipass-kind-cluster-join-multinode-2.png)

> *Note: as we can see in the first screenshot, the option "--ignore-preflight-errors=all" was added, as we are running K8s in a container and if we omit it, then the join command will fail*

At the end of the command output, it suggests to run the `kubectl get nodes` on the `control-plane`, which is now our cluster in the primary node:

```bash
# Check the name of the cluster created
kind get clusters
# Check the node from KinD perspective
kind get nodes --name swarmenetes
# Check the node from Kubernetes perspective with a detailed output
kubectl get nodes -o wide
```

![image-20201116172530878](/images/wsl2-kind-cluster-multinode-check-complete.png)

Let us also have a view at the pods created and their "placement":

```bash
# Check the pods created
kubectl get pods --all-namespaces -o wide
```

![image-20201116173400930](/images/wsl2-kind-cluster-pods-check.png)

And that concludes this (very?) lengthy but oh so fun blog.

# Conclusion

When blogging about WSL and/or simply being part of this incredible community, we quickly learn we are now living in a world where "AND" is more powerful than "OR".

The Cloud Native world is no different and I hope this blog post can show that when we mix different technologies, maybe, something totally cool (and potentially useless, I must admit) can be created.

And even if this blog is really not meant to be run elsewhere than on our own computer(s), the learning it provided for the different layers is the big gain, at least for me. And I really hope you will gain something from it too.

 

> ***\>>> Nunix out <<<***