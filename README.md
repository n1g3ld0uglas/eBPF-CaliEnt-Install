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

Create the following config map in the tigera-operator namespace using the host and port determined above:
```
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/eBPF-CaliEnt-Install/main/configmap.yaml
```

