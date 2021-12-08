<h1>Installing cStor</h1>


>Topics covered under this section
>- [cStor Operators](#cstor-operators)
>- [Why cStor Operators?](#why-cstor-operators)
>- [Deploying cStor Operators](#deploying-cstor-operators)
>     - [Prerequisites](#prerequisites)
>     - [Installtaion](#installation)
>     - Creation of Storage Pools
>     - Creation of Storage Classes
>     - Deploy a Sample Application
>     - Upgrade
>     - Uninstall
>- Advanced operations
>     - Backup and Restore
>     - Pools
>         - Scaling cStor pools
>         - Block Device Tagging
>         - Tuning pools
>     - Volumes
>        - Expanding cStor volumes
>        - [Tuning cStor volumes](#tuning-vol)
>- Troubleshooting(Docs)
>- Alpha Features
>- Roadmaps
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

Troubleshooting(Docs)

Alpha Features
Roadmaps
