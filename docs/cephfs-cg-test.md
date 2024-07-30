# How to test Cephfs Consistency Group

## Environment Pre-request

Since the Ceph and Ceph-csi with the Consistency Group feature has not yet GA released, some additional configuration is required during deployment:

- Deploy the development build of Ceph: https://shaman.ceph.com/builds/ceph
- Deploy the latest build of external snapshotter: https://github.com/kubernetes-csi/external-snapshotter
- Deploy the latest build of ceph-csi: https://github.com/ceph/ceph-csi/tree/devel

An automation script has been provided for configuring this setup: 
https://github.com/youhangwang/ramen/blob/deploy/test/drenv-cg-start.sh. However, you still need to manually build the images for the external snapshotter and ceph-csi to use this script.

## Deploy Application with PVCs in a Consistency Group

Here is an application featuring CephFS Consistency Groups: 
- channel: https://github.com/youhangwang/ocm-ramen-samples.git/channel
- path: https://github.com/youhangwang/ocm-ramen-samples/tree/main/subscription/deployment-k8s-regional-cephfs

This application includes four PVCs, which are grouped into two Consistency Groups: cg1 and cg2. The Consistency Group designation is managed through the label `ramendr.openshift.io/consistency-group` on the PVCs. Currently, this label needs to be added manually. Ramen will soon automatically apply this label to PVCs based on their associated StorageClass.

```
kubectl get pvc -o custom-columns=NAME:.metadata.name,LABELS:.metadata.labels -n deployment-cephfs --context dr1

NAME               LABELS
busybox-cg1-pvc1   map[app:deployment-cephfs app.kubernetes.io/part-of:deployment-cephfs appname:busybox ramendr.openshift.io/consistency-group:cg1]
busybox-cg1-pvc2   map[app:deployment-cephfs app.kubernetes.io/part-of:deployment-cephfs appname:busybox ramendr.openshift.io/consistency-group:cg1]
busybox-cg2-pvc1   map[app:deployment-cephfs app.kubernetes.io/part-of:deployment-cephfs appname:busybox ramendr.openshift.io/consistency-group:cg2]
busybox-cg2-pvc2   map[app:deployment-cephfs app.kubernetes.io/part-of:deployment-cephfs appname:busybox ramendr.openshift.io/consistency-group:cg2]
```

## Enable Ramen Consistency Group Feature

By default, Ramen does not handle Consistency Groups, meaning it will continue to manage each PVC independently as before.

To enable Ramen to manage PVCs with the label `ramendr.openshift.io/consistency-group`, you need to add the annotation `drplacementcontrol.ramendr.openshift.io/is-cg-enabled: true` to the DRPC.

```
kchub get drpc deployment-cephfs-drpc -n deployment-cephfs -o yaml
apiVersion: ramendr.openshift.io/v1alpha1
kind: DRPlacementControl
metadata:
  annotations:
    drplacementcontrol.ramendr.openshift.io/app-namespace: deployment-cephfs
    drplacementcontrol.ramendr.openshift.io/is-cg-enabled: "true"
    drplacementcontrol.ramendr.openshift.io/last-app-deployment-cluster: dr2
  name: deployment-cephfs-drpc
  namespace: deployment-cephfs
  ...
spec:
  ...
```

## Application DR

Once the Ramen Consistency Group feature is enabled(add DRPC is-cg-enabled annotation) and the application PVC includes the label `ramendr.openshift.io/consistency-group`, you can activate the Consistency Group Disaster Recovery (DR) for the application.

Additionally, I have provided a demo video of the testing process for reference.

### Enable DR

#### Steps

Just enable DR as current way. From a user perspective, enabling the Consistency Group (CG) does not change the way you use the ramen system.

#### Expect Result

Workload using Consistency group is propagated successfully to secondary.

#### What happens

Internally, Ramen create additional resources to support CG. These additional resources require special attention during testing:

On the Primary Cluster, the additional resources created are:
- replicationgroupsource: Each Consistency Group will have a corresponding replicationgroupsource to manage the Consistency Group.
- volumegroupsnapshot: For each Consistency Group, a volumegroupsnapshot is created during each data sync. This snapshot captures a consistent state of the PVCs included in a Consistency Group. After the data sync is completed, the volumegroupsnapshot will be deleted.
- restored PVC: During each data sync, a temporary PVC is created for each Application PVC. This temporary PVC is named using the format `cephfscg-<Application PVC Name>` and is used to restore data from the volumegroupsnapshot. After the data sync is completed, this temporary PVC will be deleted.

On the Secondary Cluster, the additional resources created are:
- replicationgroupdestination: Each Consistency Group will have a corresponding replicationgroupdestination to manage the Consistency Group.
- volumesnapshot: Once the data transfer for an Application PVC is completed, Ramen will create a volumesnapshot on the Secondary Cluster to preserve the transferred data. After all PVCs in a Consistency Group have completed data transfer, Ramen will record all volumesnapshot entries in the replicationgroupdestination and remove the volumesnapshots from the previous round.


### Failover

#### Steps

Just trigger failover as current way. From a user perspective, the failover process remains the same whether or not Consistency Groups (CG) are enabled.

#### Expect Result

Failover to secondary completes successfully.

#### What happens

Internally, Ramen performs differnt operations to support CG:

When restoring a PVC on the secondary cluster in failover, Ramen reads the VolumeSnapshot from the replicationgroupdestination, as this destination holds the most recent and complete snapshots of the Consistency Group.

### Relocate
#### Steps

Just trigger relocate as current way. From a user perspective, the relocate process remains the same whether or not Consistency Groups (CG) are enabled. 

#### Expect Result

Relocation back to the preferred cluster succeeds.

#### What happens

Internally, Ramen performs differnt operations to support CG:

When restoring a PVC on the secondary cluster in relocate, Ramen reads the VolumeSnapshot from the replicationgroupdestination, as this destination holds the most recent and complete snapshots of the Consistency Group.