# ClusterAPI Provider AWS (CAPA)

With the management cluster with k0smotron, AWS CCM and AWS CSI EBS we are able to create k0s control planes but the ClusterAPI (CAPI) components are missing. By using `clusterctl`, we can initialize the CAPI cluster and already specify the infrastructure provider.

>After creating the management cluster place its kubeconfig under `~/.kube/config` so other tools such as `clusterctl` can discover and interact with it.

### Prepare the infra provider

Prior to initiating a cluster, the configuration of the infrastructure provider is necessary. Each provider comes with its own unique set of prerequisites.

> Note: The following commands are for Mac users. Please download the (fitting binary)[https://github.com/kubernetes-sigs/cluster-api-provider-aws/releases] and make it executable if you are using Windows or Linux. 

The AWS infrastructure provider requires the `clusterawsadm` tool to be installed:
``` bash
curl -L https://github.com/kubernetes-sigs/cluster-api-provider-aws/releases/download/v2.2.0/clusterawsadm-darwin-amd64 -o clusterawsadm
chmod +x clusterawsadm
sudo mv clusterawsadm /usr/local/bin
```

`clusterawsadm` is a CLI to check requirements and automatically bootstrap the IAM resources needed for ClusterAPI. 

Start by setting up environment variables defining the AWS account to use, if these are not already defined:
``` bash
export AWS_REGION=<your-region-eg-us-east-1>
export AWS_ACCESS_KEY_ID=<your-access-key>
export AWS_SECRET_ACCESS_KEY=<your-secret-access-key>
```
If you are using multi-factor authentication, you will also need:
``` bash
export AWS_SESSION_TOKEN=<session-token>
```
`clusterawsadm` uses these details to create a CloudFormation stack in your AWS account
with the needed IAM resources:
``` bash
clusterawsadm bootstrap iam create-cloudformation-stack
```
The credentials should also be encoded and stored as a kubernetes secret.
``` bash
export AWS_B64ENCODED_CREDENTIALS=$(clusterawsadm bootstrap credentials encode-as-profile)
```
> Note: The credentials will expire if you are using Multi-Factor Authentication. This can cause the AWS Controller Manager to fail when trying to change any resources in AWS. Please make sure to renew the credentials and run the command above once again. 

### Initialize the management cluster
To initialize the management cluster with the latest released version of the providers and the infrastructure of your choice:

``` bash
clusterctl init -i aws
```

Now that we've created everything that is neccessary we can create a cluster with the k0s control plane running in the management cluster and the EC2 instance worker nodes.

Here is an example that you need to adjust: 

``` yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-orange # Namespace in which we want to deploy the child cluster
spec: {}
status: {}
---
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: mirantis-labs-aws-cluster # Cluster Name
  namespace: team-orange
spec:
  clusterNetwork:
    pods:
      cidrBlocks: [10.244.0.0/16]
    services:
      cidrBlocks: [10.96.0.0/12]
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: K0smotronControlPlane
    name: mirantis-labs-aws-cluster # Reference to the k0s control plane we are creating in the management cluster
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
    kind: AWSCluster
    name: mirantis-labs-aws-cluster # Reference to the AWSCluster CRD that trigger the AWS Controller Manager
---
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: K0smotronControlPlane
metadata:
  name: mirantis-labs-aws-cluster # Cluster Name 
  namespace: team-orange
spec:
  k0sVersion: v1.27.2-k0s.0 # https://github.com/k0sproject/k0s/releases
  persistence:
    type: emptyDir
  service:
    type: LoadBalancer
    apiPort: 6443
    konnectivityPort: 8132
---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSCluster
metadata:
  name: mirantis-labs-aws-cluster # Cluster Name 
  namespace: team-orange
  annotations:
    cluster.x-k8s.io/managed-by: k0smotron # Specifies the dependency to k0smotron
spec:
  region: eu-central-1 # Your AWS Region
  sshKeyName: jhennig-ssh-key # Your SSH Key
  network:
    vpc:
      id: vpc-095897d5b42a2181e # Machines will be created in this VPC (needs to be in the AWS Region specified)
    subnets:
      - id: subnet-07d4ba06edf1f25c0 # Machines will be created in this Subnet (needs to be in the AWS Region specified)
        availabilityZone: eu-central-1a
---
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: mirantis-labs-aws-cluster-md 
  namespace: team-orange
spec:
  clusterName: mirantis-labs-aws-cluster # Cluster Name 
  replicas: 3 # Specify how many worker nodes you want to create
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: mirantis-labs-aws-cluster # Cluster Name 
      pool: ec2-worker-small # Can be any name 
  template:
    metadata:
      labels:
        cluster.x-k8s.io/cluster-name: mirantis-labs-aws-cluster # Cluster Name 
        pool: ec2-worker-small # Can be any name 
    spec:
      clusterName: mirantis-labs-aws-cluster # Cluster Name 
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: K0sWorkerConfigTemplate
          name: mirantis-labs-aws-cluster-mc # Reference to the k0s worker node config
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
        kind: AWSMachineTemplate
        name: mirantis-labs-aws-cluster-mt # Reference to aws ec2 instance machine config
---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSMachineTemplate
metadata:
  name: mirantis-labs-aws-cluster-mt
  namespace: team-orange
spec:
  template:
    spec:
      ami:
        # Replace with your AMI ID
        id: ami-04e601abe3e1a910f # Ubuntu 22.04 in eu-central-1 
      instanceType: m5.large
      iamInstanceProfile: nodes.cluster-api-provider-aws.sigs.k8s.io # Instance Profile created by `clusterawsadm bootstrap iam create-cloudformation-stack` earlier
      cloudInit:
        # Makes CAPI AWS use k0s bootstrap cloud-init directly and not via SSM
        # Simplifies the VPC setup as we do not need custom SSM endpoints etc.
        insecureSkipSecretsManager: true
      subnet:
        # Make sure this matches the failureDomain in the Machine, i.e. you pick the subnet ID for the AZ
        id: subnet-07d4ba06edf1f25c0
      additionalSecurityGroups:
        - id: sg-0820c3a5d09a3f97c # Needs to be belong to the subnet
      sshKeyName: jhennig-key
---
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: K0sWorkerConfigTemplate
metadata:
  name: mirantis-labs-aws-cluster-mc
  namespace: team-orange
spec:
  template:
    spec:
      version: v1.27.2+k0s.0
      # More details of the worker configuration can be set here
```

After your adjustments you are ready to deploy the cluster to AWS with:

```
kubectl apply -f 'mirantis-labs-aws-cluster.yaml' 
```

After applying the manifests to the management cluster and confirming the infrastructure readiness, allow a few minutes for all components to provision. Once complete, your command line should display output similar to this:

```shell
% kubectl get cluster,machine
NAME                                    PHASE         AGE   VERSION
cluster.cluster.x-k8s.io/k0s-aws-test   Provisioned   16m   

NAME                                         CLUSTER        NODENAME   PROVIDERID                                 PHASE         AGE   VERSION
machine.cluster.x-k8s.io/k0s-aws-test-md-0   k0s-aws-test              aws:///eu-central-1a/i-05f2de7da41dc542a   Provisioned   16m   
```

You can also check the status of the cluster deployment with `clusterctl`:
```shell
% clusterctl describe cluster  
NAME                                                   READY  SEVERITY  REASON                    SINCE  MESSAGE          
Cluster/k0s-aws-test                                   True                                       25m                      
├─ClusterInfrastructure - AWSCluster/k0s-aws-test                                                                             
├─ControlPlane - K0smotronControlPlane/k0s-aws-test-cp                                                                           
└─Workers                                                                                                                  
  └─Other
```

### Configuration
This is a simple demo of k0s on AWS, k0smotron and k0s control planes in the management cluster. If you need more advanced configurations please check our documentation of k0s and k0smotron: 
- k0s Docs -> https://docs.k0sproject.io/
- k0smotron Docs -> https://docs.k0smotron.io/