# Kubernetes setup on Amazon AWS using Kops and Ansible

This repository contains tooling for deploying Kubernetes cluster in Amazon AWS using the [Kops](https://github.com/kubernetes/kops) tool. Kops is a great tool if you want to setup HA cluster and don't require too much flexibility. If you prefer flexibility instead of HA setup you should have a look at [another repsoitory](https://github.com/scholzj/aws-kubernetes) where I have Kubernetes setup implemented using Terraform and Kubeadm tool. I have also a [special *minikube* single node installation](https://github.com/scholzj/aws-minikube).

<!-- TOC depthFrom:2 -->

- [Updates](#updates)
- [Installing the cluster](#installing-the-cluster)
    - [Install Ansible](#install-ansible)
    - [Kubectl installation](#kubectl-installation)
    - [Kops installation](#kops-installation)
    - [AWS Credentials](#aws-credentials)
    - [S3 bucket for state store](#s3-bucket-for-state-store)
    - [Install Kubernetes cluster](#install-kubernetes-cluster)
    - [Install add-ons (optional)](#install-add-ons-optional)
    - [Install ingress (optional)](#install-ingress-optional)
    - [Install the tagging lambda function (optional)](#install-the-tagging-lambda-function-optional)
- [Updating the cluster](#updating-the-cluster)
- [Deleting the cluster](#deleting-the-cluster)
    - [Deleting the tagging lambda function](#deleting-the-tagging-lambda-function)
- [Frequently Asked Questions](#frequently-asked-questions)
    - [How to access Kuberntes Dashboard](#how-to-access-kuberntes-dashboard)

<!-- /TOC -->

## Updates

* *16.4.2018* Update to Kops 1.9 and Kubernetes 1.9, update addons, remove Storage Class (installed by Kops automatically) and Route53 addon (replaced by ExternalDNS addon)
* *2.1.2018* Add support for public and private topologies
* *9.12.2017* Update to Kops 1.8 and Kubernetes 1.8
* *28.11.2017* Update addon versions
* *14.10.2017* Update to Kops 1.7.1 which fixes [CVE-2017-14491](https://github.com/kubernetes/kops/blob/master/docs/advisories/cve_2017_14491.md)
* *22.8.2017* Update to Kops 1.7 and Kubernetes 1.7

## Installing the cluster

The cluster can be deployed from your local host (tested with MacOS and Linux) by following the steps described below. If you cannot install Ansible, kubectl or kops on your local PC or in case your local PC is running Windows, you can create a EC2 host in Amazon AWS and run the installation from this host.

### Install Ansible

Download and install Ansible - you can follow the guide from [Ansible website](http://docs.ansible.com/ansible/latest/intro_installation.html).

### Kubectl installation

Install the latest version of `kubectl` on Linux or MacOS:
```
ansible-playbook install-kubectl.yaml
```
You may need either `--ask-sudo-pass` or `ansible_become_pass`


### Kops installation

Install the latest version of `Kops` utility on Linux or MacOS:
```
ansible-playbook install-kops.yaml
```
You may need either `--ask-sudo-pass` or `ansible_become_pass`

### AWS Credentials

Export the AWS credentials whih will be used to authenticate with Amazon AWS:
```
export AWS_ACCESS_KEY_ID="XXX"
export AWS_SECRET_ACCESS_KEY="XXX"
```

### S3 bucket for state store

Create S3 bucket to store where `Kops` will store its information:
```
ansible-playbook create-state-store.yaml
```

**The bucket will contain also the access details for the clusters configured with Kops. It should be secured accordingly.**

### Install Kubernetes cluster

Create the Kubernetes cluster using `Kops`:
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
| `master_zones` | List of availability zones in which the master nodes should be installed. Must be an odd number (1, 3 or 5) of zones (at least 3 zones are needed for AWS setup accross availability zones). If not specified, `aws_zones` will be used instead | `eu-west-1a,eu-west-1b,eu-west-1c` |
| `dns_zone` | Name of the Rote 53 hosted zone. Must be reflected in the cluster name. | `my-cluster.com` |
| `network_cidr` | A new VPC will be created. It will use the CIDR specified in this option. | `172.16.0.0/16` |
| `topology` | Defines whether the cluster should be deployed into private subnet (`private` - more secure) with bastion host or into public subnet (`public` - less secure). | `private` |
| `kubernetes_networking` | Defines which networking plugin should be used in Kubernetes. Tested with Calico only. | `calico` |
| `master_size` | EC2 size of the nodes used for the Kubernetes masters (and Etcd hosts) | `m4.large` |
| `master_count` | Number of EC2 master hosts. | `3` |
| `master_volume_size` | Size of the master disk volume in GB. | `50` |
| `master_max_price` | Optional, max price for master spot instances. | `0.05` |
| `master_profile` | Optional, custom master IAM role. | `arn:aws:iam::1234567890108:instance-profile/kops-custom-master-role` |
| `node_size` | EC2 size of the nodes used as workers. | `m4.large` |
| `node_count` | Number of EC2 worker hosts (initial count). | `6` |
| `node_volume_size` | Size of the node disk volume in GB. | `50` |
| `node_max_price` | Optional, max price for node spot instances. | `0.05` |
| `node_profile` | Optional, custom node IAM role. | `arn:aws:iam::1234567890108:instance-profile/kops-custom-node-role` |
| `node_autoscaler_min` | Minimum number of nodes (for the autoscaler). | `3` |
| `node_autoscaler_max` | Maximum number of nodes (for the autoscaler). | `6` |
| `base_image` | Image used for all the instances | `kope.io/k8s-1.11-debian-stretch-amd64-hvm-ebs-2018-08-17` |
| `kubernetes_version` | Version of kubernetes which should be used. | `1.11` |
| `iam.allow_container_registry` | Optional, boolean to allow read access to Amazon ECR | `true` |
| `iam.legacy` | Optional, boolean to use the legacy IAM privileges | `false` |


Additionally to the Kubernetes cluster it self, an AWS Lambda function may be created which will run periodically to tag all resources creating by Kops and by Kubernetes. To use it, a tag must be specified :
```
ansible-playbook create.yaml --tags "use_lambda"
``` 
It will use following tags:
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
| `tag_creator` | Value for the Creator tag | `MyName` |
| `tag_owner` | Value for the Owner tag | `MyName` |
| `tag_application` | Value for the Application tag | `MyApp` |
| `tag_costcenter` | Value for the CostCenter tag | `1234` |
| `tag_product` | Value for the Product tag | `MyProduct` |
| `tag_confidentiality` | Value for the Confidentiality tag | `StrictlyConfidential` |
| `tag_environment` | Value for the Environment tag | `Development` |

Additionally to these tags, all resources without the `Name` tag will be named according to the cluster name (e.g. `kubernetes.my-cluster.com-resource`)

### Install add-ons (optional)

Currently, the supported add-ons are:
* Kubernetes dashboard
* Heapster for resource monitoring
* External DNS
* Cluster Autoscaler

To install the add-ons run
```
ansible-playbook addons.yaml
```

### Install ingress (optional)

Ingress can be used route inbound traffic from the outside of the Kubernetes cluster. It can be used for SSL termination, virtual hosts, load balancing etc. For more details about ingress, go to [Kubernetes website](https://kubernetes.io/docs/concepts/services-networking/ingress/).

To install ingress controller based on Nginx, run
```
ansible-playbook ingress.yaml
```

### Install the tagging lambda function (optional)

The AWS Lambda function can be used for tagging of resources created by the Kubernetes installation. To install it run:
```
ansible-playbook install-lambda.yaml
```

## Updating the cluster

All updates to the running Kubernetes cluster can be done directly using `Kops`. The Ansible playbooks from this project only simplify the initial setup.

## Deleting the cluster

To delete the cluster export the AWS credentials:
```
export AWS_ACCESS_KEY_ID="XXX"
export AWS_SECRET_ACCESS_KEY="XXX"
```

And run:
```
ansible-playbook delete.yaml
```

### Deleting the tagging lambda function

If you installed the AWS Lambda for tagging, you can remove it using this command:
```
ansible-playbook uninstall-lambda.yaml
```

## Frequently Asked Questions

### How to access Kuberntes Dashboard

The Kubernetes Dashboard addon is by default not exposed to the internet. This is intentional for security reasons (no authentication / authorization) and to save costs for Amazon AWS ELB load balancer.

You can access the dashboard easily fro any computer with installed and configured `kubectl`:
1) From command line start `kubectl proxy`
2) Go to your browser and open [http://127.0.0.1:8001/ui](http://127.0.0.1:8001/ui)
