---
layout: post
title:  Run Distributed TensorFlow Example
date:   2016-12-28T14:40:00+8:00
---

If you have followed [this]({{site.baseurl}}/2016/12/17/install-nvidia-gpu-driver-and-nvidia-docker-in-ubuntu-ec2-instance/) and [this]({{site.baseurl}}/2016/12/28/run-tensorflow-example-with-gpu-support-in-ubuntu-ec2-instance/) to setup a ec2 instance with GPU support, you are ready to run distirbuted tensorflow examples. 


You might want to create image (AMI) of this instnace first so you can create more instances for running distributed executions.

Here I create two **tensorflow worker** and one **parameter server** instances.
Please note that the instance running parameter server need not have GPU driver installed, since it's merely used to store parameter calculated by the workers.

![]({{site.baseurl}}/images/distributed-tensorflow-ec2-instances.png)

Make sure the traffic between these instances are not blocked by the firewall. 
Configure **security group** based on your internal IP range so they can talk to each other.

![]({{site.baseurl}}/images/security-group-configurations-for-distributed-tensorflow-ec2-instances.png)

Login to all three instances and save the [example.py provided by Imanol](https://github.com/ischlag/distributed-tensorflow-example) to local file system.

Modify `parameter_servers` and `workers` to your instance IP address.

```python
# cluster specification
parameter_servers = ["172.31.24.102:2220"]
workers = ["172.31.3.133:2220", "172.31.13.139:2220"]
cluster = tf.train.ClusterSpec({"ps":parameter_servers, "worker":workers})
```

Then execute the commands on the three instances.

```bash
python example.py --job_name="worker" --task_index=0 # on worker 1 instance
python example.py --job_name="worker" --task_index=1 # on worker 2 instance
python example.py --job_name="ps" --task_index=0 # on parameter server instance

# You'll see the similar output on your worker instance.
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

Congradulations! You have successfully run the example.

For more information about distributed tensorflow, take a look at these references.

* [Distributed Tensorflow Documentation](https://www.tensorflow.org/how_tos/distributed/)
* [Distributed Tensorflow on Github](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/g3doc/how_tos/distributed/index.md)
* [Testing Distributed Runtime in TensorFlow](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/tools/dist_test)
* [Distributed TensorFlow by Leo K. Tam](http://leotam.github.io/general/2016/03/13/DistributedTF.html)
* [Distributed Tensorflow Example by Imanol Schlag](https://ischlag.github.io/2016/06/12/async-distributed-tensorflow/)
