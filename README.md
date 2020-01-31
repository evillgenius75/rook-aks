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
kubectl apply -f https://raw.githubusercontent.com/evillgenius75/rook-aks/master/manifests/rook-operator.yaml
```

Verify the rook-ceph-operator, rook-ceph-agent, and rook-discover pods are in the `Running` state before proceeding (should take about two minutes):

```shell
kubectl -n rook-ceph-system get po
```

3. Now let's install the Rook Cluster as a CephCluster kind. This is a custom resource defined by the Rook operator.

```shell
kubectl apply -f https://raw.githubusercontent.com/evillgenius75/rook-aks/master/manifests/rook-cluster.yaml
```

Use kubectl to list the pods in the rook-ceph namespace. You will see a few pods and 2 completed jobs. ensure all the pods are running and jobs are completed before moving on to the next step.

```shell
kubectl -n rook-ceph get po

kubectl -n rook-ceph get pod

NAME                                                   READY     STATUS      RESTARTS
rook-ceph-mgr-a-5977cffcf8-vsmqz                       1/1       Running     0
rook-ceph-mon-a-74dc9d4bcd-62g8s                       1/1       Running     0
rook-ceph-mon-b-558cf6f9d4-7sh24                       1/1       Running     0
rook-ceph-osd-0-585bf769cc-jp5sw                       1/1       Running     0
rook-ceph-osd-1-76576b697f-v9zwz                       1/1       Running     0
rook-ceph-osd-prepare-aks-agentpool-19295063-0-btkbb   0/2       Completed   0
rook-ceph-osd-prepare-aks-agentpool-19295063-1-2fsfz   0/2       Completed   0
```

4. Install the shared file system component:

```shell
kubectl apply -f https://raw.githubusercontent.com/evillgenius75/rook-aks/master/manifests/rook-filesystem.yaml
```

To confirm the file system is configured, wait for the mds pods to start:

```shell
kubectl -n rook-ceph get pod -l app=rook-ceph-mds
```

You should be able to see the following pods once they are all running:

```shell
NAME                                    READY  STATUS    AGE  Node
rook-ceph-mds-myfs-a-5b7cbd4977-6tgv2   1/1    Running   1m   aks-agentpool-33026748-0
rook-ceph-mds-myfs-b-85dc94778c-8qfkb   1/1    Running   1m   aks-agentpool-33026748-1
```

## You now have a Azure Disk backed shared file system running in Cluster that is highly available!

We can now deploy a workload to the Ceph system using a volume definition. Here is an example of what would be used to create a mount to a container in a pod define a ceph volume:

```yaml
apiVersion: apps/v1
kind: Deployment
...
    spec:
      containers:
      - image: wordpress
        imagePullPolicy: Always
        name: wordpress-rook
        volumeMounts:
        - mountPath: "/var/www/html"
          name: ceph-volume
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
        - name: ceph-volume
          flexVolume:
            driver: ceph.rook.io/rook
            fsType: ceph
            options:
              fsName: myfs # name of the filesystem specified in the filesystem CRD.
              clusterNamespace: rook-ceph # namespace where the Rook cluster is deployed
              # by default the path is /, but you can override and mount a specific path of the filesystem by using the path attribute
              # the path must exist on the filesystem, otherwise mounting the filesystem at that path will fail
              # path: /some/path/inside/cephfs
```

As you can see the process to mount a volume in the container spec is the same as any kubernetes workload. Here the container mount path `/var/www/html` is mounted to a volume called `ceph-volume`. The Volume definition called the `ceph.rookio/rook` driver and the fsType is `ceph`.

If you want to test performance you can run an `fio` based workload to get read and write performance and view graphs based on the results.

