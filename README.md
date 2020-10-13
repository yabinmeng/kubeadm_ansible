# Oveview

Previously I wrote a [tutorial](https://github.com/yabinmeng/dseutilities/blob/master/documents/tutorial/k8s/kubeadm_install.md) about how to set up a simple K8s cluster using kubeadm. 

In this repo., I'm going to automate the K8s cluster setup procedure using Ansible. The automation of some other K8s relevant procedures are also automated in this repo, as summarized below:

| Ansible Playbook | Description |
|------------------|-------------|
| setup_k8s_cluster.yaml| Set up single-master K8s cluster |
| teardown_k8s_cluster.yaml | Tear down a K8s cluster with proper clean up |
| << to be added >> | ... ... |

## Testing Environment

* Ansible version: 2.10.2
* Python version: 3.8.5 
* K8s version: 1.18.9 (configurable throung Ansible variable)
* K8s Calico version: 3.16 (configurable through Ansible variable)

## Usage

```
$ ansible-playbook -i hosts.ini <playbook_yaml> --private-key=<ssh_private_key> -u <ssh_user>
```