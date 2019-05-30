---
title: 'WSL+K3d: Interop K3s is here!'
author: Nunix
date: 2019-05-30T13:40:15.581Z
description: Creating K3s cluster from WSL in a fast and reproductible way
image: /images/k3s-logo.png
tags:
  - wsl
  - k3d
  - docker
  - k3s
url: /wslk3d
categories:
  - Kubernetes
---
# Introduction

This year, I had the chance to participate to KubeCon europe and one of the (huge) takeaways was: [K3d](https://github.com/rancher/k3d)

While this could be "yet another" Kubernetes (K8s) cluster creator for **development** environments, it has 3 main advantages for my setup (remember, I live and breath in WSL):

1. K3d creates a K3s, and not K8s, cluster **inside** docker containers, making it "non-intrusive" and easier to use from a WSL distro that is already connected to the Windows docker daemon.
2. While it creates a one-node master, à la minikube, it actually allows us to directly add workers, and this is dope!
3. The options to create a cluster allow us to change ports both for the API and the cluster, meaning that with a basic script we could spun several clusters for different needs.

# Pre-requesites

Here is the list of the pre-requisites to make `k3d` run in WSL:

* [Install WSL on Windows 10](https://docs.microsoft.com/en-us/windows/wsl/install-win10)
* [Install Docker desktop for Windows](https://runnable.com/docker/install-docker-on-windows-10)
* [Install npiperelay.exe](https://github.com/jstarks/npiperelay)
* [Install kubectl on WSL](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

## \[Optional] Pengwin: the easy setup

If you own the [Pengwin distro](https://www.microsoft.com/en-us/p/pengwin/9nv1gv1pxz6p), you can install the "Docker bridge" as follow:

* Launch the `pengwin-setup` script
* Choose the `Tools` menu by pressing the space bar
  ![](/images/pengwin-setup-1-tools.png "Pengwin setup")
* Choose the `Docker` menu by pressing the space bar
  ![](/images/pengwin-setup-2-docker.png "Pengwin setup: docker")
* Once the install is completed, open a new instance
* Type `docker version`
  ![](/images/pengwin-setup-3-docker-version.png "Docker version")

# Install K3d

Now that the base is installed, the installation of `k3d` is actually very basic.

As described on the [github page](https://github.com/rancher/k3d), the easyiest way is to run the install script:

```
### Run the script with Wgetwget -q -O - https://raw.githubusercontent.com/rancher/k3d/master/install.sh | bash

### or with Curl
curl -s https://raw.githubusercontent.com/rancher/k3d/master/install.sh | bash
```

![](/images/k3d-install-wget.png "K3d install")

### **Important**:

when running scripts from Internet, please **always review** them first.

And concerning this script, as you may have noticed, do **not** run it as root, as it will only need root access in order to add it to the `/usr/local/bin` path.

## Have fun creating K3s clusters ...

Everything is now set, so it's time to test the environment:

```
### Check k3d version
k3d --version

### Create the first cluster > the name parameter is optional
k3d create --name wslk3s

### Set $KUBECONFIG to the config file created by k3d
export KUBECONFIG="$(k3d get-kubeconfig --name='wslk3d')"

### Check that the cluster is running
k3d list
kubectl cluster-info

### Additional checks
kubectl get pods --all-namespaces
kubectl get services --all-namespaces
```

![](/images/k3d-create-cluster.png "K3d cluster creation")

## ... and cleanup the resources

If creating a cluster is fast and quite easy, deleting it is even easier:

```
### Check the name of the active cluster
k3d list

### Delete the active cluster
k3d delete --name wslk3d

### Confirm that the cluster has been correctly deleted
k3d list
kubectl cluster-info
```

![](/images/k3d-delete-cluster.png "K3d cluster deletion")

# Conclusion

During the whole KubeCon I was creating and deleting clusters for the different workshops and/or demos.

And based on how I used it, this would be the "real" lesson learned and advice I can give: use `k3d` like you would use any container.

The goal is to rapidly reproduce an environment and therefore, delete it in order to cleanup the resources.

I hope you will have as much fun as I did and stay tuned for bonus sections.

> _**\>>> Nunix out <<<**_
