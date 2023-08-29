# ClusterAPI Provider Hetzner (CAPH)

With the management cluster with k0smotron, AWS CCM and AWS CSI EBS we are able to create k0s control planes but the ClusterAPI (CAPI) components are missing. By using `clusterctl`, we can initialize the CAPI cluster and already specify the infrastructure provider.

>After creating the management cluster place its kubeconfig under `~/.kube/config` so other tools such as `clusterctl` can discover and interact with it.

### Prepare the infra provider

Prior to initiating a cluster, the configuration of the infrastructure provider is necessary. Each provider comes with its own unique set of prerequisites.





### Initialize the management cluster
To initialize the management cluster with the latest released version of the providers and the infrastructure of your choice:

``` bash
clusterctl init -i hetzner
```

Now that we've created everything that is neccessary we can create a cluster with the k0s control plane running in the management cluster and the hetzner cloud instance worker nodes.

Here is an example that you need to adjust: 

``` yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-red # Namespace in which we want to deploy the child cluster
spec: {}
status: {}
---
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: mirantis-labs-hetzner # Cluster Name 
  namespace: team-red
spec:
  clusterNetwork:
    pods:
      cidrBlocks: [10.244.0.0/16]
    services:
      cidrBlocks: [10.96.0.0/12]
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: K0smotronControlPlane
    name: mirantis-labs-hetzner # Reference to the k0s control plane we are creating in the management cluster
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: HetznerCluster
    name: mirantis-labs-hetzner # Reference to the HetznerCluster CRD that trigger the Hetzner Controller Manager
---
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: K0smotronControlPlane
metadata:
  name: mirantis-labs-hetzner # Cluster Name
  namespace: team-red
spec:
  k0sVersion: v1.27.2-k0s.0 # https://github.com/k0sproject/k0s/releases
  persistence:
    type: emptyDir
  service:
    type: LoadBalancer
    apiPort: 6443
    konnectivityPort: 8132
---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: HetznerCluster
metadata:
  name: mirantis-labs-hetzner # Cluster Name
  namespace: team-red
  annotations:
    cluster.x-k8s.io/managed-by: k0smotron # Specifies the dependency to k0smotron
spec:
  controlPlaneLoadBalancer:
    enabled: false # Disables LB creation since we want to create the control plane in the management cluster
  controlPlaneEndpoint:
    host: "1.2.3.4" # Just a Placeholder - no changes needed
    port: 6443
  controlPlaneRegions:
    - fsn1 # Your Hetzner Region
  hetznerSecretRef:
    name: hetzner # Reference to the secret we've created earlier
    key:
      hcloudToken: hcloud
  sshKeys:
    hcloud:
    - name: jhennig@mirantis.com # Needs exist in the Hetzner Cloud project
---
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: mirantis-labs-hetzner-md
  namespace: team-red
  labels:
    nodepool: hetzner-worker-small
spec:
  clusterName: mirantis-labs-hetzner # Cluster Name
  replicas: 3 # Specify how many worker nodes you want to create
  selector:
    matchLabels: null
  template:
    metadata:
      labels:
        nodepool: hetzner-worker-small # Can be any name 
    spec:
      clusterName: mirantis-labs-hetzner # Cluster Name
      failureDomain: fsn1 # Your Hetzner Region
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: K0sWorkerConfigTemplate
          name: mirantis-labs-hetzner-mc # Reference to the k0s worker node config
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: HCloudMachineTemplate
        name: mirantis-labs-hetzner-mt # Reference to hetzner cloud machine config
---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: HCloudMachineTemplate
metadata:
  name: mirantis-labs-hetzner-mt
  namespace: team-red
spec:
  template:
    spec:
      imageName: ubuntu-22.04 # OS Image Name
      type: cx21 # Machine Type
      sshKeys:
        - name: jhennig@mirantis.com # Needs exist in the Hetzner Cloud project
---
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: K0sWorkerConfigTemplate
metadata:
  name: mirantis-labs-hetzner-mc
  namespace: team-red
spec:
  template:
    spec:
      version: v1.27.2+k0s.0
      # More details of the worker configuration can be set here
```

After your adjustments you are ready to deploy the cluster to Hetzner with:

```
kubectl apply -f 'mirantis-labs-hetzner-cluster.yaml' 
```

After applying the manifests to the management cluster and confirming the infrastructure readiness, allow a few minutes for all components to provision. Once complete, your command line should display output similar to this:

```shell
% kubectl get cluster,machine
NAME                                        PHASE         AGE   VERSION
cluster.cluster.x-k8s.io/k0s-hetzner-test   Provisioned   6m   

NAME                                             CLUSTER            NODENAME   PROVIDERID                                 PHASE         AGE   VERSION
machine.cluster.x-k8s.io/k0s-hetzner-test-md-0   k0s-hetzner-test                                                         Provisioned   8m   
```

You can also check the status of the cluster deployment with `clusterctl`:
```shell
% clusterctl describe cluster  
NAME                                                       READY  SEVERITY  REASON                    SINCE  MESSAGE          
Cluster/k0s-hetzner-test                                   True                                       10m                      
├─ClusterInfrastructure - HetznerCluster/k0s-hetzner-test                                                                             
├─ControlPlane - K0smotronControlPlane/k0s-hetzner-test-cp                                                                           
└─Workers                                                                                                                  
  └─Other
```

### Configuration
This is a simple demo of k0s on Hetzner, k0smotron and k0s control planes in the management cluster. If you need more advanced configurations please check our documentation of k0s and k0smotron: 
- k0s Docs -> https://docs.k0sproject.io/
- k0smotron Docs -> https://docs.k0smotron.io/