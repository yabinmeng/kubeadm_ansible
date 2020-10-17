# Oveview

Previously I wrote a [tutorial](https://github.com/yabinmeng/dseutilities/blob/master/documents/tutorial/k8s/kubeadm_install.md) about how to set up a simple K8s cluster using kubeadm.

In this repo., I'm going to demonstrate how to automate the K8s cluster setup procedure using Ansible. The focus of this repo. is on establishing a K8s cluster using kubeadm on a set of host machines. The automation procedure of creating K8s clusters on specific cloud platforms like Azure, GCP, AWS will be covered in a different repo.

## Testing Environment

* Ansible version: 2.10.2
* Python version: 3.8.5
* K8s version: 1.18.9 (configurable throung Ansible variable)
* K8s Calico version: 3.16 (configurable through Ansible variable)

# Usage Description

## Ansible Playbook Summary

### Setup K8s Cluster 

The Ansible playbook **setup_k8s_cluster.yaml** file is used to create a K8s cluster using kubeadm. At high level, the following tasks are covered in this playbook:

**1). On All Host Machines**

* Pre-condition check and preparation work, such as: 
  * Make sure each host has unique MAC address)
  * Enable Linux br_netfilter module
  * Enable bridged traffic in IP tables
  * Disable SWAP (if not disabled)
  * Install Python Openshfit Client
* Install Docker runtime
  * Change cgroup driver of Docker runtime from default "cgroupfs" to "systemd" for improved system stability 
* Install **kubeadm**, **kubelet**, and **kubectl**

**2). On K8s Master Node Host Machine**

* Initialize K8s Master using "kubeadm init", with a specific Pod network CIDR range
  * Set up proper kubeconfig to allow non-root exeuction of "kubectl" commands
* Install **calico** K8s network CNI
* Calculate K8s token and certificate hash value and store them in Ansoible facts
* Install [K8s Dashboard](https://github.com/kubernetes/dashboard) (for K8s admin purpose)

**3). On K8s Worker Node Host Machines**
* Join K8s worker nodes in the cluster
* The playbook also copies kubeconfig file to each worker node so we can run "kubectl" commands on worker nodes as well.

### Clean Up K8s Cluster

The Ansible playbook **teardown_k8s_cluster.yaml** file is used to create the K8s cluster that is created using the setup playbook. At high level, this play book does the following tasks:

**1). On K8s Master Node Host Machine**

* Remove K8s resouces from calico CNI and K8s dashboard
* Drain K8s worker nodes

**2). On K8s Worker Node Host Machines**

* Reset **kubeadm** state from "kubeadm join"

**3). On K8s Master Node Host Machine**

* Delete K8s worker nodes
* Reset "kubeadm" state from "kubeadm init"

**4). On All Host Machines**

* Reset IPTables and IPSV

## Ansible K8s Module

### Ansible kubernete community collection

Some of the K8s related tasks in this repo are executed using [Ansible K8s plugin](https://docs.ansible.com/ansible/latest/collections/community/kubernetes/k8s_module.html). This plugin is part of the [community.kubernete](https://galaxy.ansible.com/community/kubernetes?extIdCarryOver=true&sc_cid=701f2000001OH6uAAG) collection. In order to use this plug in, we need to first install it by running the following command

```
  ansible-galaxy collection install community.kubernetes
```

### Openshift Python Client

In order to run this plugin, the following prerequisits need to be satisfied.

```
* python >= 2.7
* PyYAML >= 3.11
* openshift >= 0.6
```

By default, when Ansible is installed, PyYAML is also installed. But ["openshift" python client](https://pypi.org/project/openshift/) needs to be installed explicitly, which is handeled in this repo as one task as part of **pre_k8s_install** role.

```
- name: Install Python openshift module
  pip:
    name: openshift
    extra_args: 
      --ignore-installed
```

### Using K8s Module

Once installed, we need to specify the collection name before using it in an Ansible task. As below:
```
- hosts: xxx
  ... ...
  collections:
    - community.kubernetes
  environment:
    KUBECONFIG: <path/to/.kube/config>
  tasks:
    - name: K8s task
      k8s: 
       ... ...
```

## Run playbook

Use the following command to run the Ansible playbooks

``` script=bash
  ansible-playbook -i hosts.ini <playbook_yaml> --private-key=<ssh_private_key> -u <ssh_user>
```

# K8s Dashboard

K8s dashboard is a general purpose web UI for applications running ina  K8s cluster and as well as the cluster itself.

When setting up a K8s cluster using this repo (**setup_k8s_cluster.yaml**), we also lauched K8s resources (Pods, secretes, services, et.c) that are needed for K8s dashboard in the cluster in a dedicated K8s namespace **kubernetes-dashboard**. For example, the following K8s dashboard services are created in the cluster.

```
$ kubectl get services -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
dashboard-metrics-scraper   ClusterIP   10.103.101.117   <none>        8000/TCP   127m
kubernetes-dashboard        ClusterIP   10.102.204.25    <none>        443/TCP    127m
```

## Accessing K8s Dashboard

There are various ways to access K8s dashboard web UI. For more details, please check K8s documentation at [here](https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/README.md).

### Kubectl Proxy Access

As we can see above, the default K8s dashboard deployment creates a service of "**ClusterIP**" type, which only allows for internal access within the K8s cluster. But we can access it externally through kubectl proxy. The procedure is as below:

* Copy proper kube config file from a K8s node

```
$ scp -i <ssh_private_key> <ssh_user>@<k8s_node_name>:~/.kube/config kubeconfig
```

* Run "kubectl proxy" command on the local desktop machine

```
$ kubectl --kubeconfig=./kubeconfig proxy
Starting to serve on 127.0.0.1:8001
```

Now we can access the dashboard webUI through the following URL:
* http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

**NOTE** that Kubectl proxy will convert the HTTP based local access to HTTPS based remote access to the K8s apiserver, which requires to enter login information, as below:

<img src="https://github.com/yabinmeng/kubeadm_ansible/blob/master/screenshots/login.png width=600">

