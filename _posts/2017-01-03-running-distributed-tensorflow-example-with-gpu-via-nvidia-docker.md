---
layout: post
title:  Running Distributed TensorFlow Example with GPU via Nvidia Docker
date:   2017-01-03T15:40:00+8:00
---

If you had followed [this]({{site.baseurl}}/2016/12/17/installing-nvidia-gpu-driver-and-nvidia-docker-in-ubuntu-ec2-instance/) and [this]({{site.baseurl}}/2016/12/28/running-tensorflow-example-with-gpu-support-in-ubuntu-ec2-instance/) to setup a EC2 instance with GPU support, you are ready to run distirbuted tensorflow examples. 

You might want to create image (AMI) of this instnace first so you can create more instances for running distributed examples.

Here I create two **tensorflow worker** and one **parameter server** instances.
Please note that the instance running parameter server need not to be a GPU instance, since it's merely used to store parameters calculated by the workers.

![]({{site.baseurl}}/images/distributed-tensorflow-ec2-instances.png)

Make sure the traffic between these instances are not blocked by the firewall. 
Configure **security group** based on your internal IP range so they can talk to each other.

![]({{site.baseurl}}/images/security-group-configurations-for-distributed-tensorflow-ec2-instances.png)

Based on the tensorlfow image, I've made a docker image [`gn00023040/distributed-tensorflow-example`](https://hub.docker.com/r/gn00023040/distributed-tensorflow-example/) contains the [example.py provided by Imanol](https://github.com/ischlag/distributed-tensorflow-example).

```
# Dockerfile
FROM tensorflow/tensorflow:latest-gpu
MAINTAINER Jimmy Lu <slu@linkernetworks.com>
RUN curl -O http://blog.jimmylu.idv.tw/assets/distributed-tensorflow-example.py
```

You can simply execute the commands below on three instances respectively to start the training.

```bash
# on worker 1 instance
# $WORKER_ADDRESS_LIST is a comma-separated address list of your workers. E.g. 172.31.8.107:2222,172.31.8.108:2222
# $PARAMETER_SERVER_ADDRESS_LIST is a comma-separated address list of your parameter servers. E.g. 172.31.21.167:2222
$ sudo docker run -d -p 2222:2222 `curl -s http://localhost:3476/docker/cli` gn00023040/distributed-tensorflow-example \
      python distributed-tensorflow-example.py \
          --workers="$WORKER_ADDRESS_LIST" \
          --parameter_servers="$PARAMETER_SERVER_ADDRESS_LIST" \
          --job_name="worker" \
          --task_index=0
```

```bash
# on worker 2 instance
$ sudo docker run -d -p 2222:2222 `curl -s http://localhost:3476/docker/cli` gn00023040/distributed-tensorflow-example \
      python distributed-tensorflow-example.py \
          --workers="$WORKER_ADDRESS_LIST" \
          --parameter_servers="$PARAMETER_SERVER_ADDRESS_LIST" \
          --job_name="worker" \
          --task_index=1
```

```bash
# on parameter server instance
$ sudo docker run -d -p 2222:2222 gn00023040/distributed-tensorflow-example \
      python distributed-tensorflow-example.py \
           --workers="$WORKER_ADDRESS_LIST" \
           --parameter_servers="$PARAMETER_SERVER_ADDRESS_LIST" \
           --job_name="ps" \
           --task_index=0
```

The output on one of your worker node.

```bash
# Step: 18700,  Epoch: 18,  Batch: 550 of 550,  Cost: 2.8933,  AvgTime: 7.90ms
# Step: 18902,  Epoch: 19,  Batch: 100 of 550,  Cost: 2.7845,  AvgTime: 16.36ms
# Step: 19104,  Epoch: 19,  Batch: 200 of 550,  Cost: 3.4632,  AvgTime: 15.63ms
# Step: 19305,  Epoch: 19,  Batch: 300 of 550,  Cost: 2.6277,  AvgTime: 15.66ms
# Step: 19506,  Epoch: 19,  Batch: 400 of 550,  Cost: 2.0991,  AvgTime: 15.56ms
# Step: 19706,  Epoch: 19,  Batch: 500 of 550,  Cost: 2.7716,  AvgTime: 15.53ms
# Step: 19801,  Epoch: 19,  Batch: 550 of 550,  Cost: 2.0406,  AvgTime: 7.75ms
# Step: 20005,  Epoch: 20,  Batch: 100 of 550,  Cost: 2.1827,  AvgTime: 16.25ms
# Step: 20204,  Epoch: 20,  Batch: 200 of 550,  Cost: 2.5570,  AvgTime: 15.62ms
# Step: 20403,  Epoch: 20,  Batch: 300 of 550,  Cost: 2.4907,  AvgTime: 15.51ms
# Step: 20603,  Epoch: 20,  Batch: 400 of 550,  Cost: 2.3283,  AvgTime: 15.60ms
# Step: 20804,  Epoch: 20,  Batch: 500 of 550,  Cost: 2.5826,  AvgTime: 15.64ms
# Step: 20900,  Epoch: 20,  Batch: 550 of 550,  Cost: 2.5358,  AvgTime: 7.79ms
# Test-Accuracy: 0.41
# Total Time: 221.72s
# Final Cost: 2.5358
# done
```

Because `log_device_placement` is set to **true** in this example, you'll see a lot of logs showing where the computation takes places.

```bash
# ...
# save/RestoreV2_1/shape_and_slices: (Const): /job:ps/replica:0/task:0/cpu:0
# save/RestoreV2_1/tensor_names: (Const): /job:ps/replica:0/task:0/cpu:0
# save/RestoreV2/shape_and_slices: (Const): /job:ps/replica:0/task:0/cpu:0
# save/RestoreV2/tensor_names: (Const): /job:ps/replica:0/task:0/cpu:0
# save/SaveV2/shape_and_slices: (Const): /job:ps/replica:0/task:0/cpu:0
# save/SaveV2/tensor_names: (Const): /job:ps/replica:0/task:0/cpu:0
# report_uninitialized_variables/boolean_mask/Reshape_1/shape: (Const): /job:ps/replica:0/task:0/cpu:0
# report_uninitialized_variables/boolean_mask/concat/values_0: (Const): /job:worker/replica:0/task:0/gpu:0
# report_uninitialized_variables/boolean_mask/concat/concat_dim: (Const): /job:worker/replica:0/task:0/gpu:0
# report_uninitialized_variables/boolean_mask/strided_slice/stack_2: (Const): /job:worker/replica:0/task:0/gpu:0
# report_uninitialized_variables/boolean_mask/strided_slice/stack_1: (Const): /job:worker/replica:0/task:0/gpu:0
# report_uninitialized_variables/boolean_mask/strided_slice/stack: (Const): /job:worker/replica:0/task:0/gpu:0
# report_uninitialized_variables/boolean_mask/Shape: (Const): /job:worker/replica:0/task:0/gpu:0
# report_uninitialized_variables/Const: (Const): /job:ps/replica:0/task:0/cpu:0
# ...
```

Congratulations! You have successfully run the example.

For more information about distributed tensorflow, take a look at these references.

* [Distributed Tensorflow Documentation](https://www.tensorflow.org/how_tos/distributed/)
* [Distributed Tensorflow on Github](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/g3doc/how_tos/distributed/index.md)
* [Testing Distributed Runtime in TensorFlow](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/tools/dist_test)
* [Distributed TensorFlow by Leo K. Tam](http://leotam.github.io/general/2016/03/13/DistributedTF.html)
* [Distributed Tensorflow Example by Imanol Schlag](https://ischlag.github.io/2016/06/12/async-distributed-tensorflow/)
