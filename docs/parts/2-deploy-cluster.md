<!-- See https://squidfunk.github.io/mkdocs-material/reference/ -->

# Part 2: Deploy Cluster

## Steps

### 1. Setup Work environment

We're going to walk through setting up the required tools in a Unix-like environment. This will be the environment where you will be running most of the commands in this guide. This could be a Linux host you deploy on Equinix Metal, a Mac, or a Windows machine with WSL. For this sake of this guide, we're going to presume a freshly deployed Ubuntu host.

#### Install Docker

First, we need to install Docker. This is the container runtime that we will use to run our management Kubernetes cluster. The easiest way to do this is with the docker convenience script.

```shell
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh ./get-docker.sh
```

#### Install kind (Kubernetes in Docker)

Next, we install Kind. Kind is a tool for running local Kubernetes clusters using Docker container "nodes". We'll use this to create a local kubernetes cluster that we can install Cluster API Provider Packet into and then create a Kubernetes cluster on Equinix Metal.

Please replace the version number below with the latest number from <https://github.com/kubernetes-sigs/kind/releases>.

```shell
# For AMD64 / x86_64
curl -L https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64 -o ./kind
sudo install -o root -g root -m 0755 ./kind /usr/local/bin/kind
```

#### Install clusterctl

Clusterctl is the CLI tool that handles installing and managing Cluster API based management clusters. We'll use this to install Cluster API Provider Packet into our local kubernetes cluster.

Please replace the version number below with the latest number from <https://github.com/kubernetes-sigs/cluster-api/releases>.

```shell
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.7.4/clusterctl-linux-amd64 -o ./clusterctl
sudo install -o root -g root -m 0755 ./clusterctl /usr/local/bin/clusterctl
```

#### Install kubectl

We use kubectl to interact with our kubernetes cluster.

```shell
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 ./kubectl /usr/local/bin/kubectl
```

#### Install helm

We'll use helm to easily install applications into our kubernetes cluster, we won't need it till the last section.

```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
sudo bash get_helm.sh
```

### 2. Deploy a local Kubernetes cluster

Now that we have all the tools we need, let's deploy a local Kubernetes cluster using kind.

```shell
kind create cluster --name capp
```

The output should look like the following:

```console
Creating cluster "capp" ...
✓ Ensuring node image (kindest/node:v1.30.0) 🖼
✓ Preparing nodes 📦
✓ Writing configuration 📜
✓ Starting control-plane 🕹️
✓ Installing CNI 🔌
✓ Installing StorageClass 💾
Set kubectl context to "kind-capp"
You can now use your cluster with:

kubectl cluster-info --context kind-capp

Thanks for using kind! 😊
```

If you're on a freshly installed Linux box like we are, you're kubectl should default to accessing the cluster you just created. You can verify this by running:

```shell
kubectl get nodes
```

Which will have the following output

```console
NAME                 STATUS     ROLES           AGE   VERSION
capp-control-plane   NotReady   control-plane   15s   v1.30.0
```

If you're in another environment you may need to do the following to setup your kubectl to use the kind cluster:

```shell
kind get kubeconfig --name=capp > kubeconfig-kind
export KUBECONFIG=kubeconfig-kind
```

### 3. Install Cluster API Provider Packet

Now that we have a Kubernetes cluster running, we can install Cluster API Provider Packet into it. We need to set some environment variables first, in order to properly configure the provider.

We've included some default values below for the environment variables, but you should replace them with your own values. You'll also need your API key and project ID from the Equinix Metal portal. You will need to use a User API key.

```shell
export CONTROLPLANE_NODE_TYPE=m3.small.x86
export WORKER_NODE_TYPE=m3.small.x86
export KUBERNETES_VERSION=1.30.2
export METRO=da
export PACKET_API_KEY=<YOUR_API_KEY>
export PROJECT_ID=<YOUR_PROJECT_ID>
export SSH_KEY=<YOUR_SSH_KEY>
```

Now we can install Cluster API Provider Packet into our local Kubernetes cluster.

```shell
clusterctl init --infrastructure=packet
```

The output looks something like the following:

```console
Fetching providers
Installing cert-manager Version="v1.15.1"
Waiting for cert-manager to be available...
Installing Provider="cluster-api" Version="v1.7.4" TargetNamespace="capi-system"
Installing Provider="bootstrap-kubeadm" Version="v1.7.4" TargetNamespace="capi-kubeadm-bootstrap-system"
Installing Provider="control-plane-kubeadm" Version="v1.7.4" TargetNamespace="capi-kubeadm-control-plane-system"
Installing Provider="infrastructure-packet" Version="v0.9.0" TargetNamespace="cluster-api-provider-packet-system"

Your management cluster has been initialized successfully!

You can now create your first workload cluster by running the following:

  clusterctl generate cluster [name] --kubernetes-version [version] | kubectl apply -f -
```

### 4. Deploy workload cluster with CAPP

Now that we have Cluster API Provider Packet installed, we can deploy a Kubernetes cluster on Equinix Metal. We'll use the `clusterctl generate cluster` command to generate a cluster manifest, and then apply it to our management cluster.

The following command creates the cluster with three control plane nodes and two worker nodes in order to be able to demonstrate the load balancer functionality of the cluster.

```shell
clusterctl generate cluster my-lbaas-demo --flavor emlb --control-plane-machine-count 3 --worker-machine-count 2 > my-lbaas-demo.yaml
kubectl apply -f my-lbaas-demo.yaml
```

:watch: It should take around 20 minutes for the cluster to be created.

### 5. Verify workload cluster

Let's validate that the cluster was created successfully.

First check the status of the cluster and ensure it says `Provisioned` in the PHASE column.

```shell
$ kubectl get cluster
NAME            CLUSTERCLASS   PHASE         AGE   VERSION
my-lbaas-demo                  Provisioned   19m
```

Next, check the status of the machines in the cluster. You should see the control plane and worker say `Running` in the PHASE column. Make sure you have the right count of control plane and worker nodes, for this example you should have 3 control plane nodes and 2 worker nodes.

```shell
$ kubectl get machines
NAME                                 CLUSTER         NODENAME                             PROVIDERID                                            PHASE     AGE   VERSION
my-lbaas-demo-control-plane-br5xj    my-lbaas-demo   my-lbaas-demo-control-plane-br5xj    equinixmetal://f23a7719-57fb-48ff-81a7-564d36b1b869   Running   10m   v1.30.2
my-lbaas-demo-control-plane-mmm5j    my-lbaas-demo   my-lbaas-demo-control-plane-mmm5j    equinixmetal://8693714a-0e9d-4d9c-81e1-5c4fdaec7a43   Running   20m   v1.30.2
my-lbaas-demo-control-plane-xmm8z    my-lbaas-demo   my-lbaas-demo-control-plane-xmm8z    equinixmetal://9ee9e770-fac2-4041-bde0-c54597daff20   Running   15m   v1.30.2
my-lbaas-demo-worker-a-tmjck-bggv9   my-lbaas-demo   my-lbaas-demo-worker-a-tmjck-bggv9   equinixmetal://05cae0b9-508e-49f1-a8a3-bddc76c6ebd1   Running   20m   v1.30.2
my-lbaas-demo-worker-a-tmjck-nr5zk   my-lbaas-demo   my-lbaas-demo-worker-a-tmjck-nr5zk   equinixmetal://b720948a-a3ac-4187-8051-0687c6cb1fa2   Running   20m   v1.30.2
```

### 6. Access the workload cluster

Now that the cluster is up and running, we can access it using kubectl.

First we need to get the kubeconfig for the cluster. We can do this by running the following command:

```shell
clusterctl get kubeconfig my-lbaas-demo > kubeconfig
export KUBECONFIG=kubeconfig
```

Now we can run kubectl commands against the cluster. For example, to get the nodes in the cluster:

```shell
$ kubectl get nodes
NAME                                 STATUS     ROLES           AGE   VERSION
my-lbaas-demo-control-plane-br5xj    NotReady   control-plane   11m   v1.30.2
my-lbaas-demo-control-plane-mmm5j    NotReady   control-plane   24m   v1.30.2
my-lbaas-demo-control-plane-xmm8z    NotReady   control-plane   19m   v1.30.2
my-lbaas-demo-worker-a-tmjck-bggv9   NotReady   <none>          16m   v1.30.2
my-lbaas-demo-worker-a-tmjck-nr5zk   NotReady   <none>          20m   v1.30.2
```

You'll note the nodes say they're `NotReady`. This is because we haven't installed a CNI plugin yet. We'll do that in the next section.

### 7. Install a CNI plugin

We need to install a CNI plugin in order to get networking working in our cluster. We'll use Calico for this example. We'll be using helm to do the install, but you can visit the [Calico website](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises) for other installation methods like operator or manifest based.

```shell
helm repo add projectcalico https://docs.tigera.io/calico/charts
helm install calico projectcalico/tigera-operator --namespace tigera-operator --create-namespace
```

Wait about 30 seconds for the calico pods to start up, then run the following command to check that they're all running.

```shell
$ kubectl get pods -n calico-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-85cd86bb8b-hjd6n   1/1     Running   0          63s
calico-node-474qq                          1/1     Running   0          63s
calico-node-dbqsp                          1/1     Running   0          63s
calico-node-pd9c2                          1/1     Running   0          63s
calico-node-rr5vg                          1/1     Running   0          63s
calico-node-vm5fk                          1/1     Running   0          63s
calico-typha-7df687ff48-mmn44              1/1     Running   0          54s
calico-typha-7df687ff48-r5ftw              1/1     Running   0          63s
calico-typha-7df687ff48-wwd8b              1/1     Running   0          54s
csi-node-driver-fp5h5                      2/2     Running   0          63s
csi-node-driver-lhddd                      2/2     Running   0          63s
csi-node-driver-rrvfs                      2/2     Running   0          63s
csi-node-driver-wmcn8                      2/2     Running   0          63s
csi-node-driver-x6ptt                      2/2     Running   0          63s
```

Now you can check the nodes again and they should be `Ready`.

```shell
$ kubectl get nodes
NAME                                 STATUS   ROLES           AGE   VERSION
my-lbaas-demo-control-plane-br5xj    Ready    control-plane   22m   v1.30.2
my-lbaas-demo-control-plane-mmm5j    Ready    control-plane   34m   v1.30.2
my-lbaas-demo-control-plane-xmm8z    Ready    control-plane   29m   v1.30.2
my-lbaas-demo-worker-a-tmjck-bggv9   Ready    <none>          27m   v1.30.2
my-lbaas-demo-worker-a-tmjck-nr5zk   Ready    <none>          30m   v1.30.2
```

### 8. We're done

We've successfully deployed a Kubernetes cluster with a highly available control plane on Equinix Metal using Cluster API Provider Packet. Your cluster is now up and running and ready for you to deploy your applications to it. Which we'll do in the next step when we configure the cluster for service based load balancing.

For more information on what it means to have a highly available control plane, see the [Kubernetes documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/#stacked-etcd-topology) about a stacked etcd topology, which is what we've deployed here.

### Common issues that may arise

- No more devices of a specific plan available in the metro you deployed the cluster to.
  - Try deploying to a different metro.
  - Try deploying a different plan.

## Discussion

Before proceeding to the next part let's take a few minutes to discuss what we did. Here are some questions to start the discussion.

- Why do we need to install these tools?
- Are there other ways to do the installation of these items?
- Can I use another kubernetes deployment method to create the management cluster?
