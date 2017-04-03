# AWS deployment

## Prerequisites

* Install Ansible
* Install Kops (https://github.com/kubernetes/kops) (see below)
* Install kubectl (see below)
* Create secure Amazon S3 bucket, which the Kops tool will use as the storage for cluster configurations. (see below) **The bucket will contain also the access details for the clusters configured with Kops. It should be secured accordingly.**

### Kubectl installation

Playbook `install-kubectl.yaml` can be used for installing the latest version of kubectl on Linux or MacOS. To install it run
```
ansible-playbook install-kubectl.yaml
```

### Kops installation

Playbook `install-kops.yaml` can be used for installing the latest version of Kops utility on Linux or MacOS. To install it run
```
ansible-playbook install-kops.yaml
```

### S3 bucket for state store

Playbook `create-state-store.yaml` can be used to create the S3 bucket to store the Kops state. To install it run
```
ansible-playbook create-state-store.yaml
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

Additionally to the Kubernetes cluster it self, an AWS Lambda function will be created which will run periodically to tag all resources creating by Kops and by Kubernetes. It will use following tags:
* Creator
* Owner
* Application
* CostCenter
* Product
* Confidentiality
* Environment

The tags are configured in also in `group_vars/all/vars.yaml` using following variables:

| Option | Explanation | Example |
|--------|-------------|---------|
| `tag_creator` | Value for the Creator tag | `scholzj` |
| `tag_owner` | Value for the Owner tag | `scholzj` |
| `tag_application` | Value for the Application tag | `MyApp1` |
| `tag_costcenter` | Value for the CostCenter tag | `123456` |
| `tag_product` | Value for the Product tag | `Risk` |
| `tag_confidentiality` | Value for the Confidentiality tag | `StrictlyConfidential` |
| `tag_environment` | Value for the Environment tag | `DEV` |

Additionally to these tags, all resources without the `Name` tag will be named according to the cluster name (e.g. `kubernetes.my-cluster.com-resource`)

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

## Install and deleting the tagging lambda function

The AWS Lambda function for tagging of resources (the related IAM and CloudWatch objects) can be also installed and uninstalled separately. To install it run:
```
ansible-playbook install-lambda.yaml
```

To uninstall it run:
```
ansible-playbook uninstall-lambda.yaml
```
