---
title: "Win10+Docker: Hybrid swarm on the Desktop"
author: Nunix
date: 2020-05-01T19:25:26.286Z
description: Windows 10 and Docker Desktop unlimited
tags:
  - windows 10
  - docker
  - docker desktop
  - docker swarm
  - wsl2
  - hyper-v
  - vagrant
  - multipass
  - portainer
url: /hybridswarm
categories:
  - Docker
---
*Win10+Docker: Hybrid swarm on the Desktop*

# Introduction

Windows 10 2004 brings possibilities ... lots of possibilities.

Which means that if you're like me, and like to mix things that originally shouldn't be, then you will be in Nirvana.

This blog is very much pushing some boundaries, mixing with some small "hacks", but I guarantee you it's 100% fun.

So if you're not afraid of testing "impossible" scenarios, jump along and, of course, **DO NOT** try this at Work. Stay at Home, you should know it by now (ok, bad confinement pun)

# Sources

As I often stated: all ideas have an inspiration or are an alternative to something already done/existing.

For this blog post, my main inspirations were:

* Youtube: 3 videos by [Kallie \[MSFT\]](https://www.youtube.com/channel/UCbSVoaqgE3XNPfiVuBN_LRQ/videos)
* Blog: [Hybrid docker swarm on Google Cloud](https://collabnix.com/building-hybrid-docker-swarm-mode-cluster-on-google-cloud-platform/) by [Captain Ajeet Singh Raina](https://twitter.com/ajeetsraina)
* Gist: [Hybrid (Windows+Linux) Docker Swarm](https://gist.github.com/roommen/1cb4e04ab42721b9c61f9f154196744b) by [Runcy Oommen](https://twitter.com/runcyoommen)

Other references will be properly linked in the blog post.

So to everyone who already took the time to create content: THANK YOU!

# Prerequisites

The following list might be quite long, so I won't explain how to install it. However I will put the links that I used myself for this particular setup.

And, for the first time, please try this setup if you have 16Go RAM or more (32Go to be on the safe side). There's quite a lot of virtualization overhead, so if you have only 8Go RAM, it might short.

Here is the shopping list:

* OS: Windows 10 with the latest update (2004) or if you reading it now, the Insider Release Preview is the minimum
* Win10 Features: 

  * [Hyper-v](https://techcommunity.microsoft.com/t5/itops-talk-blog/step-by-step-enabling-hyper-v-for-use-on-windows-10/ba-p/267945)
  * [Virtualization Platform and Windows Subsystem for Linux (v2) enabled](https://www.thomasmaurer.ch/2019/06/install-wsl-2-on-windows-10/)
* WSL2: [a Store or Custom distro](https://wiki.ubuntu.com/WSL)
* [Multipass](https://multipass.run/)
* [Vagrant](https://www.vagrantup.com/intro/getting-started/install.html)
* [Docker Desktop for Windows](https://docs.docker.com/docker-for-windows/install/)
* \[Optional] [Windows Terminal](https://devblogs.microsoft.com/commandline/windows-terminal-preview-v0-10-release/)
* \[Optional] [Visual Studio Code](https://code.visualstudio.com/)

Ok, once you have installed all the pieces above, let's setup them all and create "the Great Hybrid Swarm".

# Docker Desktop: 2 daemons running wild

For the first setup, well we will already be applying our first "hack": running **both** the Windows and Linux daemon.

In the past, I blogged about it twice ([here](http://darthnunix.blogspot.com/2016/10/docker-for-windows-2-daemons-enter-in.html) and [there](http://wslcorsair.blogspot.com/2018/04/docker-wsl-get-2-daemon-for-price-of-1.html)) and actually this very cool hack is still possible with the latest Docker Desktop version.

As there's a lot of setup to be done, I will just show the way to do it, without going too much into details:

1. Start Docker Desktop and switch to Windows Containers (if not the default)

![](/images/docker-settings-switch-win-containers.png)

![](/images/docker-settings-switch-win-containers-confirm.png)

2. In powershell set the variable `$env:DOCKER_HOST` to the Docker Windows named pipe

   ```bash
   # Sets the variable DOCKER_HOST for the Windows docker client
   $env:DOCKER_HOST='npipe:////./pipe/docker_engine_windows'
   ```

![](/images/docker-windows-named-pipe.png)

3. Once done, switch back to Linux containers, it will be needed for the WSL2 node

![](/images/docker-settings-switch-linux-containers.png)

![](/images/docker-settings-switch-linux-containers-confirm.png)

And we're done on the first node. Please note that this won't remain upon restart of Docker Desktop or reboot of the computer.

So it's quite safe to do it and won't harm the "standard" setup of your computer. 

# WSL2: Linux on the (Docker) Desktop

For the WSL2 distro, let's take the "stable" distro in it's newest and shiny version:

![](/images/docker-swarm-workers-join.png)

Now, in order to have access to Docker Desktop, thanks to the new configuration and the WSL2 backend, I have to enable the integration in the settings:

![](/images/wsl2-docker-settings-enable-distro.png)

Once done, click on "Apply & Restart" and confirm docker is correctly mounted (bonus: [Simon's WSLConf video](https://www.youtube.com/watch?v=FD3UVCipmmE) to explain it all):

```bash
# Verifies which OS is docker running on
docker info --format '{{ .OperatingSystem }}'
# Verifies that the client can connect to the engine
docker version
```

![](/images/wsl2-docker-check-version.png)

And we've done the second node and as you can see (hopefully), Docker Desktop is running "twice". Cool right?

Let's now move into our second Linux node.

# Multipass: the Fossa is everywhere

Canonical has this multi-platform tool for creating an Ubuntu VM from the command line called [Multipass](https://multipass.run/).

The goal is, à la Docker, to have explicit while "simple" commands to create, manage and delete an Ubuntu instance. And thanks to the multi-platform availability (read: Linux, MacOS, Windows), it's gaining a strong momentum.

Ok, still here? good, let's create our new Multipass Ubuntu instance:

1. Check which virtualization backend is chosen

   ```bash
   # Check the driver used by Multipass
   multipass get local.driver
   # List all created instances
   multipass list
   ```

![](/images/multipass-check-driver.png)

2. Create and launch a new instance (the cloud-init idea came from [Bret Fisher video on multipass](https://www.pscp.tv/w/1mrGmQvNEWBGy))

   ```bash
   # Check content of a cloud-init configuration file
   cat docker.yaml
   # Creates the new instance with the cloud-init configuration file and with the focal image
   multipass launch --name <hostname> --cloud-init docker.yaml focal
   # Check that docker was installed and it's running
   multipass exec <hostname> docker version
   ```

![](/images/multipass-check-install.png)

One more node configured and ready to join the swarm. As the script from Docker (used by [Bret](https://twitter.com/BretFisher) in his video) is still not ready for Ubuntu 20.04 (a.k.a Focal Fossa), we had to install the package from the repository.

# Win2019: Hyper-Vagrant

To mix things a bit, let's use another deployment tool: [Vagrant](https://www.vagrantup.com/).

While [Multipass](https://multipass.run/) helped us have an Ubuntu instance, for Windows Server we have normally to install it on a Virtual Machine (Hyper-V or any other solution of your choice). Then run through some configuration (and reboots), to finally be able to use it.

The whole process can be done fast if we know what to do, but what if we could have the same workflow has [Multipass](https://multipass.run/): download a preconfigured image and run it.

Well, say hello to [Vagrant](https://www.vagrantup.com/intro/getting-started/). Again, I won't explain much here as, for one I'm at very beginner level with Vagrant and [(Captain) Stefan Scherer](https://twitter.com/stefscherer) has an [excellent blog about it](https://github.com/StefanScherer/docker-windows-box).

He already created a [Windows Server 2019 with Docker image](https://app.vagrantup.com/StefanScherer/boxes/windows_2019_docker) and that's what we will use:

1. Create the Vagrantfile

   ```bash
   # Download the Vagrantfile
   vagrant init StefanScherer/windows_2019_docker
   # Check the Vagrantfile > remove the commented and empty lines
   cat Vagrantfile | Select-String -NotMatch -Pattern "#|^$"
   ```

![](/images/vagrant-init.png)

2. Create a new VM -> on the first run, the image (Box) needs to be downloaded, so it might take quite some time depending on your internet speed

   ```bash
   # Create the new VM on Hyper-V
   vagrant up --provider hyperv
   ```

![](/images/vagrant-up-hyperv.png) ![](/images/vagrant-up-hyperv-username.png)

3. Check the status and if docker is correctly running

   ```bash
   # Check the status of the VM
   vagrant status
   # Check if Docker is running
   vagrant winrm --command "docker version"
   ```

![](/images/vagrant-check-vm.png)

Congratulations! We have now our four nodes: 2 Windows (1 & 4) and 2 Linux (2 & 3).

![](/images/swarm-all-installs.png)

# Docker Swarm: Hybrid is the new normal

Still with me, let's finally have some Swarm fun.

Out of the four nodes, we will have one as the manager (Windows) with three workers (1x Windows + 2x Linux).

It's small and really just for testing purpose, but it will be enough for us to have a view on how Swarm behaves and allocated resources.

## Portainer: show me thy graphics

And when we speak about resources "visualization", what better than a Web UI?

Lucky us, there a very neat project that will help us a lot: [Portainer](https://www.portainer.io/).

And lucky us (yes again), they have a [Windows installation](https://portainer.readthedocs.io/en/latest/deployment.html#windows) which relies on mounting the docker named pipe (making it on par with Linux):

```bash
# Create a new Visualizer container on the Manager node
docker run -d -p 9000:9000 -p 8000:8000 --name portainer --restart always -v \\.\pipe\docker_engine_windows:\\.\pipe\docker_engine -v C:\ProgramData\Portainer:C:\data portainer/portainer
```

![](/images/portainer-create-container.png)

> **Important:** remember that we are still using the "Docker daemon hack", so if we mount `\\.\pipe\docker_engine` it will link to the WSL2 instance (which will become a worker node)

Once the container is running, we can connect to `http://localhost:9000`

![](/images/portainer-first-run.png)

Set a password for the admin account and you're in

![](/images/portainer-first-login.png)

Let's come back to the interface once we have our swarm created.

## Swarm: a Leader arises

We are now ready to start our swarm and the first step is start it on the Manager node.

The Windows with Docker Desktop will be this node. The main reason is that, in terms of network, every other nodes "sees" it. Which is not the case between the other nodes.

We will use the main IP address, which is not optimal but functional for the test we want to do. For a longer term solution, a fix IP will be required.

Let's create the swarm:

```bash
# Get the IP from the "Ethernet" connection
$env:MainIP = (Get-NetIPAddress -InterfaceIndex (Get-NetRoute -DestinationPrefix 0.0.0.0/0).ifIndex).IPv4Address
# Remove the leading space -> yes, this is tricky
$env:MainIP = $env:MainIP.Trim()
# Create the swarm
docker swarm init --advertise-addr $env:MainIP --listen-addr ${env:MainIP}:2377
```

![](/images/docker-swarm-init.png)

## Swarm: hey ho hey ho off to work we go

Now that the Manager node is up and running, it's time to add the worker nodes to the swarm:

```bash
# Add the 3 worker nodes using the swarm join command displayed on the Manager node
docker swarm join --token <swarm token> <manager IP address>:2377
# Check the nodes added from the Manager node
docker node ls
```

![](/images/docker-swarm-workers-join.png)

Great! our swarm is now created and the workers node were added quite fast.

Let's have a look on Portainer what it looks like:

1. Select the `Local` option and click connect

![](/images/portainer-local-connection.png)

2. Click on the endpoint `local`

![](/images/portainer-local-endpoint.png)

3. On the left menu, click on the `swarm` option

![](/images/portainer-swarm-menu.png)

4. We can see our four nodes

![](/images/portainer-swarm-dashboard.png)

## Swarm: labeling is a good thing

As we are in a mixed-os Swarm, docker needs to know to whom it should assign the creation of the containers based on the OS. While Linux containers can run on Windows, it's not the case for Windows containers on Linux (yet).

Let's add a label to the nodes with their OS as the value:

```bash
# Get a list of the nodes
docker node ls
# From the manager node, add the label to the Linux nodes
docker node update --label-add os=linux <linux worker node 1>
docker node update --label-add os=linux <linux worker node 2>
# From the manager node, add the label to the Windows worker node
docker node update --label-add os=linux <windows worker node>
# Check that the label has been correctly set
 docker node inspect <worker node> --format '{{ .Spec.Labels.os }}'
```

![](/images/docker-swarm-labels.png)

On Portainer, we can click on the link "Go to cluster visualizer" and enable the option "Display node labels"

![](/images/portainer-swarm-labels.png)

With the labels applied, we can now launch our "mixed application".

## Swarm: building the foundations

We will deploy a (remixed) web app from the list of [Awesome Compose](https://github.com/docker/awesome-compose): [Asp.net (on Windows) and MS SQL (on Linux)](https://github.com/docker/awesome-compose/tree/master/aspnet-mssql).

But before we can deploy it, as `docker stack` [does not support](https://docs.docker.com/compose/compose-file/#build) building an image during deployment, we will need to build it before and then reference it in our modified compose file.

> Note: Quick step prior the building, download (`git clone`) the Awesome Compose repository to the directory of your choice.

Let's build the image:

1. Go to the `app` directory which contains the `Dockerfile`

   ```bash
   # Move to the app directory > in the screenshot you will see a copy of aspnet-mssql directory
   cd <path to Awesome Compose directory>/aspnet-mssql/app
   # List the content and ensure the Dockerfile is there
   dir
   ```

![](/images/docker-swarm-get-files.png)

2. Edit the `Dockerfile`

   ```bash
   # Open the Dockerfile in the editor of your choice > I will use VSCode
   code Dockerfile
   # Modify both FROM lines in order to have the Windows image tags
   FROM mcr.microsoft.com/dotnet/core/sdk:3.1.201-nanoserver-1909 AS build
   ...
   FROM mcr.microsoft.com/dotnet/core/aspnet:3.1.3-nanoserver-1909 AS runtime
   ...
   # Save the file
   ```

![](/images/docker-swarm-dockerfile-edit.png)

3. Build the image

   ```bash
   # Build the image with the tag you want > I will name swarmaspnet
   docker build -t <your DockerHub repository name>/<image name>:latest .
   ```

![](/images/docker-swarm-build-image.png)

4. Check the image

   ```bash
   # Check if the image has been correctly created
   docker image ls <your DockerHub repository name>/<image name>
   ```

![](/images/docker-swarm-check-image.png)

5. Finally, we will need to push the image as it's living only in our Manager node but we need it on our Windows Worker

   ```bash
   # Login to Docker Hub
   docker login
   # Push the image
   docker push <your DockerHub repository name>/<image name>:latest
   ```

![](/images/docker-swarm-push-image.png)

We have now all the components to deploy our application in a mixed environment.

## Swarm: the "divide to conquer" deployment

Let's deploy our application from the Manager node:

1. Edit the compose file to add the image reference and add constraints on the labels we created

   ```bash
   # Move one directory up, where the compose file is located
   cd ..
   # Edit the compose file with your favorite editor
   code docker-compose.yaml
   # Remove the build line and add the reference to your image
   ...
   web:
     image: <your DockerHub repository name>/<image name>:latest
   ...
   # Add the constraint to Windows OS for the WEB service
   ...
   web:
     image: nunix/swarmaspnet:latest
     deploy:
       placement:
         constraints:
           - node.labels.os == windows
   ...
   # Add the constraint to Linux OS for the DB service
   ...
   db:
     image: microsoft/mssql-server-linux
     deploy:
       placement:
         constraints:
           - node.labels.os == linux
   ...
   # Save the file
   ```

![](/images/docker-swarm-edit-compose.png)

2. Deploy the application

   ```bash
   # Deploy the application stack > I will name it hybridswarm
   docker stack deploy --compose-file docker-compose.yaml <application name>
   # Check if the stack has been correctly deployed
   docker stack ls
   # Check the services created with our deployment
   docker stack ps <application name>
   ```

![](/images/docker-swarm-stack-deploy.png)

3. Check the services and the status on the nodes

   ```bash
   # List the services
   docker service ls
   # Check on which node the DB service has been created > should be a Linux node
   docker service ps <DB service name>
   # Check on which node the WEB service has been created > should be the Windows node
   docker service ps <WEB service name>
   # [Optional] Check the containers created on worker node
   docker container ls
   ```

![](/images/docker-swarm-check-services.png)

4. Check on Portainer what is displayed > for a cleaner view, enable the "Only display running tasks" option

![](/images/portainer-check-services.png)

5. Finally, open a browser and enter the IP of the Windows worker node to see the website !

[](/images/docker-swarm-check-website.png)

And here we have, an application deployed in an hybrid infrastructure, and the whole thing running on a single computer and Windows 10.

# Conclusion

Like I said in the introduction, while the concept might help you, the mix of technologies was really to showcase the possibilities of Windows 10 and Docker Desktop.

The combo opens the door to real opportunities, and for me, it's primarily an amazing learning platform.

I really hope you will have as much fun as me, and at the same time you learnt something.

> ***\>>> Nunix out <<<***

# Bonus 1: scaling one node at the time

Well, in our setup, there is one Linux worker node that was not used, let remediate to that and scale the DB service to be replicated to two instances:

```bash
# List the services
docker service ls
# Scale the DB service to have two replicas
docker service scale <DB service name>=2
# Check if the service has been correctly replicated
docker service ls
# [Optional] Check the containers created on each Linux worker node
docker container ls
```

![](/images/docker-swarm-scale-services.png)

Check also in Portainer what is displayed

![](/images/portainer-check-scaled-services.png)

And voilà, we could replicate a service and now we do have redundancy for the DB service (at least in terms of nodes).