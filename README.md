# eBPF-CaliEnt-Install
The easiest way to start an EKS cluster that meets eBPF mode’s requirements is to use Amazon’s Bottlerocket OS, instead of the default. Bottlerocket is a container-optimised OS with an emphasis on security; it has a version of the kernel which is compatible with eBPF mode.



When using eksctl it is important to use the config-file approach to creating a cluster in order to set the additional IAM permissions that Bottlerocket requires.
```
eksctl create cluster --config-file -f https://raw.githubusercontent.com/n1g3ld0uglas/eBPF-CaliEnt-Install/main/clusterconfig.yaml
```

