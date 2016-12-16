---
layout: post
title:  How to install DC/OS on GCE
date:   2016-12-13T23:00:00+8:00
---

Follow the instructions in the DC/OS [installation guide](https://dcos.io/docs/1.8/administration/installing/cloud/gce/) and beware of following points. 

* Login to the gcloud first via command `gcloud auth login` or you'll got permission denied when creating instances.
* Configurations in `dcos_gce/group_vars/all`

  ```yaml
    project: #should be GCE project id instead of project name.
    bootstrap_public_ip: #should be the internal IP of an instance instead of its external IP.
    login_name: #should be the username of the key pair
    subnet: #can be default. If a custom network is used, make sure that instances can comunicate with each other internally.
    agent-machine-type: #should not be changed to n1-standard-1 or navistar component will be unstable and unhealthy
  ```

* Add `--user` when executing the ansible playbook 

  ```bash
    ansible-playbook -i hosts install.yml --user {username of key pair}
  ```
* Open port 80 and 8080 on master node to access DC/OS and Marathon UI respectively.

* **These have only be tested for DC/OS 1.8.**
