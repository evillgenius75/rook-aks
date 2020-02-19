# High Performance Shared File storage on AKS with Rook

For most shared file access needs on Azure Kubernetes Service (AKS) the use of the Azure Files service is adequate. Now with the ability to provision Azure Files on a [Premium Storage Account on AKS clusters with Kubernetes 1.13+](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv#create-a-storage-class), the overall performance of shared file access is even more performant.

What many have noticed however is that the type of files and access pattern will determine the performance characteristics of the shared file system. The overall consensus is that Azure Files performance best with large file read/write IO sizes. Also the way performance is provisioned is based on provisioned capacity so many  times you may need to allocate more capacity than really needed, to attain the performance target desired. A better description of this is available here: [Planning for Azure Files deployment](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-planning#file-share-performance-tiers)

With AKS another option exists however thanks to the CNCF incubating project called [Rook](https://rook.io/). Rook allows operators to provision a shared file service within cluster backed by Azure Disks which have higher performance characteristics in general. Rook can host File, Block and Object Storage in an automated way leveraging native cloud storage solutions. The Rook operator for Kubernetes extends this capability to offer automated deployment, bootstrapping, configuration, provisioning, scaling, upgrading, migration, disaster recovery, monitoring, and resource management.

Rook offers many Cloud Native Storage specifications including Ceph, EdgeFS and others. It uses the FlexVolume driver to integrate with Kubernetes. In AKS the FlexVolume plugin directory is non-standard and the kubelet is pre-configured to use that directory. This makes it easy to modify the default Rook deployment to use a specific plugin directory.

Let's start from the beginning:

## Create a nodepool for storage pods

We add taints to our nodes to make sure that no pods will be scheduled on this nodepool as long as we explicitly tolerate it. We want to have these nodes exclusively for storage pods. You can also assign the size of the nodes for this nodepool to get maximum IO based on disk configuration.

```shell
az aks nodepool add --cluster-name myrooktestclstr \
--name npstorage --resource-group rooktest-rg \
--node-count 3 \
--node-taints storage-node=true:NoSchedule
```

We should now have additional Nodes in our cluster for the second node pool.

```shell
kubectl get nodes

NAME                                 STATUS   ROLES   AGE    VERSION
aks-npstandard-33852324-vmss000000   Ready    agent   10m    v1.14.8
aks-npstandard-33852324-vmss000001   Ready    agent   10m    v1.14.8
aks-npstandard-33852324-vmss000002   Ready    agent   10m    v1.14.8
aks-npstorage-33852324-vmss000000    Ready    agent   2m3s   v1.14.8
aks-npstorage-33852324-vmss000001    Ready    agent   2m9s   v1.14.8
aks-npstorage-33852324-vmss000002    Ready    agent   119s   v1.14.8
```

## Installing Rook Components

Let’s start installing Rook by cloning the repository from GitHub:

$ git clone https://github.com/rook/rook.git
After we have downloaded the repo to our local machine, there are three steps we need to perform to install Rook:

1. Add Rook CRDs / namespace / common resources
2. Add and configure the Rook operator
3. Add the Rook cluster

So, switch to the /cluster/examples/kubernetes/ceph directory and follow the steps below.

1. Add Common Resources

```shell
$ kubectl apply -f common.yaml
```

The common.yaml contains the namespace rook-ceph, common resources (e.g. clusterroles, bindings, service accounts etc.) and some Custom Resource Definitions from Rook.

2. Add the Rook Operator

The operator is responsible for managing Rook resources and needs to be configured to run on Azure Kubernetes Service. To manage Flex Volumes, AKS uses a directory that’s different from the “default directory”. So, we need to tell the operator which directory to use on the cluster nodes.

Furthermore, we need to adjust the settings for the CSI plugin to run the corresponding daemonsets on the storage nodes (remember, we added taints to the nodes. By default, the pods of the daemonsets Rook needs to work, won’t be scheduled on our storage nodes – we need to “tolerate” this).

So, here’s the full operator.yaml file (→ **important parts**)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rook-ceph-operator
  namespace: rook-ceph
  labels:
    operator: rook
    storage-backend: ceph
spec:
  selector:
    matchLabels:
      app: rook-ceph-operator
  replicas: 1
  template:
    metadata:
      labels:
        app: rook-ceph-operator
    spec:
      serviceAccountName: rook-ceph-system
      containers:
      - name: rook-ceph-operator
        image: rook/ceph:master
        args: ["ceph", "operator"]
        volumeMounts:
        - mountPath: /var/lib/rook
          name: rook-config
        - mountPath: /etc/ceph
          name: default-config-dir
        env:
        - name: ROOK_CURRENT_NAMESPACE_ONLY
          value: "false"
→       - name: FLEXVOLUME_DIR_PATH
→         value: "/etc/kubernetes/volumeplugins"
        - name: ROOK_ALLOW_MULTIPLE_FILESYSTEMS
          value: "false"
        - name: ROOK_LOG_LEVEL
          value: "INFO"
        - name: ROOK_CEPH_STATUS_CHECK_INTERVAL
          value: "60s"
        - name: ROOK_MON_HEALTHCHECK_INTERVAL
          value: "45s"
        - name: ROOK_MON_OUT_TIMEOUT
          value: "600s"
        - name: ROOK_DISCOVER_DEVICES_INTERVAL
          value: "60m"
        - name: ROOK_HOSTPATH_REQUIRES_PRIVILEGED
          value: "false"
        - name: ROOK_ENABLE_SELINUX_RELABELING
          value: "true"
        - name: ROOK_ENABLE_FSGROUP
          value: "true"
        - name: ROOK_DISABLE_DEVICE_HOTPLUG
          value: "false"
        - name: ROOK_ENABLE_FLEX_DRIVER
          value: "false"
        # Whether to start the discovery daemon to watch for raw storage devices on nodes in the cluster.
        # This daemon does not need to run if you are only going to create your OSDs based on StorageClassDeviceSets with PVCs. --> CHANGED to false
        - name: ROOK_ENABLE_DISCOVERY_DAEMON
          value: "false"
        - name: ROOK_CSI_ENABLE_CEPHFS
          value: "true"
        - name: ROOK_CSI_ENABLE_RBD
          value: "true"
        - name: ROOK_CSI_ENABLE_GRPC_METRICS
          value: "true"
        - name: CSI_ENABLE_SNAPSHOTTER
          value: "true"
→       - name: CSI_PROVISIONER_TOLERATIONS
→         value: |
→           - effect: NoSchedule
→             key: storage-node
→             operator: Exists
→       - name: CSI_PLUGIN_TOLERATIONS
→         value: |
→           - effect: NoSchedule
→             key: storage-node
→             operator: Exists
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
      volumes:
      - name: rook-config
        emptyDir: {}
      - name: default-config-dir
        emptyDir: {}
```

3. Create the Cluster

Deploying the Rook cluster is as easy as installing the Rook operator. As we are running our cluster with the Azure Kubernetes Service – a managed service – we don’t want to manually add disks to our storage nodes. Also, we don’t want to use a directory on the OS disk (which most of the examples out there will show you) as this will be deleted when the node will be upgraded to a new Kubernetes version.

In this sample, we want to leverage Persistent Volumes / Persistent Volume Claims that will be used to request Azure Managed Disks which will in turn be dynamically attached to our storage nodes. Thankfully, when we installed our cluster, a corresponding storage class for using Premium SSDs from Azure was also created.

```shell
kubectl get storageclass

NAME                PROVISIONER                AGE
default (default)   kubernetes.io/azure-disk   15m
managed-premium     kubernetes.io/azure-disk   15m
```

Now, let’s create the Rook Cluster. Again, we need to adjust the tolerations and add a node affinity that our OSDs will be scheduled on the storage nodes (→ **important parts**):

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  dataDirHostPath: /var/lib/rook
  mon:
    count: 3
    allowMultiplePerNode: false
    volumeClaimTemplate:
      spec:
→       storageClassName: managed-premium
        resources:
          requests:
            storage: 10Gi
  cephVersion:
    image: ceph/ceph:v14.2.4-20190917
    allowUnsupported: false
  dashboard:
    enabled: true
    ssl: true
  network:
    hostNetwork: false
  storage:
    storageClassDeviceSets:
    - name: set1
      # The number of OSDs to create from this device set
      count: 4
      # IMPORTANT: If volumes specified by the storageClassName are not portable across nodes
      # this needs to be set to false. For example, if using the local storage provisioner
      # this should be false.
      portable: true
      # Since the OSDs could end up on any node, an effort needs to be made to spread the OSDs
      # across nodes as much as possible. Unfortunately the pod anti-affinity breaks down
      # as soon as you have more than one OSD per node. If you have more OSDs than nodes, K8s may
      # choose to schedule many of them on the same node. What we need is the Pod Topology
      # Spread Constraints, which is alpha in K8s 1.16. This means that a feature gate must be
      # enabled for this feature, and Rook also still needs to add support for this feature.
      # Another approach for a small number of OSDs is to create a separate device set for each
      # zone (or other set of nodes with a common label) so that the OSDs will end up on different
      # nodes. This would require adding nodeAffinity to the placement here.
      placement:
→       tolerations:
→       - key: storage-node
→         operator: Exists
→       nodeAffinity:
→         requiredDuringSchedulingIgnoredDuringExecution:
→           nodeSelectorTerms:
→           - matchExpressions:
→             - key: agentpool
→               operator: In
→               values:
→               - npstorage
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - rook-ceph-osd
                - key: app
                  operator: In
                  values:
                  - rook-ceph-osd-prepare
              topologyKey: kubernetes.io/hostname
      resources:
        limits:
          cpu: "500m"
          memory: "4Gi"
        requests:
          cpu: "500m"
          memory: "2Gi"
      volumeClaimTemplates:
      - metadata:
          name: data
        spec:
          resources:
            requests:
→             storage: 100Gi
→         storageClassName: managed-premium
          volumeMode: Block
          accessModes:
            - ReadWriteOnce
  disruptionManagement:
    managePodBudgets: false
    osdMaintenanceTimeout: 30
    manageMachineDisruptionBudgets: false
    machineDisruptionBudgetNamespace: openshift-machine-api
```

So, after a few minutes, you will see some pods running in the rook-ceph namespace. Make sure, that the OSD pods a running, before continuing with configuring the storage pool.

$ kubectl get pods -n rook-ceph
NAME                                                              READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-4qxsv                                            3/3     Running     0          28m
csi-cephfsplugin-d2klt                                            3/3     Running     0          28m
csi-cephfsplugin-jps5r                                            3/3     Running     0          28m
csi-cephfsplugin-kzgrt                                            3/3     Running     0          28m
csi-cephfsplugin-provisioner-dd9775cd6-nsn8q                      4/4     Running     0          28m
csi-cephfsplugin-provisioner-dd9775cd6-tj826                      4/4     Running     0          28m
csi-cephfsplugin-rt6x2                                            3/3     Running     0          28m
csi-cephfsplugin-tdhg6                                            3/3     Running     0          28m
csi-rbdplugin-6jkx5                                               3/3     Running     0          28m
csi-rbdplugin-clfbj                                               3/3     Running     0          28m
csi-rbdplugin-dxt74                                               3/3     Running     0          28m
csi-rbdplugin-gspqc                                               3/3     Running     0          28m
csi-rbdplugin-pfrm4                                               3/3     Running     0          28m
csi-rbdplugin-provisioner-6dfd6db488-2mrbv                        5/5     Running     0          28m
csi-rbdplugin-provisioner-6dfd6db488-2v76h                        5/5     Running     0          28m
csi-rbdplugin-qfndk                                               3/3     Running     0          28m
rook-ceph-crashcollector-aks-npstandard-33852324-vmss00000c8gdp   1/1     Running     0          16m
rook-ceph-crashcollector-aks-npstandard-33852324-vmss00000tfk2s   1/1     Running     0          13m
rook-ceph-crashcollector-aks-npstandard-33852324-vmss00000xfnhx   1/1     Running     0          13m
rook-ceph-crashcollector-aks-npstorage-33852324-vmss000001c6cbd   1/1     Running     0          5m31s
rook-ceph-crashcollector-aks-npstorage-33852324-vmss000002t6sgq   1/1     Running     0          2m48s
rook-ceph-mgr-a-5fb458578-s2lgc                                   1/1     Running     0          15m
rook-ceph-mon-a-7f9fc6f497-mm54j                                  1/1     Running     0          26m
rook-ceph-mon-b-5dc55c8668-mb976                                  1/1     Running     0          24m
rook-ceph-mon-d-b7959cf76-txxdt                                   1/1     Running     0          16m
rook-ceph-operator-5cbdd65df7-htlm7                               1/1     Running     0          31m
rook-ceph-osd-0-dd74f9b46-5z2t6                                   1/1     Running     0          13m
rook-ceph-osd-1-5bcbb6d947-pm5xh                                  1/1     Running     0          13m
rook-ceph-osd-2-9599bd965-hprb5                                   1/1     Running     0          5m31s
rook-ceph-osd-3-557879bf79-8wbjd                                  1/1     Running     0          2m48s
rook-ceph-osd-prepare-set1-0-data-sv78n-v969p                     0/1     Completed   0          15m
rook-ceph-osd-prepare-set1-1-data-r6d46-t2c4q                     0/1     Completed   0          15m
rook-ceph-osd-prepare-set1-2-data-fl8zq-rrl4r                     0/1     Completed   0          15m
rook-ceph-osd-prepare-set1-3-data-qrrvf-jjv5b                     0/1     Completed   0          15m

## Configuring Storage

Before Rook can provision persistent volumes, either a filesystem or a storage pool should be configured. In our example, a Ceph Filesystem Pool is used:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - replicated:
        size: 3
  preservePoolsOnDelete: true
  metadataServer:
    activeCount: 1
    activeStandby: true
```

Next, we also need a storage class that will be using the Rook cluster / storage pool. In our example, we will not be using Flex Volume (which will be deprecated in furture versions of Rook/Ceph), instead we use Container Storage Interface.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-cephfs
# Change "rook-ceph" provisioner prefix to match the operator namespace if needed
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  # clusterID is the namespace where operator is deployed.
  clusterID: rook-ceph
  # CephFS filesystem name into which the volume shall be created
  fsName: myfs
  # Ceph pool into which the volume shall be created
  # Required for provisionVolume: "true"
  pool: myfs-data0
  # Root path of an existing CephFS volume
  # Required for provisionVolume: "false"
  # rootPath: /absolute/path
  # The secrets contain Ceph admin credentials. These are generated automatically by the operator
  # in the same namespace as the cluster.
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
```

## You now have a Azure Disk backed shared file system running in Cluster that is highly available!

We can now deploy a workload to the Ceph system using a volume definition. Here is an example of what would be used to create a mount to a container in a pod define a ceph volume:



```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-cephfs
---
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
          persistentVolumeClaim:
            claimName: cephfs-pvc
            readOnly: false
```

As you can see the process to mount a volume in the container spec is the same as any kubernetes workload. Here the container mount path `/var/www/html` is mounted to a volume called `ceph-volume`. The Volume definition called the `cephfs-pvc` volume claim that is made from the `csi-cephfs` storageClass.

If you want to test performance you can run an `fio` based workload to get read and write performance and view graphs based on the results.

In the [perf-testing folder](./perf-testing/manifests/) of this repo, there is a dbench yaml configured to test against this shared storage.