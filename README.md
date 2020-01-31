# High Performance Shared File storage on AKS with Rook

For most shared file access needs on Azure Kubernetes Service (AKS) the use of the Azure Files service is adequate. Now with the ability to provision Azure Files on a [Premium Storage Account on AKS clusters with Kubernetes 1.13+](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv#create-a-storage-class), the overall performance of shared file access is even more performant.

What many have noticed however is that the type of files and access pattern will determine the performance characteristics of the shared file system. The overall consensus is that Azure Files performance best with large file read/write IO sizes. Also the way performance is provisioned is based on provisioned capacity so many  times you may need to allocate more capacity than really needed, to attain the performance target desired. A better description of this is available here: [Planning for Azure Files deployment](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-planning#file-share-performance-tiers)

With AKS another option exists however thanks to the CNCF incubating project called [Rook](https://rook.io/). Rook allows operators to provision a shared file service within cluster backed by Azure Disks which have higher performance characteristics in general. Rook can host File, Block and Object Storage in an automated way leveraging native cloud storage solutions. The Rook operator for Kubernetes extends this capability to offer automated deployment, bootstrapping, configuration, provisioning, scaling, upgrading, migration, disaster recovery, monitoring, and resource management.

Rook offers many Cloud Native Storage specifications including Ceph, EdgeFS and others. It uses the FlexVolume driver to integrate with Kubernetes. In AKS the FlexVolume plugin directory is non-standard and the kubelet is pre-configured to use that directory. This makes it easy to modify the default Rook deployment to use a specific plugin directory.

Let's start from the beginning:

## Deploying FelxVolume for AKS

1. Install the FlexVolume as a DaemonSet into our cluster:

```shell
kubectl apply -f https://raw.githubusercontent.com/Azure/kubernetes-keyvault-flexvol/master/deployment/kv-flexvol-installer.yaml
```

2. Install the Rook operator:

```shell
kubectl apply -f 
```

3. 