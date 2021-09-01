# eBPF-CaliEnt-Install-EKS
The easiest way to start an EKS cluster that meets eBPF mode’s requirements is to use Amazon’s Bottlerocket OS, instead of the default. Bottlerocket is a container-optimised OS with an emphasis on security; it has a version of the kernel which is compatible with eBPF mode.

Download and extract the latest release of eksctl with the following command
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```
Move the extracted binary to /usr/local/bin.
```
sudo mv /tmp/eksctl /usr/local/bin
```
Test that your installation was successful with the following command
```
eksctl version
```

Download the vended kubectl binary for your cluster's Kubernetes version from Amazon S3
```
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
```
Apply execute permissions to the binary
```
chmod +x ./kubectl
```
Create a $HOME/bin/kubectl and ensuring that $HOME/bin comes first in your $PATH
```
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
```
Add $HOME/bin path to your shell initialization file so it's configured when opening shell
```
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
```
After you install kubectl , you can verify its version with the following command
```
kubectl version --short --client
```

Download the cluster configuration file
```
wget https://raw.githubusercontent.com/n1g3ld0uglas/eBPF-CaliEnt-Install/main/clusterconfig.yaml
```

When using eksctl it is important to use the config-file approach to creating a cluster in order to set the additional IAM permissions that Bottlerocket requires.
```
eksctl create cluster --config-file clusterconfig.yaml
```

<img width="1616" alt="Screenshot 2021-07-16 at 10 55 46" src="https://user-images.githubusercontent.com/82048393/125929908-3f30dde8-c722-4152-a039-55439421d0f2.png">


For an EKS cluster, it’s important to use the domain name of the EKS-provided load balancer that is in front of the API server. 
This can be found by running the following command:
```
kubectl cluster-info
```

<img width="1066" alt="Screenshot 2021-07-16 at 11 23 42" src="https://user-images.githubusercontent.com/82048393/125933659-f0cc134b-6f7b-4080-9c4c-36be61cc77c5.png">


which should give output similar to the following:
```
Kubernetes master is running at https://9F907E01D0053CB4A993C940313F63D3.gr7.eu-west-1.eks.amazonaws.com
```



Create the following config map in the tigera-operator namespace 
```
kubectl create namespace tigera-operator
```


Use the host and port determined from the previous cluster-info command:
```
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/eBPF-CaliEnt-Install/main/configmap.yaml
```

In the following example for an AWS cloud provider integration, the StorageClass tells Calico Enterprise to use EBS disks for log storage.
```
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/eBPF-CaliEnt-Install/main/storageclass.yaml
```

Install the Tigera operator and custom resource definitions.
```
kubectl create -f https://docs.tigera.io/manifests/tigera-operator.yaml
```

<img width="1203" alt="Screenshot 2021-07-16 at 11 35 34" src="https://user-images.githubusercontent.com/82048393/125935120-bb0a69fa-31ff-446d-b694-597317fe7d3e.png">

Install the Prometheus operator and related custom resource definitions. 
The Prometheus operator will be used to deploy Prometheus server and Alertmanager to monitor Calico Enterprise metrics.

```
kubectl create -f https://docs.tigera.io/manifests/tigera-prometheus-operator.yaml
```

Install your pull secret.
```
kubectl create secret generic tigera-pull-secret \
    --type=kubernetes.io/dockerconfigjson -n tigera-operator \
    --from-file=.dockerconfigjson=config.json
```

When the main install guide tells you to apply the custom-resources.yaml, typically you will run kubectl create with the URL of the file directly. 
In this case you should instead download the file, so that you can edit it:

```
curl -o custom-resources.yaml https://docs.tigera.io/manifests/eks/custom-resources.yaml
```

<img width="1042" alt="Screenshot 2021-07-16 at 11 44 26" src="https://user-images.githubusercontent.com/82048393/125936123-480f20cb-d890-453f-b3c3-3dba5c83b66c.png">


* Edit the file in your editor of choice and find the Installation resource, which should be at the top of the file. 
* To enable eBPF mode, we need to add a new calicoNetwork section inside the spec of the Installation resource.
* You would also need to include the linuxDataplane field. 
* For EKS only, you should also add the flexVolumePath setting as shown below.

<img width="1042" alt="Screenshot 2021-07-16 at 11 51 54" src="https://user-images.githubusercontent.com/82048393/125937010-2246fab7-222d-4a96-afe4-e408b61f3956.png">


```
kubectl create -f custom-resources.yaml
```

You can monitor progress of the installation with the following command:

```
watch kubectl get tigerastatus
```

<img width="746" alt="Screenshot 2021-07-16 at 15 10 34" src="https://user-images.githubusercontent.com/82048393/125961357-de5377ce-6d9c-4a6c-9568-ab834b7b6fff.png">


In EKS, you can disable kube-proxy, reversibly, by adding a node selector that doesn’t match and nodes to kube-proxy’s DaemonSet, for example:
```
kubectl patch ds -n kube-system kube-proxy -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": "true"}}}}}'
```

# Alternatively, hook-up cluster to Calico Cloud

First, create an Amazon EKS cluster without any nodes.
```
eksctl create cluster --name my-calico-cluster --without-nodegroup
```
Since this cluster will use Calico for networking, you must delete the aws-node daemon set to disable AWS VPC networking for pods.
```
kubectl delete daemonset -n kube-system aws-node
```
Now that you have a cluster configured, you can install Calico.
```
kubectl apply -f https://docs.projectcalico.org/archive/v3.20/manifests/calico-vxlan.yaml
```
Finally, add nodes to the cluster.
```
eksctl create nodegroup --cluster my-calico-cluster --node-type t3.medium --node-ami auto --max-pods-per-node 100
```

Install process is outlined here --> https://docs.projectcalico.org/archive/v3.20/getting-started/kubernetes/managed-public-cloud/eks
