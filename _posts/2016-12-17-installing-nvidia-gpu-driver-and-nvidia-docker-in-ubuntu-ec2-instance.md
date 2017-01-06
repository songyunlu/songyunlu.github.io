---
layout: post
title:  Installing Nvidia GPU Driver And Nvidia Docker in Ubuntu EC2 instance
date:   2016-12-17T21:40:00+8:00
---

### Install Docker

Follow the instructions in [Install Docker on Ubuntu](https://docs.docker.com/engine/installation/linux/ubuntulinux/) to install `docker-engine`.

### Install GPU driver

Follow the instructions in [Deploy on Amazon EC2](https://github.com/NVIDIA/nvidia-docker/wiki/Deploy-on-Amazon-EC2) in nvidia-docker wiki to install the GPU driver and nvidia-docker.
Here I create a GPU EC2 instance using Ubuntu 16.04 LTS AMI by AWS web console intead of docker-machine (**Note that different instance type (P2, G2, CG1) has different GPU hardware, please refere to the [document](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/accelerated-computing-instances.html) and [Nvidia official website](http://www.nvidia.com/Download/Find.aspx) to find out the GPU product of your instance and its corresponding driver.**)

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

# Thu Dec 29 09:20:51 2016
# +-----------------------------------------------------------------------------+
# | NVIDIA-SMI 367.57                 Driver Version: 367.57                    |
# |-------------------------------+----------------------+----------------------+
# | GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
# | Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
# |===============================+======================+======================|
# |   0  GRID K520           Off  | 0000:00:03.0     Off |                  N/A |
# | N/A   31C    P8    17W / 125W |      0MiB /  4036MiB |      0%      Default |
# +-------------------------------+----------------------+----------------------+
# 
# +-----------------------------------------------------------------------------+
# | Processes:                                                       GPU Memory |
# |  GPU       PID  Type  Process name                               Usage      |
# |=============================================================================|
# |  No running processes found                                                 |
# +-----------------------------------------------------------------------------+
```

And the nvidia-docker.

```bash
$ sudo nvidia-docker

# Usage: docker [OPTIONS] COMMAND [arg...]
#        docker [ --help | -v | --version ]
# 
# A self-sufficient runtime for containers.
# 
# Options:
# 
#   --config=~/.docker              Location of client config files
#   -D, --debug                     Enable debug mode
#   -H, --host=[]                   Daemon socket(s) to connect to
#   -h, --help                      Print usage
#   -l, --log-level=info            Set the logging level
#   --tls                           Use TLS; implied by --tlsverify
#   --tlscacert=~/.docker/ca.pem    Trust certs signed only by this CA
#   --tlscert=~/.docker/cert.pem    Path to TLS certificate file
#   --tlskey=~/.docker/key.pem      Path to TLS key file
#   --tlsverify                     Use TLS and verify the remote
#   -v, --version                   Print version information and quit
```

### Troubleshooting

#### [Error: Unable to load the kernel module 'nvidia.ko'](http://tleyden.github.io/blog/2014/10/25/cuda-6-dot-5-on-aws-gpu-instance-running-ubuntu-14-dot-04/)

If you see the error message below.

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

#### [Error: Unable to find the kernel source tree](https://ubuntuforums.org/showthread.php?t=843914)

If the installation shows `Uable to find the kernel source tree` error, then execute the command

```bash
$ sudo apt-get install linux-headers-`uname -r`
```

#### Disable nouveau

[Install Nvidia driver instead nouveau](http://askubuntu.com/questions/481414/install-nvidia-driver-instead-nouveau) shows how to disable nouveau.

