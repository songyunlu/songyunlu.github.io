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
  minSize: 1
  role: Node
  rootVolumeSize: 30
  subnets:
  - ap-northeast-1a
  # define labels
  cloudLabels:
    KubernetesRole: kubernetes-minion 
    kube/gpu: "true"
  nodeLabels:
    gpu: "true"
```

You might need to define `nodeLabel` so that you can assingn pods to desired instances with specific labels by `nodeSelector`.

Please make sure you add `kube/` prepended to you node labels as your custom cloud label for tagging the ASG. The autoscaler will use this tag as the selector to select right AGSs for scaling nodes. You can find the implementation details in following codes.

```python
# autoscaling_groups.py
# There are some predefined selectors you can use as the tags of ASGs.
def _extract_selectors(self, region, launch_config, tags_data):
    # predefined selectors
    selectors = {
        'aws/type': launch_config['InstanceType'],
        'aws/class': launch_config['InstanceType'][0],
        'aws/ami-id': launch_config['ImageId'],
        'aws/region': region
    }
    # custom selectors
    for tag_data in tags_data:
        if tag_data['Key'].startswith('kube/'):
            selectors[tag_data['Key'][5:]] = tag_data['Value']

    # adding kube label counterparts
    selectors['beta.kubernetes.io/instance-type'] = selectors['aws/type']
    selectors['failure-domain.beta.kubernetes.io/region'] = selectors['aws/region']

    return selectors
```

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

Normally resources craeted by kubernetes are placed under `kube-system` namespace.

```bash
$ kubectl create -f secret.yaml

# secret "autoscaler" created
```

Validate the result.

```bash
$ kubectl get secrets --namespace=kube-system

# NAME                  TYPE                                  DATA      AGE
# autoscaler            Opaque                                2         2d
# default-token-sovzd   kubernetes.io/service-account-token   3         3d
```

You can follow the instructions on the [kubernetes document](http://kubernetes.io/docs/user-guide/namespaces/) to set the namespace preference so its filter could be skipped.

### Run the autoscaler

The document provides a yaml example to run the autoscaler as a Replication Controller. I made some customization based on the example.

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: autoscaler
  namespace: kube-system
  annotations:
    scheduler.alpha.kubernetes.io/affinity: >
      {
        "nodeAffinity": {
          "requiredDuringSchedulingIgnoredDuringExecution": {
            "nodeSelectorTerms": [
              {
                "matchExpressions": [
                  {
                    "key": "gpu",
                    "operator": "NotIn",
                    "values": ["true"]
                  }
                ]
              }
            ]
          }
        }
      }
spec:
  replicas: 1
  selector:
    app: autoscaler
  template:
    metadata:
      labels:
        app: autoscaler
        openai/do-not-drain: "true"
    spec:
      containers:
      - name: autoscaler
        image: quay.io/openai/kubernetes-ec2-autoscaler
        env:
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              name: autoscaler
              key: aws-access-key-id
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: autoscaler
              key: aws-secret-access-key
        - name: PYKUBE_KUBERNETES_SERVICE_HOST
          value: 100.64.0.1
        command:
            - python
            - main.py
            - --regions
            - ap-northeast-1
            - --cluster-name
            - apnortheast.k8s.linkernetworks.com
            - -vvv
            - --type-idle-threshold
            - "0"
            - --sleep
            - "30"
        imagePullPolicy: Always
      restartPolicy: Always
      dnsPolicy: Default
      # nodeSelector: 
      #   key/value pairs match to your node labels
 ```

* Change the **namespace** to `kube-system`.
* Make sure the **keys** of AWS credential environment variables in your kubernetes secret are identical to `aws-access-key-id` and `aws-secret-access-key`. 
* Modify the value of `PYKUBE_KUBERNETES_SERVICE_HOST` to the IP address of your kubernetes cluster IP. You can have it by `kubectl get services --namespaces=default` command.
  ```bash
  # NAMESPACE     NAME         CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
  # default       kubernetes   100.64.0.1    <none>        443/TCP         2d
  ```
* Remove Slack and Datadog integration if you are not using them.
* Modifiy the **command** so that `region` and `--cluster-name` match yours.
* Use **nodeSelector** or **node affinity** (see [here](http://kubernetes.io/docs/user-guide/node-selection/)) to specify which node group the autoscaler runs on. You can find all labels bound to the nodes by `kubectl get nodes --show-labels`.

`kc get pods --namespace=kube-system -l app=autoscaler` to see if the autoscaler is actually up.

```bash
# NAME               READY     STATUS    RESTARTS   AGE
# autoscaler-4fa7x   1/1       Running   0          3h
```

You can also verify if the autoscaler are running on the desired node group via command `kc describe pod autoscaler-4fa7x | grep Node`.

```bash
# Node:    ip-172-20-47-134.ap-northeast-1.compute.internal/172.20.47.134
```

### Testing

At this stage, you are successfully running up the autoscaler. Next, we are going to check if the autoscaler helps us to scale out nodes to handle bursty workloads. 

The idea here is simple, we just need to create a large number of pods at once (to simulate the bursty work load) so that the autoscaler detects scheduled pods in pending state and scales out instances. 

For cost and easy to test reason, you might want to change your machine type of instance group to a less powerful one. E.g. t2.medium (Take a look at `capacity.json` to see what instance types are supported by default).

And then use kubernetes's **deployment** to create pods.

```yaml
# kubectl create -f nginx-deployment.yaml

# nginx-deployment.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 200
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
      nodeSelector:
        gpu: true
```

This will create 200 replicas of pods running nginx at once. You have to define the `nodeSelector` with right labels so the pods are scheduled to running on the desired nodes.

Try `kubectl rollout status deployment/nginx` and `kubectl get pods --namespace=kube-system -l app=nginx` to watch the rollout status and the state of pods respectively.

```bash
$ kubectl rollout status deployment/nginx

# Waiting for rollout to finish: 0 of 100 updated replicas are available...
# Waiting for rollout to finish: 1 of 100 updated replicas are available...
# Waiting for rollout to finish: 2 of 100 updated replicas are available...
# Waiting for rollout to finish: 3 of 100 updated replicas are available...
# Waiting for rollout to finish: 4 of 100 updated replicas are available...
# Waiting for rollout to finish: 5 of 100 updated replicas are available...
# Waiting for rollout to finish: 6 of 100 updated replicas are available...
# Waiting for rollout to finish: 7 of 100 updated replicas are available...
# Waiting for rollout to finish: 8 of 100 updated replicas are available...
# ...

$ kubectl get pods --namespace=kube-system -l app=nginx

# NAME                     READY     STATUS    RESTARTS   AGE
# nginx-2193136555-06hc2   1/1       Running   0          37m
# nginx-2193136555-0ap6v   1/1       Running   0          37m
# nginx-2193136555-0h2bx   1/1       Running   0          40m
# nginx-2193136555-16gv5   1/1       Running   0          37m
# nginx-2193136555-18qwt   1/1       Running   0          40m
# nginx-2193136555-1l5z3   1/1       Running   0          37m
# nginx-2193136555-1riut   1/1       Running   0          40m
# nginx-2193136555-1yl5r   1/1       Running   0          40m
# nginx-2193136555-1yqns   1/1       Running   0          40m
# nginx-2193136555-2dgfo   1/1       Running   0          37m
# nginx-2193136555-2doxy   1/1       Running   0          37m
# nginx-2193136555-2eurc   1/1       Running   0          40m
# nginx-2193136555-2j8m5   1/1       Running   0          37m
# nginx-2193136555-31dtw   1/1       Running   0          37m
# ...
```

If everything goes well, you should see instances are created to handle the workload.

```bash
$ kubectl logs -f autoscaler-4fa7x

# ...
# 2017-01-12 08:48:46,190 - autoscaler.cluster - INFO - KubePod(kube-system, nginx-2193136555-uxf6l) is pending ({"gpu": "true"})
# 2017-01-12 08:48:46,190 - autoscaler.cluster - INFO - KubePod(kube-system, nginx-2193136555-w9a4e) is pending ({"gpu": "true"})
# 2017-01-12 08:48:46,190 - autoscaler.cluster - INFO - KubePod(kube-system, nginx-2193136555-wb154) is pending ({"gpu": "true"})
# 2017-01-12 08:48:46,190 - autoscaler.cluster - INFO - KubePod(kube-system, nginx-2193136555-x0rav) is pending ({"gpu": "true"})
# 2017-01-12 08:48:46,190 - autoscaler.cluster - INFO - ========= Scaling for {"gpu": "true"} ========
# 2017-01-12 08:48:46,190 - autoscaler.cluster - DEBUG - pending: [KubePod(kube-system, nginx-2193136555-16gv5), KubePod(kube-system, nginx-2193136555-1l5z3), KubePod(kube-system, nginx-2193136555-2j8m5), KubePod(kube-system, nginx-2193136555-3meox), KubePod(kube-system, nginx-2193136555-55e3s)]
# 2017-01-12 08:48:46,191 - autoscaler.cluster - DEBUG - group: AutoScalingGroup(gpu-nodes.apnortheast.k8s.linkernetworks.com, {"aws/ami-id": "ami-999af2fe", "aws/class": "t", "aws/region": "ap-northeast-1", "aws/type": "t2.medium", "beta.kubernetes.io/instance-type": "t2.medium", "failure-domain.beta.kubernetes.io/region": "ap-northeast-1", "gpu": "true"})
# 2017-01-12 08:48:46,192 - autoscaler.cluster - DEBUG - units_needed: 7
# 2017-01-12 08:48:46,192 - autoscaler.cluster - DEBUG - units_requested: 7
# 2017-01-12 08:48:46,192 - autoscaler.autoscaling_groups - INFO - Desired 8, currently at 1
# 2017-01-12 08:48:46,192 - autoscaler.autoscaling_groups - INFO - Kube node: 1 schedulable, 0 unschedulable
# 2017-01-12 08:48:46,192 - autoscaler.autoscaling_groups - INFO - ASG: AutoScalingGroup(gpu-nodes.apnortheast.k8s.linkernetworks.com, {"aws/ami-id": "ami-999af2fe", "aws/class": "t", "aws/region": "ap-northeast-1", "aws/type": "t2.medium", "beta.kubernetes.io/instance-type": "t2.medium", "failure-domain.beta.kubernetes.io/region": "ap-northeast-1", "gpu": "true"}) new_desired_capacity: 8
# {"units_requested": 7, "_log_streaming_target_mapping": "kubernetes-ec2-autoscaler", "asg": "AutoScalingGroup(gpu-nodes.apnortheast.k8s.linkernetworks.com, {\"aws/ami-id\": \"ami-999af2fe\", \"aws/class\": \"t\", \"aws/region\": \"ap-northeast-1\", \"aws/type\": \"t2.medium\", \"beta.kubernetes.io/instance-type\": \"t2.medium\", \"failure-domain.beta.kubernetes.io/region\": \"ap-northeast-1\", \"gpu\": \"true\"})", "pod_name": "kube-system/nginx-2193136555-16gv5", "time": "2017-01-12T08:48:46.243862", "message": "scale", "pod_id": "ecf6b43f-d8a3-11e6-badd-064a8e8c4235"}
# ...
```

![]({{site.baseurl}}/images/autoscaler-scale-out.png)

Phew, not so easy right? I guess some pull requests are needed for the project to improve the documentation :)
