# AWS deployment

## Prerequisites

* Install Kops (https://github.com/kubernetes/kops) - currently the master is supported. 1.4.x will not work.
* Install Ansible
* Install kubectl (see bellow)
* Create secure Amazon S3 bucket, which the Kops tool will use as the storage for cluster configurations. **The bucket will contain also the access details for the clusters configured with Kops. Tt should be secured accordingly.**

### Kubectl installation

Playbook `install-kubectl.yaml` can be used for installing the latest version of kubectl on Linux or MacOS. To install it run
```
ansible-playbook install-kubectl.yaml
```

## Install Kubernetes cluster

The Kubernetes cluster can be created by running
```
ansible-playbook create.yaml
```

The main configuration of the cluster is in the variables in `group_vars/all/vars.yaml`. Following table shows the different options.

| Option | Explanation | Example |
|--------|-------------|---------|
| `cluster_name` | Name of the cluster which should be created. The name has to end with the domain name of the DNS zone hosted in Route 53. | `kubernetes.my-cluster.com` |
| `state_store` | Name of the Amazon S3 bucket which should be used as a Kops state store. It should start with `s3://`. | `s3://kops-state-store` |
| `ssh_public_key` | Path to the public part of the SSH key, which should be used for the SSH access to the Kubernetes hosts | `~/.ssh/id_rsa.pub` |
| `aws_region` | AWS region where the cluster should be installed. | `eu-west-1` |
| `aws_zones` | List of availability zones in which the cluster should be installed. Must be an odd number (1, 3 or 5) of zones (at least 3 zones are needed for AWS setup accross availability zones). | `eu-west-1a,eu-west-1b,eu-west-1c` |
| `dns_zone` | Name of the Rote 53 hosted zone. Must be reflected in the cluster name. | `my-cluster.com` |
| `network_cidr` | A new VPC will be created. It will use the CIDR specified in this option. | `172.35.0.0/16` |
| `kubernetes_networking` | Defines which networking plugin should be used in Kubernetes. Tested with Calico only. | `calico` |
| `master_size` | EC2 size of the nodes used for the Kubernetes masters (and Etcd hosts) | `t2.small` |
| `node_size` | EC2 size of the nodes used as workers. | `t2.small` |
| `node_count` | Nomber of EC2 worker hosts. | `6` |

## Updating Kubernetes cluster

The Kubernetes cluster setup is done using the Kops tool only. All updates to it can be done using Kops. The Ansible playbooks from this project only simplify the initial setup.

## Delete Kubernetes cluster

To delete the cluster run
```
ansible-playbook delete.yaml
```

## Install add-ons

Currently, the supported add-ons are:
* Kubernetes dashboard
* Heapster for resource monitoring
* Storage class for automatic provisioning of persisitent volumes

To install the add-ons run
```
ansible-playbook addons.yaml
```
