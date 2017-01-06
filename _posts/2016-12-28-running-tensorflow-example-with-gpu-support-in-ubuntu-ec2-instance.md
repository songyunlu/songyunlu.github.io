---
layout: post
title:  Running TensorFlow Example with GPU support in Ubuntu EC2 Instance
date:   2016-12-28T11:40:00+8:00
---

Follow up the [previous post]({{site.baseurl}}/2016/12/17/install-nvidia-gpu-driver-and-nvidia-docker-in-ubuntu-ec2-instance/), we can now run [tensorflow with GPU docker image](http://askubuntu.com/questions/481414/install-nvidia-driver-instead-nouveau) directly.

### Nvidia Docker architecture

Let us take a look at the architecture of Nvidia Docker images. As you can see, we have installed the CUDA driver (GPU driver) on the host machine to control and communicate with GPUs. Based on that, every GPU application in the docker container sould require [nvidia/cuda](https://hub.docker.com/r/nvidia/cuda/) (CUDA toolkit) image to get access to CUDA APIs (E.g. tensorflow docker image).

<span class="no-border">![]({{site.baseurl}}/images/nvidia-docker-architecture.png)</span>

Please make sure that you use the right version of CUDA toolkit based on the driver, or you'll get the *Failed to initialize NVML: Driver/library version mismatch* or *CUDA driver version is insufficient for CUDA runtime version* errors when running the application. 

You can find out the version combatibility table [here](https://github.com/NVIDIA/nvidia-docker/wiki/CUDA#requirements).

### Start a TensorFlow environment with *nvidia-docker*

First, ssh into your EC2 ubuntu instnace then run the docker container by `nvidia-docker`.

```bash
$ sudo nvidia-docker run -d -p 8888:8888 tensorflow/tensorflow:latest-gpu

# 09b23108...... (container id)
```

Retrieve logs from the detached container.

```bash
$ sudo nvidia-docker logs $CONTAINER_ID

# [I 04:19:58.174 NotebookApp] Writing notebook server cookie secret to /root/.local/share/jupyter/runtime/notebook_cookie_secret
# [W 04:19:58.229 NotebookApp] WARNING: The notebook server is listening on all IP addresses and not using encryption. This is not recommended.
# [I 04:19:58.403 NotebookApp] Serving notebooks from local directory: /notebooks
# [I 04:19:58.404 NotebookApp] 0 active kernels
# [I 04:19:58.404 NotebookApp] The Jupyter Notebook is running at: http://[all ip addresses on your system]:8888/?token=91859f1cf......
# [I 04:19:58.404 NotebookApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
```

### Start a TensorFlow environment with *docker*

Alternatively, you can use `docker` instead of `nvidia-docker` wrapper. 

```bash
# You can find out $DRIVER_VERSION via `nvidia-smi` command.
$ sudo docker volume create --name=nvidia_driver_$DRIVER_VERSION -d nvidia-docker
$ sudo docker run --volume=nvidia_driver_$DRIVER_VERSION:/usr/local/nvidia:ro --device=/dev/nvidiactl --device=/dev/nvidia-uvm --device=/dev/nvidia0 -it -p 8888:8888 tensorflow/tensorflow:latest-gpu
```

Please refer to the [wiki of nvidia-docker](https://github.com/NVIDIA/nvidia-docker/wiki/Internals) for more details.

### Run examples on the Jupyter Notebook.

As you can see from the logs, the docker image comes with Jupyter notebook pre-installed. You can access it via a browser at `http://[ec2 public domain name]:8888?token=91859f1cf......` (do not forget to open the port on the **security group**).

![]({{site.baseurl}}/images/jupyter-notebook-with-tensorflow-example.png)

Open one of the notebook and execute the examples to see if tensoflow work as expected.

To see if the tensorflow uses the GPU as we want, create a new notebook and execute the code below. 

```python
import tensorflow as tf

a = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[2, 3], name='a')
b = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[3, 2], name='b')
c = tf.matmul(a, b)
sess = tf.Session(config=tf.ConfigProto(log_device_placement=True))
print sess.run(c)

# THE OUTPUT ON YOUR TERMINAL SCREEN.
# I tensorflow/stream_executor/dso_loader.cc:128] successfully opened CUDA library libcublas.so locally
# I tensorflow/stream_executor/dso_loader.cc:128] successfully opened CUDA library libcudnn.so locally
# I tensorflow/stream_executor/dso_loader.cc:128] successfully opened CUDA library libcufft.so locally
# I tensorflow/stream_executor/dso_loader.cc:128] successfully opened CUDA library libcuda.so.1 locally
# I tensorflow/stream_executor/dso_loader.cc:128] successfully opened CUDA library libcurand.so locally
# I tensorflow/stream_executor/cuda/cuda_gpu_executor.cc:937] successful NUMA node read from SysFS had negative value (-1), but there must be at least one NUMA node, so returning NUMA node zero
# I tensorflow/core/common_runtime/gpu/gpu_device.cc:885] Found device 0 with properties:
# name: GRID K520
# major: 3 minor: 0 memoryClockRate (GHz) 0.797
# pciBusID 0000:00:03.0
# Total memory: 3.94GiB
# Free memory: 3.91GiB
# I tensorflow/core/common_runtime/gpu/gpu_device.cc:906] DMA: 0
# I tensorflow/core/common_runtime/gpu/gpu_device.cc:916] 0:   Y
# I tensorflow/core/common_runtime/gpu/gpu_device.cc:975] Creating TensorFlow device (/gpu:0) -> (device: 0, name: GRID K520, pci bus id: 0000:00:03.0)
# Device mapping:
# /job:localhost/replica:0/task:0/gpu:0 -> device: 0, name: GRID K520, pci bus id: 0000:00:03.0
# I tensorflow/core/common_runtime/direct_session.cc:255] Device mapping:
# /job:localhost/replica:0/task:0/gpu:0 -> device: 0, name: GRID K520, pci bus id: 0000:00:03.0
# 
# MatMul: (MatMul): /job:localhost/replica:0/task:0/gpu:0
# I tensorflow/core/common_runtime/simple_placer.cc:827] MatMul: (MatMul)/job:localhost/replica:0/task:0/gpu:0
# b: (Const): /job:localhost/replica:0/task:0/gpu:0
# I tensorflow/core/common_runtime/simple_placer.cc:827] b: (Const)/job:localhost/replica:0/task:0/gpu:0
# a: (Const): /job:localhost/replica:0/task:0/gpu:0
# I tensorflow/core/common_runtime/simple_placer.cc:827] a: (Const)/job:localhost/replica:0/task:0/gpu:0
```

The log `/job:localhost/replica:0/task:0/gpu:0` shows that we are using GPU device for the computations.

