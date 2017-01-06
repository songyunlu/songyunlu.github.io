---
layout: post
title:  Setting up OpenAI's Kubernetes EC2 autoscaler in the cluster installed by kops
date:   2017-01-05T17:50:00+8:00
---

Recently, OpenAI published a [blog post](https://openai.com/blog/infrastructure-for-deep-learning/) describing their deep learning infrastructure. In the post, they mentioned their workload is bursty and unpredictable, so they developed and opensourced [kubernetes-ec2-autoscaler](https://github.com/openai/kubernetes-ec2-autoscaler) to scale in/out GPU instances based on their workload dynamically and automatically. 

We had similar workload pattern as OpenAI in Linker Networks, so I decided to give it a try in the cluster I set up with kops. But it took me some time to figure out the correct way to make it work. 

### Add the right tags to Auto Scaling group and the instance it controlls

Kops created a auto-scaling group with three default tags

Name              | Value
------------------|---------------------------------------------
KubernetesCluster | apnortheast.k8s.linkernetworks.com
------------------|---------------------------------------------
Name              | gpu-nodes.apnortheast.k8s.linkernetworks.com
------------------|---------------------------------------------
k8s.io/role/node  | 1
------------------|---------------------------------------------

According to the README of autoscaler

> Your Auto Scaling Groups must be tagged with <code class="highlight-rouge">KubernetesCluster</code>, whose value should be passed to the flag <code class="highlight-rouge">--cluster-name</code>, and <code class="highlight-rouge">Role</code> with a value of <code class="highlight-rouge">minion</code>.

We already have the tag `KubernetesCluster` defined, so all we need is to add additional tag `Role=minion` to the ASG. But in the end it doesn't seem to work. I traced the code and found that in the `autoscaling_groups.py`, the correct value of `Role` tag is defined to be `worker` or `kubernetes-minion`.

```python
class AutoScalingGroups(object):
    _BOTO_CLIENT_TYPE = 'autoscaling'

    _CLUSTER_KEY = 'KubernetesCluster'
    _ROLE_KEYS = ('KubernetesRole', 'Role')
    # the value of Role tag should be 'worker' or 'kubernetes-minion' instead of 'minion'
    _WORKER_ROLE_VALUES = ('worker', 'kubernetes-minion') 
...
```

For ASG which created by kops, we can use `cloudLabels` in the spec of instance group to add AWS tags.

```yaml
apiVersion: kops/v1alpha2
kind: InstanceGroup
metadata:
  creationTimestamp: "2017-01-05T07:54:17Z"
  name: gpu-nodes
spec:
  associatePublicIp: true
  image: 051280282358/gpu-ready-instance-for-g2-instance-type
  machineType: g2.2xlarge
  maxSize: 10
  minSize: 0
  role: Node
  rootVolumeSize: 30
  subnets:
  - ap-northeast-1a
  # define labels
  cloudLabels:
    Role: kubernetes-minion 
    gpu: "true"
  nodeLabels:
    gpu: "true"
```

You might need to define `nodeLabel` so that you can assingn pods to desired instances with specific labels by `nodeSelector`.

### Enable instance protection to Auto Scaling group

Autoscaler also requires you to enable [Instance Protection](http://docs.aws.amazon.com/autoscaling/latest/userguide/as-instance-termination.html#instance-protection) so the instance termination is controlled by the autosaler. 

Currenlty you can't enable this settings via kops (see issue [#490](https://github.com/kubernetes/kops/issues/490)). You have to do it manually by ASG web console or aws-cli.

### Base64 encoded access key/secret

The document says that the access key and secret on the document. However, the description is not clearly emphasized. It's most likely you'll miss this setting and get errors of permission denied when accessing to AWS resources. Make sure to set **base64 encoded access key and secret of the IAM user** to the secret you'll create in kubernetes.

```yaml
# secret.yaml

apiVersion: v1
kind: Secret
metadata:
  name: autoscaler
  namespace: kube-system
data:
  aws-access-key-id: [base64 encoded access key]
  aws-secret-access-key: [base64 encoded secret access key]
```

Note that kops creates two namespace `default` and `kube-system` by default. The later is different from `system` shown in the document.

```bash
$ kubectl get namespace

# NAME          STATUS    AGE
# default       Active    1d
# kube-system   Active    1d
```

I choosed to create all the resources under `kube-system` namespace.
