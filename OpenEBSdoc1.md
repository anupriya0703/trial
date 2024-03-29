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
>     - [Upgrade](#upgrade)
>     - [Uninstallation](#uninstall)
>- [Advanced operations](#adv-operations)
>     - [Backup and Restore](#backup-restore)
>     - [Pools](#pools)
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
 <b>Deploying a busybox application</b>
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

<div id="upgrade">
<details>
  <summary><b>Upgrade</b></summary>
This document describes the steps for OpenEBS Upgrade path: 1.8.0 or later to a newer release up to 2.12.0

   ### Prerequisites for upgrading OpenEBS 
    

**Note: All steps described in this document need to be performed from a machine that has access to Kubernetes master.**

**Note: It is mandatory to make sure to that all OpenEBS control plane and data plane components are running with the expected version before the upgrade.**

**Note: If the current version is 2.0.0 or below please run the given command to cleanup old upgradetask resources which can result in [error](https://github.com/openebs/openebs/issues/3392).**
```bash
kubectl -n <openebs-namespace> delete utasks --all
```

- **For upgrading to the latest release (2.12.0), the previous version should be minimum 1.6.0**

- Note down the `namespace` where openebs components are installed.
  The following document assumes that namespace to be `openebs`.

- Note down the `openebs service account`.
  The following command will help you to determine the service account name.
  ```sh
  $ kubectl get deploy -n openebs -l name=maya-apiserver -o jsonpath="{.items[*].spec.template.spec.serviceAccount}"
  ```
  The examples in this document assume the service account name is `openebs-maya-operator`.

- Verify that OpenEBS Control plane is indeed in expected version. Say 1.12.0
  ```sh
  $ kubectl get pods -n openebs -l openebs.io/version=1.12.0
  ```

  The output will list the control plane services mentioned below, as well as some
  of the data plane components.
  ```sh
  NAME                                           READY   STATUS    RESTARTS   AGE
  maya-apiserver-7b65b8b74f-r7xvv                1/1     Running   0          2m8s
  openebs-admission-server-588b754887-l5krp      1/1     Running   0          2m7s
  openebs-localpv-provisioner-77b965466c-wpfgs   1/1     Running   0          85s
  openebs-ndm-5mzg9                              1/1     Running   0          103s
  openebs-ndm-bmjxx                              1/1     Running   0          107s
  openebs-ndm-operator-5ffdf76bfd-ldxvk          1/1     Running   0          115s
  openebs-ndm-v7vd8                              1/1     Running   0          114s
  openebs-provisioner-678c549559-gh6gm           1/1     Running   0          2m8s
  openebs-snapshot-operator-75dc998946-xdskl     2/2     Running   0          2m6s
  ```

  Verify that `apiserver` is listed. If you have installed with helm charts,
  the apiserver name may be openebs-apiserver.


### Upgrade the OpenEBS Control Plane

Upgrade steps vary depending on the way OpenEBS was installed by you.
Below are steps to upgrade using some common ways to install OpenEBS:

### Prerequisite for control plane upgrade
1. Make sure all the blockdevices that are in use by cstor or localPV are connected to the node.
2. Make sure that all manually created and claimed blockdevices are excluded in the NDM configmap path
filter.

**NOTE: Upgrade of LocalPV rawblock volumes are not supported. Please exclude it in configmap**

eg: If partitions or dm devices are used, make sure it is added to the config map.
To edit the config map, run the following command
```bash
kubectl edit cm openebs-ndm-config -n openebs
```

Add the partitions or manually created disks into path filter if not already present

```yaml
- key: path-filter
        name: path filter
        state: true
        include: ""
        exclude: "loop,/dev/fd0,/dev/sr0,/dev/ram,/dev/dm-,/dev/md,/dev/rbd, /dev/sda1, /dev/nvme0n1p1"
``` 

Here, `/dev/sda1` and `/dev/nvm0n1p1` are partitions that are in use and blockdevices were manually created. It needs
to be included in the path filter of configmap

**Note: If you have any queries or see something unexpected, please reach out to the OpenEBS maintainers via [Github Issue](https://github.com/openebs/openebs/issues) or via #openebs channel on [Kubernetes Slack](https://slack.k8s.io).**

### Upgrade using kubectl (using openebs-operator.yaml):

**Use this mode of upgrade only if OpenEBS was installed using openebs-operator.yaml.**

**The sample steps below will work if you have installed OpenEBS without
modifying the default values in openebs-operator.yaml. If you have customized
the openebs-operator.yaml for your cluster, you will have to download the
desired openebs-operator.yaml and customize it again**

```
#Upgrade to OpenEBS control plane components to desired version. Say 2.12.0
$ kubectl apply -f https://openebs.github.io/charts/2.12.0/openebs-operator.yaml
```

### Upgrade using helm chart (using openebs/openebs, openebs-charts repo, etc.,):

**The sample steps below will work if you have installed openebs with
default values provided by openebs/openebs helm chart.**

Before upgrading via helm, please review the default values available with
latest openebs/openebs chart.
(https://github.com/openebs/charts/blob/master/charts/openebs/values.yaml).

- If the default values seem appropriate, you can use the below commands to
  update OpenEBS. [More](https://hub.helm.sh/charts/openebs/openebs) details about the specific chart version.
  ```sh
  $ helm upgrade --reset-values <release name> openebs/openebs --version 2.12.0
  ```
- If not, customize the values into your copy (say custom-values.yaml),
  by copying the content from above default yamls and edit the values to
  suite your environment. You can upgrade using your custom values using:
  ```sh
  $ helm upgrade <release name> openebs/openebs --version 2.12.0 -f custom-values.yaml`
  ```

### Using customized operator YAML or helm chart.
As a first step, you must update your custom helm chart or YAML with desired
release tags and changes made in the values/templates. After updating the YAML
or helm chart or helm chart values, you can use the above procedures to upgrade
the OpenEBS Control Plane components.

### After Upgrade
From 2.0.0 onwards, OpenEBS uses a new algorithm to generate the UUIDs for blockdevices to identify any type of disk across the 
nodes in the cluster. Therefore, blockdevices that were not used (Unclaimed state) in earlier versions will be made
Inactive and new resources will be created for them. Existing devices that are in use will continue to work normally.

**Note: After upgrading to 2.0.0 or above. If the devices that were in use before the upgrade are no longer required and becomes unclaimed at any point of time. Please restart NDM daemon pod on that node to sync those devices with the latest changes.**

### Upgrade the OpenEBS Pools and Volumes

**Note:**
- It is highly recommended to schedule a downtime for the application using the
OpenEBS PV while performing this upgrade. Also, make sure you have taken a
backup of the data before starting the below upgrade procedure.
- please have the following link handy in case the volume gets into read-only during upgrade
  https://docs.openebs.io/docs/next/t-volume-provisioning.html#recovery-readonly-when-kubelet-is-container
- If the pool and volume images have the prefix `quay.io/openebs/` then please add the flag
    ```yaml
    - "--to-version-image-prefix=openebs/"
    ```
  as the new multi-arch images are not pushed to quay.
  It can also be used specify any other private repository or airgap prefix in use.
- Before proceeding with the upgrade of the OpenEBS Data Plane components like cStor or Jiva,  verify that OpenEBS Control plane is indeed in desired version
  You can use the following command to verify components are in 2.12.0:
  ```sh
  $ kubectl get pods -n openebs -l openebs.io/version=2.12.0
  ```
  The above command should show that the control plane components are upgrade.
  The output should look like below:
  ```sh
  NAME                                           READY   STATUS    RESTARTS   AGE
  maya-apiserver-7b65b8b74f-r7xvv                1/1     Running   0          2m8s
  openebs-admission-server-588b754887-l5krp      1/1     Running   0          2m7s
  openebs-localpv-provisioner-77b965466c-wpfgs   1/1     Running   0          85s
  openebs-ndm-5mzg9                              1/1     Running   0          103s
  openebs-ndm-bmjxx                              1/1     Running   0          107s
  openebs-ndm-operator-5ffdf76bfd-ldxvk          1/1     Running   0          115s
  openebs-ndm-v7vd8                              1/1     Running   0          114s
  openebs-provisioner-678c549559-gh6gm           1/1     Running   0          2m8s
  openebs-snapshot-operator-75dc998946-xdskl     2/2     Running   0          2m6s
  ```

**Note: If you have any queries or see something unexpected, please reach out to the OpenEBS maintainers via [Github Issue](https://github.com/openebs/openebs/issues) or via #openebs channel on [Kubernetes Slack](https://slack.k8s.io).**

As you might have seen by now, control plane components and data plane components
work independently. Even after the OpenEBS Control Plane components have been
upgraded to 1.12.0, the Storage Pools and Volumes (both jiva and cStor)
will continue to work with older versions.

You can use the below steps for upgrading cstor and jiva components.

Starting with 1.1.0, the upgrade steps have been changed to eliminate the
need for downloading scripts. You can use `kubectl` to trigger an upgrade job
using Kubernetes Job spec.

The following instructions provide details on how to create your Upgrade Job specs.
Please ensure the `from` and `to` versions are as per your upgrade path. The below
examples show upgrading from 1.12.0 to 2.12.0.
### Upgrade cStor Pools

Extract the SPC name using `kubectl get spc`

```sh
NAME                AGE
cstor-disk-pool     26m
cstor-sparse-pool   24m
```

The Job spec for upgrade cstor pools is:

```yaml
#This is an example YAML for upgrading cstor SPC.
#Some of the values below needs to be changed to
#match your openebs installation. The fields are
#indicated with VERIFY
---
apiVersion: batch/v1
kind: Job
metadata:
  #VERIFY that you have provided a unique name for this upgrade job.
  #The name can be any valid K8s string for name. This example uses
  #the following convention: cstor-spc-<flattened-from-to-versions>
  name: cstor-spc-1120240

  #VERIFY the value of namespace is same as the namespace where openebs components
  # are installed. You can verify using the command:
  # `kubectl get pods -n <openebs-namespace> -l openebs.io/component-name=maya-apiserver`
  # The above command should return status of the openebs-apiserver.
  namespace: openebs
spec:
  backoffLimit: 4
  template:
    spec:
      #VERIFY the value of serviceAccountName is pointing to service account
      # created within openebs namespace. Use the non-default account.
      # by running `kubectl get sa -n <openebs-namespace>`
      serviceAccountName: openebs-maya-operator
      containers:
      - name:  upgrade
        args:
        - "cstor-spc"

        # --from-version is the current version of the pool
        - "--from-version=1.12.0"

        # --to-version is the version desired upgrade version
        - "--to-version=2.12.0"

        # If the pools and volumes images have the prefix `quay.io/openebs/`
        # then please add this flag as the new multi-arch images are not pushed to quay.
        # It can also be used specify any other private repository or airgap prefix in use.
        # "--to-version-image-prefix=openebs/"

        # Bulk upgrade is supported
        # To make use of it, please provide the list of SPCs
        # as mentioned below
        - "cstor-sparse-pool"
        - "cstor-disk-pool"
    
        #Following are optional parameters
        #Log Level
        - "--v=4"
        #DO NOT CHANGE BELOW PARAMETERS
        env:
        - name: OPENEBS_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        tty: true

        # the image version should be same as the --to-version mentioned above
        # in the args of the job
        image: openebs/m-upgrade:<same-as-to-version>
        imagePullPolicy: Always
      restartPolicy: OnFailure
---
```


### Upgrade cStor Volumes

Extract the PV name using `kubectl get pv`

```sh
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                                  STORAGECLASS           REASON    AGE
pvc-1085415d-f84c-11e8-aadf-42010a8000bb   5G         RWO            Delete           Bound     default/demo-cstor-sparse-vol1-claim   openebs-cstor-sparse             22m
pvc-a4aba0e9-8ad3-4d18-9b34-5e6e7cea2eb3   4G         RWO            Delete           Bound    default/cstor-disk-vol   openebs-cstor-disk            53s
```

Create a Kubernetes Job spec for upgrading the cstor volume. An example spec is as follows:
```yaml
#This is an example YAML for upgrading cstor volume.
#Some of the values below needs to be changed to
#match your openebs installation. The fields are
#indicated with VERIFY
---
apiVersion: batch/v1
kind: Job
metadata:
  #VERIFY that you have provided a unique name for this upgrade job.
  #The name can be any valid K8s string for name. This example uses
  #the following convention: cstor-vol-<flattened-from-to-versions>
  name: cstor-vol-1120240

  #VERIFY the value of namespace is same as the namespace where openebs components
  # are installed. You can verify using the command:
  # `kubectl get pods -n <openebs-namespace> -l openebs.io/component-name=maya-apiserver`
  # The above command should return status of the openebs-apiserver.
  namespace: openebs

spec:
  backoffLimit: 4
  template:
    spec:
      #VERIFY the value of serviceAccountName is pointing to service account
      # created within openebs namespace. Use the non-default account.
      # by running `kubectl get sa -n <openebs-namespace>`
      serviceAccountName: openebs-maya-operator
      containers:
      - name:  upgrade
        args:
        - "cstor-volume"

        # --from-version is the current version of the volume
        - "--from-version=1.12.0"

        # --to-version is the version desired upgrade version
        - "--to-version=2.12.0"

        # If the pools and volumes images have the prefix `quay.io/openebs/`
        # then please add this flag as the new multi-arch images are not pushed to quay.
        # It can also be used specify any other private repository or airgap prefix in use.
        # "--to-version-image-prefix=openebs/"

        # Bulk upgrade is supported from 1.9
        # To make use of it, please provide the list of PVs
        # as mentioned below
        - "pvc-c630f6d5-afd2-11e9-8e79-42010a800065"
        - "pvc-a4aba0e9-8ad3-4d18-9b34-5e6e7cea2eb3"
        
        #Following are optional parameters
        #Log Level
        - "--v=4"
        #DO NOT CHANGE BELOW PARAMETERS
        env:
        - name: OPENEBS_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        tty: true

        # the image version should be same as the --to-version mentioned above
        # in the args of the job
        image: openebs/m-upgrade:<same-as-to-version>
        imagePullPolicy: Always
      restartPolicy: OnFailure
---
```

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


## <a class="anchor" aria-hidden="true" id="adv-operations"></a>Advanced operations 
   <div id="backup-restore">
   <details>
     <summary><b>Snapshot and Clone of a cStor volume(Backup and Restore)</b></summary>


     An OpenEBS snapshot is a set of reference markers for data at a particular point in time. A snapshot act as a detailed table of contents, with accessible copies of data that user can roll back to the required point of instance. Snapshots in OpenEBS are instantaneous and are managed through kubectl.

During the installation of OpenEBS, a snapshot-controller and a snapshot-provisioner are setup which assist in taking the snapshots. During the snapshot creation, snapshot-controller creates VolumeSnapshot and VolumeSnapshotData custom resources. A snapshot-provisioner is used to restore a snapshot as a new Persistent Volume(PV) via dynamic provisioning.

### Creating a cStor volume Snapshot

1.  Before proceeding to create a cStor volume snapshot and use it further for restoration, it is necessary to create a `VolumeSnapshotClass`. Copy the following YAML specification into a file called `snapshot_class.yaml`.

    ```
    kind: VolumeSnapshotClass
    apiVersion: snapshot.storage.k8s.io/v1
    metadata:
    name: csi-cstor-snapshotclass
    annotations:
      snapshot.storage.kubernetes.io/is-default-class: "true"
    driver: cstor.csi.openebs.io
    deletionPolicy: Delete
    ```

    The deletion policy can be set as `Delete or Retain`. When it is set to Retain, the underlying physical snapshot on the storage cluster is retained even when the VolumeSnapshot object is deleted.
    To apply, execute:

    ```
    kubectl apply -f snapshot_class.yaml
    ```

    **Note:** In clusters that only install `v1beta1` version of VolumeSnapshotClass as the supported version(eg. OpenShift(OCP) 4.5 ), the following error might be encountered.

    ```
    no matches for kind "VolumeSnapshotClass" in version "snapshot.storage.k8s.io/v1"
    ```

    In such cases, the apiVersion needs to be updated to `apiVersion: snapshot.storage.k8s.io/v1beta1`

2.  For creating the snapshot, you need to create a YAML specification and provide the required PVC name into it. The only prerequisite check is to be performed is to ensure that there is no stale entries of snapshot and snapshot data before creating a new snapshot. Copy the following YAML specification into a file called `snapshot.yaml`.

    ```
    apiVersion: snapshot.storage.k8s.io/v1
    kind: VolumeSnapshot
    metadata:
    name: cstor-pvc-snap
    spec:
    volumeSnapshotClassName: csi-cstor-snapshotclass
    source:
      persistentVolumeClaimName: cstor-pvc
    ```

    Run the following command to create the snapshot,

    ```
    kubectl create -f snapshot.yaml
    ```

    To list the snapshots, execute:

    ```
    kubectl get volumesnapshots -n default
    ```

    Sample Output:

    ```shell hideCopy
    NAME                        AGE
    cstor-pvc-snap              10s
    ```

    A VolumeSnapshot is analogous to a PVC and is associated with a `VolumeSnapshotContent` object that represents the actual snapshot. To identify the VolumeSnapshotContent object for the VolumeSnapshot execute:

    ```
    kubectl describe volumesnapshots cstor-pvc-snap -n default
    ```

    Sample Output:

    ```shell hideCopy
    Name:         cstor-pvc-snap
    Namespace:    default
    .
    .
    .
    Spec:
    Snapshot Class Name:    cstor-csi-snapshotclass
    Snapshot Content Name:  snapcontent-e8d8a0ca-9826-11e9-9807-525400f3f660
    Source:
      API Group:
      Kind:       PersistentVolumeClaim
      Name:       cstor-pvc
    Status:
    Creation Time:  2020-06-20T15:27:29Z
    Ready To Use:   true
    Restore Size:   5Gi

    ```

    The `SnapshotContentName` identifies the `VolumeSnapshotContent` object which serves this snapshot. The `Ready To Use` parameter indicates that the Snapshot has been created successfully and can be used to create a new PVC.

**Note:** All cStor snapshots should be created in the same namespace of source PVC.

### Cloning a cStor Snapshot

Once the snapshot is created, you can use it to create a PVC. In order to restore a specific snapshot, you need to create a new PVC that refers to the snapshot. Below is an example of a YAML file that restores and creates a PVC from a snapshot.

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: restore-cstor-pvc
spec:
 storageClassName: cstor-csi-disk
 dataSource:
   name: cstor-pvc-snap
   kind: VolumeSnapshot
   apiGroup: snapshot.storage.k8s.io
 accessModes:
   - ReadWriteOnce
 resources:
   requests:
     storage: 5Gi
```

The `dataSource` shows that the PVC must be created using a VolumeSnapshot named `cstor-pvc-snap` as the source of the data. This instructs cStor CSI to create a PVC from the snapshot. Once the PVC is created, it can be attached to a pod and used just like any other PVC.

To verify the creation of PVC execute:

```
kubectl get pvc
```

Sample Output:

```shell hideCopy
NAME                           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS              AGE
restore-cstor-pvc              Bound    pvc-2f2d65fc-0784-11ea-b887-42010a80006c   5Gi        RWO            cstor-csi-disk            5s
```
   </details>
  <div id="pools">
   <details>
     <summary><b>Advanced operations on pools</b></summary> 
          <h2>Scaling up cStor pools</h2>
              Once the cStor storage pools are created you can scale-up your existing cStor pool.
To scale-up the pool size, you need to edit the CSPC YAML that was used for creation of CStorPoolCluster.

Scaling up can done by two methods:

1. [Adding new nodes(with new disks) to the existing CSPC](#adding-disk-new-node)
2. [Adding new disks to existing nodes](#adding-disk-same-node)

**Note:** The dataRaidGroupType: can either be set as stripe or mirror as per your requirement. In the following example it is configured as stripe.

### Adding new nodes(with new disks) to the existing CSPC

A new node spec needs to be added to previously deployed YAML,

```
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-disk-pool
 namespace: openebs
spec:
 pools:
   - nodeSelector:
       kubernetes.io/hostname: "worker-node-1"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-10ad9f484c299597ed1e126d7b857967"
     poolConfig:
       dataRaidGroupType: "stripe"

   - nodeSelector:
       kubernetes.io/hostname: "worker-node-2"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-3ec130dc1aa932eb4c5af1db4d73ea1b"
     poolConfig:
       dataRaidGroupType: "stripe"

   - nodeSelector:
       kubernetes.io/hostname: "worker-node-3"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-01afcdbe3a9c9e3b281c7133b2af1b68"
     poolConfig:
       dataRaidGroupType: "stripe"

   # New node spec added -- to create a cStor pool on worker-3
   - nodeSelector:
       kubernetes.io/hostname: "worker-node-4"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-02d9b2dc8954ce0347850b7625375e24"
     poolConfig:
       dataRaidGroupType: "stripe"

```

Now verify the status of CSPC and CSPI(s):

```
kubectl get cspc -n openebs
```

Sample Output:

```shell hideCopyshell hideCopy
NAME                     HEALTHYINSTANCES   PROVISIONEDINSTANCES   DESIREDINSTANCES   AGE
cspc-disk-pool           4                  4                      4                  8m5s
```

```
kubectl get cspi -n openebs
```

Sample Output:

```shell hideCopyshell hideCopy
NAME                  HOSTNAME         FREE     CAPACITY    READONLY   STATUS   AGE
cspc-disk-pool-d9zf   worker-node-1    28800M   28800071k   false      ONLINE   7m50s
cspc-disk-pool-lr6z   worker-node-2    28800M   28800056k   false      ONLINE   7m50s
cspc-disk-pool-x4b4   worker-node-3    28800M   28800056k   false      ONLINE   7m50s
cspc-disk-pool-rt4k   worker-node-4    28800M   28800056k   false      ONLINE   15s

```

As a result of this, we can see that a new pool have been added, increasing the number of pools to 4

### Adding new disks to existing nodes

A new `blockDeviceName` under `blockDevices` needs to be added to previously deployed YAML. Execute the following command to edit the CSPC,

```
kubectl edit cspc -n openebs cstor-disk-pool
```

Sample YAML:

```
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-disk-pool
 namespace: openebs
spec:
 pools:
   - nodeSelector:
       kubernetes.io/hostname: "worker-node-1"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-10ad9f484c299597ed1e126d7b857967"
           - blockDeviceName: "blockdevice-f036513d98f6c7ce31fd6e1ac3fad2f5" //# New blockdevice added
     poolConfig:
       dataRaidGroupType: "stripe"

   - nodeSelector:
       kubernetes.io/hostname: "worker-node-2"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-3ec130dc1aa932eb4c5af1db4d73ea1b"
           - blockDeviceName: "blockdevice-fb7c995c4beccd6c872b7b77aad32932" //# New blockdevice added
     poolConfig:
       dataRaidGroupType: "stripe"

   - nodeSelector:
       kubernetes.io/hostname: "worker-node-3"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-01afcdbe3a9c9e3b281c7133b2af1b68"
           - blockDeviceName: "blockdevice-46ddda7223b35b81415b0a1b12e40bcb" //# New blockdevice added
     poolConfig:
       dataRaidGroupType: "stripe"

```
<h2>Block Device Tagging</h2>
   ## Block Device Tagging

NDM provides you with an ability to reserve block devices to be used for specific applications via adding tag(s) to your block device(s). This feature can be used by cStor operators to specify the block devices which should be consumed by cStor pools and conversely restrict anyone else from using those block devices. This helps in protecting against manual errors in specifying the block devices in the CSPC yaml by users.

1. Consider the following block devices in a Kubernetes cluster, they will be used to provision a storage pool. List the labels added to these block devices,

```
kubectl get bd -n openebs --show-labels
```

Sample Output:

```shell hideCopy
NAME                                           NODENAME               SIZE          CLAIMSTATE   STATUS   AGE   LABELS
blockdevice-00439dc464b785256242113bf0ef64b9   worker-node-3          21473771008   Unclaimed    Active   34h   kubernetes.io/hostname=worker-node-3,ndm.io/blockdevice-type=blockdevice,ndm.io/managed=true
blockdevice-022674b5f97f06195fe962a7a61fcb64   worker-node-1          21473771008   Unclaimed    Active   34h   kubernetes.io/hostname=worker-node-1,ndm.io/blockdevice-type=blockdevice,ndm.io/managed=true
blockdevice-241fb162b8d0eafc640ed89588a832df   worker-node-2          21473771008   Unclaimed    Active   34h   kubernetes.io/hostname=worker-node-2,ndm.io/blockdevice-type=blockdevice,ndm.io/managed=true

```

2. Now, to understand how block device tagging works we will be adding `openebs.io/block-device-tag=fast` to the block device attached to worker-node-3 _(i.e blockdevice-00439dc464b785256242113bf0ef64b9)_

```
kubectl label bd blockdevice-00439dc464b785256242113bf0ef64b9 -n openebs  openebs.io/block-device-tag=fast
```

```
kubectl get bd -n openebs blockdevice-00439dc464b785256242113bf0ef64b9 --show-labels
```

Sample Output:

```shell hideCopy
NAME                                           NODENAME             SIZE          CLAIMSTATE   STATUS   AGE   LABELS
blockdevice-00439dc464b785256242113bf0ef64b9   worker-node-3        21473771008   Unclaimed    Active   34h   kubernetes.io/hostname=worker-node-3,ndm.io/blockdevice-type=blockdevice,ndm.io/managed=true,openebs.io/block-device-tag=fast
```

Now, provision cStor pools using the following CSPC YAML. Note, `openebs.io/allowed-bd-tags:` is set to `cstor, ssd` which ensures the CSPC will be created using the block devices that either have the label set to cstor or ssd, or have no such label.

```
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
name: cspc-disk-pool
namespace: openebs
annotations:
  # This annotation helps to specify the BD that can be allowed.
  openebs.io/allowed-bd-tags: cstor,ssd
spec:
pools:
  - nodeSelector:
      kubernetes.io/hostname: "worker-node-1"
    dataRaidGroups:
    - blockDevices:
        - blockDeviceName: "blockdevice-022674b5f97f06195fe962a7a61fcb64"
    poolConfig:
      dataRaidGroupType: "stripe"
- nodeSelector:
      kubernetes.io/hostname: "worker-node-2"
    dataRaidGroups:
      - blockDevices:
          - blockDeviceName: "blockdevice-241fb162b8d0eafc640ed89588a832df"
    poolConfig:
      dataRaidGroupType: "stripe"
- nodeSelector:
      kubernetes.io/hostname: "worker-node-3"
    dataRaidGroups:
      - blockDevices:
          - blockDeviceName: "blockdevice-00439dc464b785256242113bf0ef64b9"
    poolConfig:
      dataRaidGroupType: "stripe"
```

Apply the above CSPC file for CSPIs to get created and check the CSPI status.

```
kubectl apply -f cspc.yaml
```

```
kubectl get cspi -n openebs
```

Sample Output:

```shell hideCopy
NAME             HOSTNAME        FREE   CAPACITY    READONLY PROVISIONEDREPLICAS HEALTHYREPLICAS STATUS   AGE
cspc-stripe-b9f6 worker-node-2   19300M 19300614k   false      0                     0          ONLINE   89s
cspc-stripe-q7xn worker-node-1   19300M 19300614k   false      0                     0          ONLINE   89s

```

Note that CSPI for node **worker-node-3** is not created because:

- CSPC YAML created above has `openebs.io/allowed-bd-tags: cstor, ssd` in its annotation. Which means that the CSPC operator will only consider those block devices for provisioning that either do not have a BD tag, openebs.io/block-device-tag, on the block device or have the tag with the values set as `cstor or ssd`.
- In this case, the blockdevice-022674b5f97f06195fe962a7a61fcb64 (on node worker-node-1) and blockdevice-241fb162b8d0eafc640ed89588a832df (on node worker-node-2) do not have the label. Hence, no restrictions are applied on it and they can be used as the CSPC operator for pool provisioning.
- For blockdevice-00439dc464b785256242113bf0ef64b9 (on node worker-node-3), the label `openebs.io/block-device-tag` has the value fast. But on the CSPC, the annotation openebs.io/allowed-bd-tags has value cstor and ssd. There is no fast keyword present in the annotation value and hence this block device cannot be used.

**NOTE:**

1. To allow multiple tag values, the bd tag annotation can be written in the following comma-separated manner:

```
  openebs.io/allowed-bd-tags: fast,ssd,nvme
```

2. BD tag can only have one value on the block device CR. For example,
   - openebs.io/block-device-tag: fast
     Block devices should not be tagged in a comma-separated format. One of the reasons for this is, cStor allowed bd tag annotation takes comma-separated values and values like(i.e fast, ssd ) can never be interpreted as a single word in cStor and hence BDs tagged in above format cannot be utilised by cStor.
3. If any block device mentioned in CSPC has an empty value for `the openebs.io/block-device-tag`, then those block devices will not be considered for pool provisioning and other operations. Block devices with empty tag value are implicitly not allowed by the CSPC operator.

<h2>Tuning cStor Pools</h2>

Allow users to set available performance tunings in cStor Pools based on their workload. cStor pool(s) can be tuned via CSPC and is the recommended way to do it. Below are the tunings that can be applied:

**Resource requests and limits:** This ensures high quality of service when applied for the pool manager containers.

**Toleration for pool manager pod:** This ensures scheduling of pool pods on the tainted nodes.

**Set priority class:** Sets the priority levels as required.

**Compression:** This helps in setting the compression for cStor pools.

**ReadOnly threshold:** Helps in specifying read only thresholds for cStor pools.

**Example configuration for Resource and Limits:**

Following CSPC YAML specifies resources and auxResources that will get applied to all pool manager pods for the CSPC. Resources get applied to cstor-pool containers and auxResources gets applied to sidecar containers i.e. cstor-pool-mgmt and pool-exporter.

In the following CSPC YAML we have only one pool spec (@spec.pools). It is also possible to override the resource and limit value for a specific pool.

```
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-disk-pool
 namespace: openebs
spec:
 resources:
   requests:
     memory: "2Gi"
     cpu: "250m"
   limits:
     memory: "4Gi"
     cpu: "500m"

 auxResources:
   requests:
     memory: "500Mi"
     cpu: "100m"
   limits:
     memory: "1Gi"
     cpu: "200m"
 pools:
   - nodeSelector:
       kubernetes.io/hostname: worker-node-1

     dataRaidGroups:
     - blockDevices:
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f36
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f37

     poolConfig:
       dataRaidGroupType: mirror
```

Following CSPC YAML explains how the resource and limits can be overridden. If you look at the CSPC YAML, there are no resources and auxResources specified at pool level for worker-node-1 and worker-node-2 but specified for worker-node-3. In this case, for worker-node-1 and worker-node-2 the resources and auxResources will be applied from @spec.resources and @spec.auxResources respectively but for worker-node-3 these will be applied from @spec.pools[2].poolConfig.resources and @spec.pools[2].poolConfig.auxResources respectively.

```
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-disk-pool
 namespace: openebs
spec:
 resources:
   requests:
     memory: "64Mi"
     cpu: "250m"
   limits:
     memory: "128Mi"
     cpu: "500m"
 auxResources:
   requests:
     memory: "50Mi"
     cpu: "400m"
   limits:
     memory: "100Mi"
     cpu: "400m"
  pools:
   - nodeSelector:
       kubernetes.io/hostname: worker-node-1
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f36
           - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f37
     poolConfig:
       dataRaidGroupType: mirror

   - nodeSelector:
       kubernetes.io/hostname: worker-node-2
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f39
           - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f40
     poolConfig:
       dataRaidGroupType: mirror

   - nodeSelector:
       kubernetes.io/hostname: worker-node-3
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f42
           - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f43
     poolConfig:
       dataRaidGroupType: mirror
       resources:
         requests:
           memory: 70Mi
           cpu: 300m
         limits:
           memory: 130Mi
           cpu: 600m
       auxResources:
         requests:
           memory: 60Mi
           cpu: 500m
         limits:
           memory: 120Mi
           cpu: 500m

```

**Example configuration for Tolerations:**

Tolerations are applied in a similar manner like resources and auxResources. The following is a sample CSPC YAML that has tolerations specified. For worker-node-1 and worker-node-2 tolerations are applied form @spec.tolerations but for worker-node-3 it is applied from @spec.pools[2].poolConfig.tolerations

```
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-disk-pool
 namespace: openebs
spec:

 tolerations:
 - key: data-plane-node
   operator: Equal
   value: true
   effect: NoSchedule

 pools:
   - nodeSelector:
       kubernetes.io/hostname: worker-node-1

     dataRaidGroups:
     - blockDevices:
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f36
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f37

     poolConfig:
       dataRaidGroupType: mirror

   - nodeSelector:
       kubernetes.io/hostname: worker-node-2

     dataRaidGroups:
     - blockDevices:
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f39
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f40

     poolConfig:
       dataRaidGroupType: mirror

   - nodeSelector:
       kubernetes.io/hostname: worker-node-3

     dataRaidGroups:
     - blockDevices:
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f42
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f43

     poolConfig:
       dataRaidGroupType: mirror
       tolerations:
       - key: data-plane-node
         operator: Equal
         value: true
         effect: NoSchedule

       - key: apac-zone
         operator: Equal
         value: true
         effect: NoSchedule
```

**Example configuration for Priority Class:**

Priority Classes are also applied in a similar manner like resources and auxResources. The following is a sample CSPC YAML that has a priority class specified. For worker-node-1 and worker-node-2 priority classes are applied from @spec.priorityClassName but for worker-node-3 it is applied from @spec.pools[2].poolConfig.priorityClassName. Check more info about [priorityclass](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/#priorityclass).

**Note:**

1. Priority class needs to be created beforehand. In this case, high-priority and ultra-priority priority classes should exist.
2. The index starts from 0 for @.spec.pools list.

   ```
   apiVersion: cstor.openebs.io/v1
   kind: CStorPoolCluster
   metadata:
   name: cstor-disk-pool
   namespace: openebs
   spec:

   priorityClassName: high-priority

   pools:
     - nodeSelector:
         kubernetes.io/hostname: worker-node-1

       dataRaidGroups:
       - blockDevices:
           - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f36
           - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f37

       poolConfig:
         dataRaidGroupType: mirror

     - nodeSelector:
         kubernetes.io/hostname: worker-node-2

       dataRaidGroups:
       - blockDevices:
           - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f39
           - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f40

       poolConfig:
         dataRaidGroupType: mirror

     - nodeSelector:
         kubernetes.io/hostname: worker-node-3

       dataRaidGroups:
       - blockDevices:
           - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f42
           - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f43

       poolConfig:
         dataRaidGroupType: mirror
         priorityClassName: utlra-priority
   ```

   **Example configuration for Compression:**

   Compression values can be set at **pool level only**. There is no override mechanism like it was there in case of tolerations, resources, auxResources and priorityClass. Compression value must be one of

   - on
   - off
   - lzjb
   - gzip
   - gzip-[1-9]
   - zle
   - lz4

**Note:** lz4 is the default compression algorithm that is used if the compression field is left unspecified on the cspc. Below is the sample yaml which has compression specified.

```
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-disk-pool
 namespace: openebs
spec:
 pools:
   - nodeSelector:
       kubernetes.io/hostname: worker-node-1

     dataRaidGroups:
     - blockDevices:
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f36
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f37

     poolConfig:
       dataRaidGroupType: mirror
       compression: lz4
```

**Example configuration for Read Only Threshold:**

RO threshold can be set in a similar manner like compression. ROThresholdLimit is the threshold(percentage base) limit for pool read only mode. If ROThresholdLimit (%) amount of pool storage is consumed then the pool will be set to readonly. If ROThresholdLimit is set to 100 then entire pool storage will be used. By default it will be set to 85% i.e when unspecified on the CSPC. ROThresholdLimit value will be 0 < ROThresholdLimit <= 100. Following CSPC yaml has the ReadOnly Threshold percentage specified.

```
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-csi-disk
 namespace: openebs
spec:
 pools:
   - nodeSelector:
       kubernetes.io/hostname: worker-node-1

     dataRaidGroups:
     - blockDevices:
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f36
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f37

     poolConfig:
       dataRaidGroupType: mirror

       roThresholdLimit : 70
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





