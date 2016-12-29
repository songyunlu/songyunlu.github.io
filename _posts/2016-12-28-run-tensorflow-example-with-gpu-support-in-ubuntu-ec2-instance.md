---
layout: post
title:  Run TensorFlow Example with GPU support in Ubuntu EC2 Instance
date:   2016-12-28T11:40:00+8:00
---

Follow up the [previous post]({{site.baseurl}}/2016/12/17/install-nvidia-gpu-driver-and-nvidia-docker-in-ubuntu-ec2-instance/), we can now run tensorflow with GPU docker image directly<sup>[1](#1)</sup>.

First, ssh into your EC2 ubuntu instnace then run the docker container by `nvidia-docker`.

```bash
sudo nvidia-docker run -d -p 8888:8888 tensorflow/tensorflow:latest-gpu

# 09b23108...... (container id)
```

Retrieve logs from the detached container.

```bash
sudo nvidia-docker logs [container id]

# [I 04:19:58.174 NotebookApp] Writing notebook server cookie secret to /root/.local/share/jupyter/runtime/notebook_cookie_secret
# [W 04:19:58.229 NotebookApp] WARNING: The notebook server is listening on all IP addresses and not using encryption. This is not recommended.
# [I 04:19:58.403 NotebookApp] Serving notebooks from local directory: /notebooks
# [I 04:19:58.404 NotebookApp] 0 active kernels
# [I 04:19:58.404 NotebookApp] The Jupyter Notebook is running at: http://[all ip addresses on your system]:8888/?token=91859f1cf......
# [I 04:19:58.404 NotebookApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
```

As you can see, the docker image comes with Jupyter notebook pre-installed. You can access it via a browser at `http://[ec2 public domain name]:8888?token=91859f1cf......` (do not forget to open port access on the **security group**).

![]({{site.baseurl}}/images/jupyter-notebook-with-tensorflow-example.png)

Open one of the notebook and execute the examples to see if tensoflow work as expected.

### Reference
[<a name="1">1</a>] [Tensorflow on Docker Hub](https://hub.docker.com/u/tensorflow/), Docker Hub
