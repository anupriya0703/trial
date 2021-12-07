<h1>Installing cStor</h1>


>Topics covered under this section
>- [cStor Operators](#cstor-operators)
>- [Why cStor Operators?](#why-cstor-operators)
>- Deploying cStor Operators
>     -  Prerequisites
>     - Installtaion
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
CSPC API comes with a variety of tunables and features and the API can be viewed for [here](https://github.com/openebs/api/blob/HEAD/pkg/apis/cstor/v1/cstorpoolcluster.go).


## <a class="anchor" aria-hidden="true" id="why-cstor-operators"></a>Why cStor Operators?



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
