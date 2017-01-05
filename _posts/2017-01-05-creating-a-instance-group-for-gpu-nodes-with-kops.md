---
layout: post
title:  Creating a instance group for GPU nodes with kops
date:   2017-01-05T15:50:00+8:00
---

To create an instance group contains GPU instances, firt use the command `kops create ig $IG_NAME` to edit your instance group spec.

```yaml
# `kops create ig gpu-nodes` will create your editor of choice to edi the spec.

apiVersion: kops/v1alpha2
kind: InstanceGroup
metadata:
  creationTimestamp: "2017-01-05T07:54:17Z"
  name: gpu-nodes
spec:
  associatePublicIp: true
  image: 051280282358/gpu-ready-instance-for-g2-instance-type  # custom image
  machineType: g2.2xlarge
  maxSize: 10  # Setting maximum number of instances in the instance group.
  minSize: 1  # One instance will up in the instance group
  role: Node
  rootVolumeSize: 30  # setting root volume to 30GB
  subnets:
  - ap-northeast-1a
```

Then issue the command `kops update cluster $CLUSTER_NAME` to reivew the change.

```bash
# Will create resources:
#   LaunchConfiguration/gpu-nodes.apnortheast.k8s.linkernetworks.com
#     ImageID               051280282358/gpu-ready-instance-for-g2-instance-type
#     InstanceType          g2.2xlarge
#     SSHKey                name:kubernetes.apnortheast.k8s.linkernetworks.com-e6:9f:20:49:38:6c:fd:aa:bb:05:ef:96:3b:c6:81:37 id:kubernetes.apnortheast.k8s.linkernetworks.com-e6:9f:20:49:38:6c:fd:aa:bb:05:ef:96:3b:c6:81:37
#     SecurityGroups        [name:nodes.apnortheast.k8s.linkernetworks.com id:sg-a50183c2]
#     AssociatePublicIP     true
#     IAMInstanceProfile    name:nodes.apnortheast.k8s.linkernetworks.com id:nodes.apnortheast.k8s.linkernetworks.com
#     RootVolumeSize        30
#     RootVolumeType        gp2
# 
#   AutoscalingGroup/gpu-nodes.apnortheast.k8s.linkernetworks.com
#     MinSize               1
#     MaxSize               10
#     Subnets               [name:ap-northeast-1a.apnortheast.k8s.linkernetworks.com id:subnet-b87e1ece]
#     Tags                  {k8s.io/role/node: 1, Name: gpu-nodes.apnortheast.k8s.linkernetworks.com, KubernetesCluster: apnortheast.k8s.linkernetworks.com}
#     LaunchConfiguration   name:gpu-nodes.apnortheast.k8s.linkernetworks.com
```

Finally `kops update cluster $KOPS_CLUSTER_NAME --yes` to apply the change.

```bash
# I0105 15:58:44.697782   29759 dns.go:124] Pre-creating DNS records
# I0105 15:58:45.686061   29759 update_cluster.go:188] Exporting kubecfg for cluster
# Wrote config for apnortheast.k8s.linkernetworks.com to "/Users/Jimmy/.kube/config"
# Kops has changed your kubectl context to apnortheast.k8s.linkernetworks.com
# ➜  songyunlu.github.io git:(master) ✗ kops update cluster $KOPS_CLUSTER_NAME --yes
```

Check the result by `kops get ig gpu-nodes`.

```bash
# NAME    ROLE  MACHINETYPE  MIN  MAX  SUBNETS
# gpu-nodes  Node  g2.2xlarge  1  10  ap-northeast-1a
```

You should see one more auto scaling group for GPU nodes is created on AWS web console.

![]({{site.baseurl}}/images/auto-scaling-group-for-gpu-nodes.png)

And one GPU instance started.

![]({{site.baseurl}}/images/ec2-gpu-instance-started-by-auto-scaling-group.png)
