---
layout: post
title:  Install Nvidia GPU Driver And Nvidia Docker in Ubuntu EC2 instance
date:   2016-12-17T21:40:00+8:00
---

### Install Docker

Follow the instructions on the docker website<sup>[1](#1)</sup> to install `docker-engine`.

### Install Nvidia GPU driver

Follow the instructions in nvidia-docker wiki<sup>[2](#2)</sup> to install the gpu driver and nvidia-docker.
Here I create a GPU EC2 instance using Ubuntu 16.04 LTS AMI by AWS web console intead of docker-machine.

![]({{site.baseurl}}/images/create-ubuntu-gpu-instance-on-aws.png)

```bash
# ssh to the instance 
$ ssh -i {your key}.pem ubuntu@{instance ip or domain name}

# Install official NVIDIA driver package
$ sudo apt-key adv --fetch-keys http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/7fa2af80.pub
$ sudo sh -c 'echo "deb http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64 /" > /etc/apt/sources.list.d/cuda.list'
$ sudo apt-get update && sudo apt-get install -y --no-install-recommends cuda-drivers

# Install nvidia-docker and nvidia-docker-plugin
$ wget -P /tmp https://github.com/NVIDIA/nvidia-docker/releases/download/v1.0.0-rc.3/nvidia-docker_1.0.0.rc.3-1_amd64.deb
$ sudo dpkg -i /tmp/nvidia-docker*.deb && rm /tmp/nvidia-docker*.deb
```

Next, verify the driver is installed.

```bash
$ nvidia-smi
```

![]({{site.baseurl}}/images/nvidia-smi.png)

And the nvidia-docker.

```bash
$ nvidia-docker
```

### Troubleshooting

#### Error: Unable to load the kernel module 'nvidia.ko'

If you see the error message like this<sup>[3](#3)</sup>

```
Unable to load the kernel module 'nvidia.ko'.  This happens most frequently when this kernel module was built against the wrong or
         improperly configured kernel sources, with a version of gcc that differs from the one used to build the target kernel, or if a driver
         such as rivafb, nvidiafb, or nouveau is present and prevents the NVIDIA kernel module from obtaining ownership of the NVIDIA graphics
         device(s), or no NVIDIA GPU installed in this system is supported by this NVIDIA Linux graphics driver release.

         Please see the log entries 'Kernel module load error' and 'Kernel messages' at the end of the file '/var/log/nvidia-installer.log'
         for more information.
```

execute following commands to solve the issue. 

```bash
$ sudo apt-get install linux-image-extra-virtual
$ reboot
```

#### Error: Unable to find the kernel source tree

If the installation shows `Uable to find the kernel source tree` error<sup>[4](#4)</sup>, then execute the command

```bash
$ sudo apt-get install linux-headers-`uname -r`
```

#### Disable nouveau

[Install Nvidia driver instead nouveau](http://askubuntu.com/questions/481414/install-nvidia-driver-instead-nouveau) shows how to disable nouveau.

### Reference
[<a name="1">1</a>] [Install Docker on Ubuntu](https://docs.docker.com/engine/installation/linux/ubuntulinux/), Docker<br/>
[<a name="2">2</a>] [Deploy on Amazon EC2](https://github.com/NVIDIA/nvidia-docker/wiki/Deploy-on-Amazon-EC2), nvidia-docker, Nvidia<br/>
[<a name="3">3</a>] [CUDA 6.5 on AWS GPU Instance Running Ubuntu 14.04](http://tleyden.github.io/blog/2014/10/25/cuda-6-dot-5-on-aws-gpu-instance-running-ubuntu-14-dot-04/), 
Seven Story Rabbit Hole<br/>
[<a name="4">4</a>] [NVIDIA driver install - Error: Unable to find the kernel source tree](https://ubuntuforums.org/showthread.php?t=843914), Ubuntu forums<br/>
