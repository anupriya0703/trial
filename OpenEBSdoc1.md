<h1>Installing cStor</h1>


>Topics covered under this section
>- [cStor Operators](#cstor-operators)
>- [Why cStor Operators?](#why-cstor-operators)
>- [Deploying cStor Operators](#deploying-cstor-operators)
>     - [Prerequisites](#prerequisites)
>     - [Installtaion](#installation)
>     - [Creation of Storage Pools](#cspc)
>     - [Creation of Storage Classes](#sc)
>     - [Deploy a Sample Application](#application)
>     - Upgrade
>     - [Uninstallation](#uninstall)
>- Advanced operations
>     - Backup and Restore
>     - Pools
>         - Scaling cStor pools
>         - Block Device Tagging
>         - Tuning pools
>     - Volumes
>        - Expanding cStor volumes
>        - [Tuning cStor volumes](#tuning-vol)
>- [Troubleshooting](#troubleshooting)
>- [Alpha Features](#alphaFeatures)
>- [Roadmaps](#Roadmaps)
>- Contribute
>- Adopters and Feedback



## <a class="anchor" aria-hidden="true" id="cstor-operators"></a>What are cStor Operators

cStor operators are a collection of enhanced Kubernetes operators used for managing OpenEBS cStor Data Engine. At a high-level, cstor operators consist of following components.
 - cspc-operator
 - pool-manager
 - cvc-operator
 - volume-manager

An OpenEBS admin/user can use CSPC(CStorPoolCluster) API (YAML) to provision cStor pools in a Kubernetes cluster. As the name suggests, CSPC can be used to create a cluster of cStor pools across Kubernetes nodes. It is the job of cspc-operator to reconcile the CSPC object and provision CStorPoolInstance(s) as specified in the CSPC. A cStor pool is provisioned on node by utilising the disks attached to the node and is represented by CStorPoolInstance(CSPI) custom resource in a Kubernetes cluster. One has freedom to specify the disks that they want to use for pool provisioning.
CSPC API comes with a variety of tunables and features and the API can be viewed for <a href="https://github.com/openebs/api/blob/HEAD/pkg/apis/cstor/v1/cstorpoolcluster.go">here</a>.

Once a CSPC is created, cspc-operator provision CSPI CR and pool-manager deployment on each node where cStor pool should be created. The pool-manager deployment watches for its corresponding CSPI on the node and finally executes commands to perform pool operations e.g pool provisioning.
More info on cStor Pool CRs can be found [here](https://github.com/openebs/cstor-operators/blob/develop/docs/developer-guide/cstor-pool.md).


> **Note**: It is not recommended to modify the CSPI CR and pool-manager in the running cluster unless you know what you are trying to do. CSPC should be the only point of interaction.

Once the CStor pool(s) get provisioned successfully after creating CSPC, admin/user can create PVC to provision csi CStor volumes. When a user creates PVC, CStor CSI driver creates CStorVolumeConfig(CVC) resource, managed and reconciled by the cvc-controller which creates different volume-specific resources for each persistent volume, later managed by their respective controllers, more info can be found [here](https://github.com/openebs/cstor-operators/blob/develop/docs/developer-guide/cstor-volume.md).
The cStor operators work in conjunction with the [cStor CSI driver](https://github.com/openebs/cstor-csi) to provide cStor volumes for stateful workloads.

## <a class="anchor" aria-hidden="true" id="why-cstor-operators"></a>Why cStor Operators?

 cStor operators support a whole range of advanced operations on cStor pools and volumes. Some of which are listed below:
- Provision and De-provision cStor pools.
- Pool expansion by adding disk.
- Disk replacement by removing a disk.
- Volume replica scale up and scale down.
- Volume resize.
- Backup and Restore via Velero-plugin.
- Seamless upgrades of cStor Pools and Volumes
- Support migration from old cStor operators (using SPC) to new cStor operators using CSPC and CSI Driver.

<i>Each of these operations is detailed here.</i>

## <a class="anchor" aria-hidden="true" id="deploying-cstor-operators"></a>Deploying cStor Operators
   <div id="prerequisites">
   <details>
     <summary><b>Prerequisites</b></summary>

   1. Kubernetes version 1.17 or higher.
   2. iSCSI initiator utils installed on all the worker nodes. 
   >In case of a Rancher based cluster ensure the rerequisites mentioned [here](https://github.com/   openebs/cstor-operators/blob/develop/docs/troubleshooting/rancher_prerequisite.md) are met.



  | OPERATING SYSTEM | iSCSI PACKAGE         | Commands to install iSCSI                                | Verify iSCSI Status         |
  | ---------------- | --------------------- | -------------------------------------------------------- | --------------------------- |
  | RHEL/CentOS      | iscsi-initiator-utils | <ul><li>sudo yum install iscsi-initiator-utils -y</li><li>sudo systemctl enable --now iscsid</li></ul> | sudo systemctl status iscsid.service |
  | Ubuntu/Debian   | open-iscsi            |  <ul><li>sudo apt install open-iscsi -y</li><li>sudo systemctl enable --now iscsid</li></ui>| sudo systemctl status iscsid.service |
  | RancherOS        | open-iscsi            |  <ul><li>sudo ros s enable open-iscsi</li><li>sudo ros s up open-iscsi</li></ui>| ros service list iscsi |

   3. You have disks attached to nodes to provision the storage. The disks MUST not have any filesystem and the disks MUST not be mounted on the Node. cStor requires raw block devices. You can use the `lsblk -fa` command to check if the disks have a filesystem or if the disk is mounted.
   </details>

<div id="installation">
   <details>
     <summary><b>Installation</b></summary>

   Check for existing NDM components in your openebs namespace. Execute the following command:

```
$ kubectl -n openebs get pods -l openebs.io/component-name=ndm

NAME                                                              READY   STATUS    RESTARTS   AGE
openebs-ndm-gctb7                                                 1/1     Running   0          6d7h
openebs-ndm-sfczv                                                 1/1     Running   0          6d7h
openebs-ndm-vgdnv                                                 1/1     Running   0          6d6h
```

If you have got an output as displayed above, then it is recommended that you proceed with installation using the [CStor operators helm chart](https://openebs.github.io/cstor-operators). You will have to exclude `openebs-ndm` charts from the installation. Sample command:

```
helm install openebs-cstor openebs-cstor/cstor -n openebs --set openebsNDM.enabled=false
```

<details>
  <summary>Click here if you're using MicroK8s.</summary>

  ```bash
  microk8s helm3 install openebs-cstor openebs-cstor/cstor -n openebs --set-string csiNode.kubeletDir="/var/snap/microk8s/common/var/lib/kubelet/" --set openebsNDM.enabled=false
  ```
</details>

If you did not get any meaningful output (as above), then you do not have NDM components installed. Proceed with any one of the installation options below.

### Using Helm Charts:
 
Install CStor operators and CSI driver components using the [CStor Operators helm charts](https://openebs.github.io/cstor-operators). Sample command:

```bash
helm install openebs-cstor openebs-cstor/cstor -n openebs --create-namespace
```
<details>
  <summary>Click here if you're using MicroK8s.</summary>

  ```bash
  microk8s helm3 install openebs-cstor openebs-cstor/cstor -n openebs --create-namespace --set-string csiNode.kubeletDir="/var/snap/microk8s/common/var/lib/kubelet/"
  ```
</details>


[Click here](https://github.com/openebs/cstor-operators/blob/HEAD/deploy/helm/charts/README.md) for detailed instructions.

### Using Operator:

Install the latest release using CStor Operator yaml.

```bash
kubectl apply -f https://openebs.github.io/charts/cstor-operator.yaml
```
<details>
  <summary>Click here if you're using MicroK8s.</summary>

  ```bash
  microk8s kubectl apply -f https://openebs.github.io/charts/microk8s-cstor-operator.yaml
  ```
</details>


### Local Development:


Alternatively, you may also install the development version  of CStor Operators using:

```bash
$ git clone https://github.com/openebs/cstor-operators.git
$ cd cstor-operators
$ kubectl create -f deploy/yamls/rbac.yaml
$ kubectl create -f deploy/yamls/ndm-operator.yaml
$ kubectl create -f deploy/crds
$ kubectl create -f deploy/yamls/cspc-operator.yaml
$ kubectl create -f deploy/yamls/csi-operator.yaml
```

 **Note: If running on K8s version lesser than 1.17, you will need to comment the `priorityClassName: system-cluster-critical` in the csi-operator.yaml**
 
Once installed using any of the above methods, verify that all NDM and CStor operators pods are running. 

```bash
$ kubectl get pod -n openebs

NAME                                                              READY   STATUS    RESTARTS   AGE
cspc-operator-5fb7db848f-wgnq8                                    1/1     Running   0          6d7h
cvc-operator-7f7d8dc4c5-sn7gv                                     1/1     Running   0          6d7h
openebs-cstor-admission-server-7585b9659b-rbkmn                   1/1     Running   0          6d7h
openebs-cstor-csi-controller-0                                    7/7     Running   0          6d7h
openebs-cstor-csi-node-dl58c                                      2/2     Running   0          6d7h
openebs-cstor-csi-node-jmpzv                                      2/2     Running   0          6d7h
openebs-cstor-csi-node-tfv45                                      2/2     Running   0          6d7h
openebs-ndm-gctb7                                                 1/1     Running   0          6d7h
openebs-ndm-operator-7c8759dbb5-58zpl                             1/1     Running   0          6d7h
openebs-ndm-sfczv                                                 1/1     Running   0          6d7h
openebs-ndm-vgdnv                                                 1/1     Running   0          6d6h
```

Check that blockdevices are created:

```bash
$ kubectl get bd -n openebs

NAME                                           NODENAME           SIZE          CLAIMSTATE   STATUS   AGE
blockdevice-01afcdbe3a9c9e3b281c7133b2af1b68   worker3            21474836480   Unclaimed    Active   2m10s
blockdevice-10ad9f484c299597ed1e126d7b857967   worker1            21474836480   Unclaimed    Active   2m17s
blockdevice-3ec130dc1aa932eb4c5af1db4d73ea1b   worker2            21474836480   Unclaimed    Active   2m12s
```

NOTE:
1. It can take little while for blockdevices to appear when the application is warming up.
2. For a blockdevice to appear, you must have disks attached to node.
</details>

<div id="uninstall">
<details>
     <summary><b>Uninstallation</b></summary>


## Cleaning up a cStor setup

Follow the steps below to cleanup of a cStor setup. On successful cleanup you can reuse the cluster's disks/block devices for other storage engines.

1. Delete the application or deployment which uses CSI based cStor CAS engine. In this example we are  going to delete the Busybox application that was deployed previously. To delete, execute:

   ```
   kubectl delete pod <pod-name>
   ```

   Example command:
  
   ```
   kubectl delete busybox
   ```

   Verify that the application pod has been deleted

   ```
   kubectl get pods
   ```

   Sample Output:

   ```shell hideCopy
   No resources found in default namespace.
   ```

2. Next, delete the corresponding PVC attached to the application. To delete PVC, execute:

   ```
   kubectl delete pvc <pvc-name>
   ```

   Example command:

   ```
   kubectl delete pvc cstor-pvc
   ```

   Verify that the application-PVC has been deleted.

   ```
    kubectl get pvc
   ```

   Sample Output:

   ```shell hideCopy
    No resources found in default namespace.
   ```

3. Delete the corresponding StorageClass used by the application PVC.

   ```
    kubectl delete sc <storage-class-name>
   ```

   Example command:

   ```
   kubectl delete sc cstor-csi-disk
   ```

   To verify that the StorageClass has been deleted, execute:

   ```
   kubectl get sc
   ```

   Sample Output:

   ```shell hideCopy
    No resources found
   ```

4. The blockdevices used to create CSPCs will currently be in claimed state. To get the blockdevice details, execute:

   ```
    kubectl get bd -n openebs
   ```

   Sample Output:

   ```shell hideCopy
    NAME                                          NODENAME         SIZE         CLAIMSTATE  STATUS   AGE
    blockdevice-01afcdbe3a9c9e3b281c7133b2af1b68  worker-node-3    21474836480  Claimed     Active   2m10s
    blockdevice-10ad9f484c299597ed1e126d7b857967  worker-node-1    21474836480  Claimed     Active   2m17s
    blockdevice-3ec130dc1aa932eb4c5af1db4d73ea1b  worker-node-2    21474836480  Claimed     Active   2m12s
   ```

   To get these blockdevices to unclaimed state delete the associated CSPC. To delete, execute:

   ```
   kubectl delete cspc <CSPC-name> -n openebs
   ```

   Example command:

   ```
   kubectl delete cspc cstor-disk-pool -n openebs
   ```

   Verify that the CSPC and CSPIs have been deleted.

   ```
    kubectl get cspc -n openebs
   ```

   Sample Output:

   ```shell hideCopy
    No resources found in openebs namespace.
   ```

   ```
    kubectl get cspi -n openebs
   ```

   Sample Output:

   ```shell hideCopy
    No resources found in openebs namespace.
   ```

   Now, the blockdevices must be unclaimed state. To verify, execute:

   ```
    kubectl get bd -n openebs
   ```

   Sample output:

   ```shell hideCopy
    NAME                                          NODENAME         SIZE         CLAIMSTATE   STATUS   AGE
    blockdevice-01afcdbe3a9c9e3b281c7133b2af1b68  worker-node-3    21474836480   Unclaimed   Active   21m10s
    blockdevice-10ad9f484c299597ed1e126d7b857967  worker-node-1    21474836480   Unclaimed   Active   21m17s
    blockdevice-3ec130dc1aa932eb4c5af1db4d73ea1b  worker-node-2    21474836480   Unclaimed   Active   21m12s
   ```

</details>


<div id="cpsc">
<details>
  <summary>
 <b>Creation of Storage pools</b>
  </summary>
  For simplicity, this guide will provision a stripe pool on three nodes. A minimum of 3 replicas (on 3 nodes) is recommended for high-availability.

1. Use the CSPC file from [examples/cspc/cspc-single.yaml](/examples/cspc/cspc-single.yaml) and modify by performing
follwing steps:

   Modify CSPC to add your node selector for the node where you want to provision the pool.
   
   List the nodes with labels:

   ```bash
   kubectl get node --show-labels
   ```
   
   ```bash
   NAME               STATUS   ROLES    AGE    VERSION   LABELS
   master1            Ready    master   5d2h   v1.18.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master1,kubernetes.io/os=linux,node-role.kubernetes.io/master=

   worker1            Ready    <none>   5d2h   v1.18.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker1,kubernetes.io/os=linux

   worker2            Ready    <none>   5d2h   v1.18.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker2,kubernetes.io/os=linux

   worker3            Ready    <none>   5d2h   v1.18.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker3,kubernetes.io/os=linux

   ```
   
   In this guide, worker1 is picked. Modify the CSPC yaml to use this worker.
   (Note: Use the value from labels kubernetes.io/hostname=worker1 as this label value and node name could be different in some platforms)

   ```yaml
   kubernetes.io/hostname: "worker1"
   ```

   Modify CSPC to add blockdevice attached to the same node where you want to provision the pool.
   
   ```bash
   kubectl get bd -n openebs
   ```
   
   ```bash
   NAME                                           NODENAME           SIZE          CLAIMSTATE   STATUS   AGE
   blockdevice-01afcdbe3a9c9e3b281c7133b2af1b68   worker3            21474836480   Unclaimed    Active   2m10s
   blockdevice-10ad9f484c299597ed1e126d7b857967   worker1            21474836480   Unclaimed    Active   2m17s
   blockdevice-3ec130dc1aa932eb4c5af1db4d73ea1b   worker2            21474836480   Unclaimed    Active   2m12s
   ```
    
   ```yaml
   - blockDeviceName: "blockdevice-10ad9f484c299597ed1e126d7b857967"
   ```
   
   Finally the CSPC YAML looks like the following :
   ```yaml
   apiVersion: cstor.openebs.io/v1
   kind: CStorPoolCluster
   metadata:
     name: cstor-storage
     namespace: openebs
   spec:
     pools:
       - nodeSelector:
           kubernetes.io/hostname: "worker-1"
         dataRaidGroups:
           - blockDevices:
               - blockDeviceName: "blockdevice-10ad9f484c299597ed1e126d7b857967"
         poolConfig:
           dataRaidGroupType: "stripe"
   
       - nodeSelector:
           kubernetes.io/hostname: "worker-2" 
         dataRaidGroups:
           - blockDevices:
               - blockDeviceName: "blockdevice-3ec130dc1aa932eb4c5af1db4d73ea1b"
         poolConfig:
           dataRaidGroupType: "stripe"
      
       - nodeSelector:
           kubernetes.io/hostname: "worker-3"
         dataRaidGroups:
           - blockDevices:
               - blockDeviceName: "blockdevice-01afcdbe3a9c9e3b281c7133b2af1b68"
         poolConfig:
           dataRaidGroupType: "stripe"
   ```

2.  Apply the modified CSPC YAML.

    ```bash
    kubectl apply -f cspc-single.yaml
    ```
3. Check if the pool instances report their status as 'ONLINE'.

    ```bash
    kubectl get cspc -n openebs
    ```

    ```bash
    NAME            HEALTHYINSTANCES   PROVISIONEDINSTANCES   DESIREDINSTANCES   AGE
    cstor-storage   1                  1                      1                  2m2s

    ```

    ```bash
    kubectl get cspi -n openebs
    ```

    ```bash
    NAME                 HOSTNAME           ALLOCATED   FREE     CAPACITY   STATUS   AGE
    cstor-storage-vn92   worker1            260k        19900M   19900M     ONLINE   2m17s
    cstor-storage-al65   worker2            260k        19900M   19900M     ONLINE   2m17s
    cstor-storage-y7pn   worker3            260k        19900M   19900M     ONLINE   2m17s
    ```

</details>

<div id="sc">
<details>
  <summary>
 <b>Creation of Storage Class</b>
  </summary>
  Once your pool instances have come online, you can proceed with volume provisioning.
    Create a storageClass to dynamically provision volumes using OpenEBS CSI provisioner.
    A sample storageClass:

   ```yaml
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
     name: cstor-csi
   provisioner: cstor.csi.openebs.io
   allowVolumeExpansion: true
   parameters:
     cas-type: cstor
     # cstorPoolCluster should have the name of the CSPC
     cstorPoolCluster: cstor-storage
     # replicaCount should be <= no. of CSPI
     replicaCount: "3"
   ```

   Create a storageClass using above example.

   ```bash
   kubectl apply -f csi-cstor-sc.yaml
   ```

   You will need to specify the correct cStor CSPC from your cluster
   and specify the desired `replicaCount` for the volume. The `replicaCount`
   should be less than or equal to the max pool instances available.

- Create a PVC yaml using above created StorageClass name

    ```yaml
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: demo-cstor-vol
    spec:
      storageClassName: cstor-csi
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
     ```

    Apply the above pvc yaml to dynamically create volume and verify that
    the PVC has been successfully created and bound to a PersistentVolume (PV).

    ```bash
    $ kubectl get pvc
    NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
    demo-cstor-vol    Bound    pvc-52d88903-0518-11ea-b887-42010a80006c   5Gi        RWO            cstor-csi-stripe   10s
    ```

- Verify that the all volume-specific resources have been created
    successfully. Check if CStorColumeConfig(cvc) is in `Bound` state.

    ```bash
    $ kubectl get cstorvolumeconfig -n openebs
    NAME                                         CAPACITY   STATUS    AGE
    pvc-52d88903-0518-11ea-b887-42010a80006c2    5Gi        Bound     60s
    ```

    Verify volume and its replicas are in `Healthy` state.

    ```bash
    $ kubectl get cstorvolume -n openebs
    NAME                                         CAPACITY   STATUS    AGE
    pvc-52d88903-0518-11ea-b887-42010a80006c2    5Gi        Healthy   60s
    ```

    ```bash
    $ kubectl get cstorvolumereplica -n openebs
    NAME                                                          ALLOCATED   USED    STATUS    AGE
    pvc-52d88903-0518-11ea-b887-42010a80006c-cstor-storage-vn92   6K          6K      Healthy   60s
    pvc-52d88903-0518-11ea-b887-42010a80006c-cstor-storage-al65   6K          6K      Healthy   60s
    pvc-52d88903-0518-11ea-b887-42010a80006c-cstor-storage-y7pn   6K          6K      Healthy   60s
    ```

</details>


<div id="application">
<details>
  <summary>
 <b>Application</b>
  </summary>
  Create an application and use the above created PVC.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: busybox
      namespace: default
    spec:
      containers:
      - command:
           - sh
           - -c
           - 'date >> /mnt/openebs-csi/date.txt; hostname >> /mnt/openebs-csi/hostname.txt; sync; sleep 5; sync; tail -f /dev/null;'
        image: busybox
        imagePullPolicy: Always
        name: busybox
        volumeMounts:
        - mountPath: /mnt/openebs-csi
          name: demo-vol
      volumes:
      - name: demo-vol
        persistentVolumeClaim:
          claimName: demo-cstor-vol
    ```

    Verify that the pod is running and is able to write data to the volume.

    ```bash
    $ kubectl get pods
    NAME      READY   STATUS    RESTARTS   AGE
    busybox   1/1     Running   0          97s
    ```

    The example busybox application will write the current date into the
    mounted path at `/mnt/openebs-csi/date.txt` when it starts.

    ```bash
    $ kubectl exec -it busybox -- cat /mnt/openebs-csi/date.txt
    Wed Jul 12 07:00:26 UTC 2020
    ```
</details>

## <a class="anchor" aria-hidden="true" id="tuning-vol"></a>Tuning cStor volumes
<details>
<summary>
Replica Affinity to create a volume replica on specific pool
</summary>
For StatefulSet applications, to distribute single replica volume on specific cStor pool we can use replicaAffinity enabled scheduling. This feature should be used with delay volume binding i.e. volumeBindingMode: WaitForFirstConsumer in StorageClass. When volumeBindingMode is set to WaitForFirstConsumer the csi-provisioner waits for the scheduler to select a node. The topology of the selected node will then be set as the first entry in preferred list and will be used by the volume controller to create the volume replica on the cstor pool scheduled on preferred node
</details>

<details>
<summary>
Volume Target Pod Affinity
</summary>
The Stateful workloads access the OpenEBS storage volume by connecting to the Volume Target Pod. Target Pod Affinity policy can be used to co-locate volume target pod on the same node as the workload. This feature makes use of the Kubernetes Pod Affinity feature that is dependent on the Pod labels. For this labels need to be added to both, Application and volume Policy. Given below is a sample YAML of CStorVolumePolicy having target-affinity label using kubernetes.io/hostname as a topologyKey in CStorVolumePolicy:
</details>

</details>
<details>
<summary>
Volume Tunable
</summary>
Performance tunings based on the workload can be set using Volume Policy. The list of tunings that can be configured are given below:

queueDepth:
This limits the ongoing IO count from iscsi client on Node to cStor target pod. The default value for this parameter is set at 32.
luworkers:
cStor target IO worker threads, sets the number of threads that are working on QueueDepth queue. The default value for this parameter is set at 6. In case of better number of cores and RAM, this value can be 16, which means 16 threads will be running for each volume.
zvolWorkers:
cStor volume replica IO worker threads, defaults to the number of cores on the machine. In case of better number of cores and RAM, this value can be 16.
Given below is a sample YAML that has the above parameters configured
</details>


<details>
<summary>
Memory and CPU Resources QoS
</summary>
CStorVolumePolicy can also be used to configure the volume Target pod resource requests and limits to ensure QoS. Given below is a sample YAML that configures the target container's resource requests and limits, and auxResources configuration for the sidecar containers.

To know more about Resource configuration in Kubernetes, click here.
</details>


<details>
<summary>
Toleration for target pod to ensure scheduling of target pods on tainted nodes
</summary>
This Kubernetes feature allows users to taint the node. This ensures no pods are be scheduled to it, unless a pod explicitly tolerates the taint. This Kubernetes feature can be used to reserve nodes for specific pods by adding labels to the desired node(s).

One such scenario where the above tunable can be used is: all the volume specific pods, to operate flawlessly, have to be scheduled on nodes that are reserved for storage.
</details>


<details>
<summary>
Priority class for volume target deployment
</summary>
Priority classes can help in controlling the Kubernetes schedulers decisions to favor higher priority pods over lower priority pods. The Kubernetes scheduler can even preempt lower priority pods that are running, so that pending higher priority pods can be scheduled. Setting pod priority also prevents lower priority workloads from impacting critical workloads in the cluster, especially in cases where the cluster starts to reach its resource capacity. To know more about PriorityClasses in Kubernetes, click here.
</details>

## <a class="anchor" aria-hidden="true" id="troubleshooting"></a>Troubleshooting

## <a class="anchor" aria-hidden="true" id="alphaFeatures"></a>Alpha Features

## <a class="anchor" aria-hidden="true" id="roadmaps"></a>Roadmaps





