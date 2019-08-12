---
title: 'WSL+Docker: K8s joins the party'
author: Nunix
date: 2019-08-08T15:41:56.304Z
description: WSL2 + Docker + K8s = FUN!!!
image: /images/docker-context-logo.png
tags:
  - wsl2
  - docker
  - kubernetes
  - k8s
  - k3d
  - k3s
url: /wsl2dockerk8s
categories:
  - Docker
---
# WSL2+Docker: K8s joins the party

## Introduction

Docker and Microsoft have announced the release of [Docker Desktop for Windows with WSL2 support](https://blog.docker.com/2019/07/5-things-docker-desktop-wsl2-tech-preview/).

What it really means, is that an `Ubuntu 18.04` distro can be the endpoint for Docker (more distros might follow).

I do recommend (strongly), to have a look on the following blog posts from:

* [Thomas Maurer](https://www.thomasmaurer.ch/2019/08/run-linux-containers-with-docker-desktop-and-wsl-2/)
* [Scott Hanselman](https://www.hanselman.com/blog/DockerDesktopForWSL2IntegratesWindows10AndLinuxEvenCloser.aspx)

And now that we are all set, this blog post will explain how to had a Kubernetes endpoint to the `wsl context` and, thanks to the always awesome `interop` functionalities, make it accessible from everywhere.

## Pre-requisites

As usual, I do not take anything for granted, so here are the tools that we will be using:

1. Windows 10 Insider version 18945 minimum (currently on fast ring only)
2. [Docker for Windows WSL2 tech preview](https://docs.docker.com/docker-for-windows/wsl-tech-preview/)
3. [K3d](https://github.com/rancher/k3d) which is a tool for creating [K3s](https://github.com/rancher/k3s) cluster on Docker -> should be installed on WSL2
4. [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) -> should be installed on Windows at least and optionally on WSL2

> **IMPORTANT**: please follow the blog posts and the links above for installing the different components. The installation of those components will not be covered in this blog post. Makes it shorter to read.

## Setup: it's all about contexts

Right after the Docker installation is done and the `WSL 2 Tech Preview` is activated (through the menu), we have the following contexts:

![](/images/docker-install-contexts.png)

> TIP: if the `wsl` context is not the default (marked by an `*` near the name), run the following command:

```powershell
PS> docker context use wsl
```

![](/images/docker-context-switch.png)

As we can see, the context **do not** have any `kubernetes endpoint` or even an `orchestrator` set.

So let's add one with our _beloved_ K3d.

## Wait, why not using the default Docker Kubernetes?
And that's a very good question. The issue here is how Kubernetes is setup within Docker Desktop.

First we will need to add the same `kubernetes endpoint` than the `default` context.
To do that, we will use the config file:

```powershell
PS> docker context update --kubernetes config-file=$HOME\.kube\config --default-stack-orchestrator kubernetes wsl
```

![](/images/docker-context-k8s-wsl.png)

This actually would work great from Windows Powershell as `kubectl` would also leverage the config from `$HOME\.kube\config`:

![](/images/docker-context-kubectl-info.png)

However, from `wsl`, even if we set the variable `KUBECONFIG` to the docker kubernetes config file, we will face the following issue:

```bash
PS> wsl
$ export KUBECONFIG=/mnt/c/Users/<your username>/.kube/config
$ kubectl cluster-info
```

![](/images/docker-context-k8s-wsl-error.png)

I won't go to much in-depth here, specially because I still have to find a solution.
But if someone knows how to access it, then please feel free to share.

## K3d to save the day
Back to our topic on how to add a `kubernetes endpoint` from WSL?

First, a new `docker context` as the `default` context cannot be modified:

```bash
$ docker context list
$ docker context create --docker from=default wsl
$ docker context use wsl
$ docker context list
```

![](/images/docker-context-k8s-wsl-create.png)

Then, a new `k3d` cluster will be created:

```bash
$ k3d create --api-port $(hostname -I | awk '{ print $1 }'):16443
$ export KUBECONFIG="$(k3d get-kubeconfig --name='k3s-default')"
$ kubectl cluster-info
```

![](/images/docker-context-k8s-wsl-k3d-create.png)

Finally we can add the `kubernetes endpoint` do the context created:

```bash
$ docker context update --kubernetes config-file=$KUBECONFIG --default-stack-orchestrator=kubernetes wsl
```

![](/images/docker-context-k8s-wsl-update.png)

> **NOTE:** as you can see in printscreen, it seems to have a "visual bug" with the default context. Upon restart of the daemon, the context will be correct again

## WSLENV makes it consistent
While we have now both `wsl` contexts configured, there are literally not the same.

And talking about contexts, we have also the `kubectl` contexts that are misaligned:

![](/images/docker-context-comparison.png)

While we will not be able to use the Windows context (for now), the other way is actually possible.

So let us "pass" the `KUBECONFIG` variable from `wsl` to `powershell` and add it to the `wsl` context on Windows side:

```bash
$ export KUBECONFIGWSL=$KUBECONFIG
$ export WSLENV=KUBECONFIGWSL/p
$ powershell.exe
PS> echo $env:KUBECONFIGWSL
PS> docker context update --kubernetes config-file=$env:KUBECONFIGWSL --default-stack-orchestrator=kubernetes wsl
PS> docker context ls
```

![](/images/docker-context-wslenv.png)

And to ensure everything works, let's also update the `KUBECONFIG` variable from Windows:

```powershell
PS> kubectl.exe config get-contexts
PS> $env:KUBECONFIG = "$HOME\.kube\config;$env:KUBECONFIGWSL"
PS> kubectl.exe config get-contexts
PS> kubectl.exe cluster-info
PS> kubectl.exe config use-context default
PS> kubectl.exe config get-contexts
PS> kubectl.exe cluster-info
```

![](/images/docker-context-wslenv-switch.png)

> **NOTE:** ensure to separate the paths by a semicolon "**;**" as we are in Windows, and not the usual colon "*:*"

## Conclusion
All the technologies seem to work very well together and while I faced quite some (tough) challenges, the overall setup feels quite nice.

Still, I really think that the moment we will be able to use the `Docker Desktop Kubernetes` endpoint from the Docker context inside `wsl`, then I think this setup will feel even more "light".

So do not hesitate to test this setup and find even better ways. And share, I'm as usual roaming in Twitter [@nunixtech](https://twitter.com/nunixtech)


> \>\>\> *Nunix out* <<<
