# Tech Talk - k0smotron - Part 2 - ClusterAPI

This repository contains all necessary code snippets and a guide to replicate the demo shown in the Tech Talk [Expanding Horizons: k0smotron's Integration with Cluster API and Beyond](https://www.mirantis.com/labs/expanding-horizons-k0smotron-s-integration-with-cluster-api-and-beyond).

### Versions
The following version will be used in this repository: 

- k0s (v1.27.4)
- k0smotron (v0.5.1)
- AWS Cloud Controller Manager
- AWS EBS CSI Driver
- AWS EBS StorageClass
- ClusterAPI (v1.5.0)

### Preperation

Like in the Tech Talk Demo we will use AWS to create the k0s Management Cluster. 
For preparation, get and apply your AWS credentials in the terminal, so we can use Terraform to create necessary resources.
> **NOTE**: The k0s Management Cluster can run in any infrastructure / cloud (private/public). In this demo we will use AWS.

## Management Cluster Installation
Please check the ```terraform.tfvars``` and adjust the AMI in ```main.tf``` as needed. 

``` shell
terraform init

terraform apply -auto-approve

terraform output -raw k0s_cluster | k0sctl apply --no-wait --debug --config -

terraform output -raw k0s_cluster | k0sctl kubeconfig --config -
```

et voilà, a k0s cluster with 1 controller, 1 worker, integration into AWS and k0smotron.

> **NOTE**: If you want to create a high available cluster, please set the controller_count to 3. This will automatically create a ELB.

## AWS - Create a k0s Cluster with EC2 instances as worker nodes

Please follow this (guide)[aws/guide.md] to install the ClusterAPI Controller and the AWS Infrastructure Controller into the management cluster.

Happy testing! 

## Hetzner - Create a k0s Cluster with Hetzner Cloud Machines as worker nodes

Please follow this (guide)[hetzner/guide.md] to install the ClusterAPI Controller and the Hetzner Infrastructure Controller into the management cluster.

Happy testing! 

## Notes

### Configuration
This is a simple demo of k0s on AWS, k0smotron and k0s control planes in the management cluster. If you need more advanced configurations please check our documentation of k0s and k0smotron: 
- k0s Docs -> https://docs.k0sproject.io/
- k0smotron Docs -> https://docs.k0smotron.io/

## Day 2 Operations
We are planning a third part which will cover Day 2 Cluster Operations. Please have a look for the third part of the k0smotron series at https://www.mirantis.com/labs/ and the third repository that will be available shortly after the Tech Talk.
Otherwise you can find a detailed guide on how this works [here](https://docs.k0smotron.io/stable/cluster-api/).