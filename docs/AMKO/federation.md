## Federation

AMKO uses federation to replicate AMKO configuration to a set of member clusters. This ensures a seamless recovery of AMKO configs for disaster recovery.

### Federation Set
The set of clusters registered on AMKO is considered as a federation set. One of these clusters in the federation set must be designated as the **leader**. And, all the other clusters are designated as the **followers**.

### Responsibilities of a leader and a follower
1. A **leader** cluster is responsible for distributing the AMKO configuration to all the follower clusters in the federation set. AMKO in the leader cluster is also responsible to create/update GSLB Services on the Avi Controller.
2. AMKO in a **follower** cluster runs passively and does not carry out federation activities or GslbService sync operations unlike the leader.

Post a disaster, the user should manually pick any of the erstwhile follower clusters and designate that as the new leader. The user must ensure that only a single AMKO is designated as the **leader** at any given point in time.

### Federated Objects
Following objects are federated from the leader cluster to the follower:
1. `GSLBConfig`
2. `GlobalDeploymentPolicy` or `GDP`

Add/Update/Delete events of the above objects are federated to the follower clusters.

### Flow
Assume the following topology (a cluster is a kubernetes kubernetes/openshift cluster):
- Cluster 1 in Site 1
- Cluster 2 in Site 2
- Cluster 3 in Site 3

![Alt text](images/amko_fed_initial.png?raw=true "AMKO federation flow")

Each site has an Avi Controller deployed. Site 1's Avi Controller is chosen as the GSLB Leader and all the other sites are GSLB followers. AMKO is deployed on all three sites. For cluster 1 (in site 1), it is marked as the leader. And, for clusters 2 and 3, it is marked as follower. User adds/updates the `GSLBConfig` and `GDP` objects in cluster 1, AMKO's federator in cluster 1 federates the changes to these objects to all the follower clusters.

### Federation Configuration using AMKOCluster CRD
A CRD called `AMKOCluster` governs the federation. A typical `AMKOCluster` object for a leader AMKO looks like this:
```yaml
apiVersion: amko.vmware.com/v1alpha1
kind: AMKOCluster
metadata:
  name: amkocluster-sample
  namespace: avi-system
spec:
  isLeader: true
  clusterContext: cluster1
  version: 1.4.2
  clusters:
  - cluster1
  - cluster2
status:
  conditions:
  - status: valid AMKOCluster object
    type: current AMKOCluster Validation
  - status: all cluster clients fetched
    type: member cluster initialisation
  - status: validated all member clusters
    type: member cluster validation
  - status: federated to all valid clusters successfully
    type: GSLBConfig Federation
  - status: federated to all valid clusters successfully
    type: GDP Federation
```
1. `namespace`: namespace of this object must be `avi-system`.
2. `isLeader`: Users must specify whether the AMKO in the current cluster is leader. Default value is `false`. If set to `false`, AMKO won't sync any objects to the Avi Controller, and the AMKO federator won't federate the objects to the member clusters.
3. `clusterContext`: Users must specify the current cluster's context. Providing the wrong cluster context can cause undefined behavior.
4. `version`: Current cluster's AMKO version. If installed via helm, this field gets automatically filled up.
5. `clusters`: Member cluster list on which federation will be performed. Current cluster (if present) in this list will be ignored.
6. `status`: Indicates the current state of federation. Following types are reflected in the status:
  * `current AMKOCluster Validation`: indicates the validity of the current `AMKOCluster` object.
  * `member cluster initialisation`: indicates whether the cluster contexts given in `spec.clusters` were fetched and initialised from the `gslb-config-secret` secret. If a member cluster given in `spec.clusters` is not found in the `gslb-config-secret`, this step would fail.
  * `member cluster validation`: The federator validates all the member clusters in the `spec.clusters` list and indicates a success/error. Validation includes some sanity checks, version mismatch checks, leader checks etc.
  * `GSLBConfig federation`: The federator indicates whether it was able to federate the `GSLBConfig` object to all the clusters in `spec.clusters` successfully.
  * `GDP Federation`: The federator indicates whether it was able to federate the `GDP`/`GlobalDeploymentPolicy` object to all the clusters in `spec.clusters` successfully.

**Note** that if `helm` is used to deploy AMKO, this Custom Resource will be installed, and the users have to provide these values via `values.yaml`.


**Additional Notes**:
* The federation set can be a subset of the overall member cluster set used for GSLB.
* `spec.clusterContext` must contain the current cluster's context.
* `spec.version` is compared against the versions of all member clusters. All AMKO clusters must have the same version as the `leader` cluster. The federation logic will not work if there's a version mismatch.
* `spec.clusters` contains the federation cluster set.
* Only one AMKO can be a leader, all other AMKOs have to be followers. If there are two leaders at any point, federation will stop and the error will be written to the `AMKOCluster`'s status.

### Disasters and Recovery
During a cluster down event on the `leader` AMKO, the federation of config objects will stop. However, at this point, all other clusters participating in federation would be synced with up to date configuration of the erstwhile GSLB `leader`. Hence, switching to a new AMKO `leader` does not require any manual steps of recovering the AMKO config objects.

Let's assume that there are 3 sites with one cluster in each of them:
1. Cluster 1 in Site 1
2. Cluster 2 in Site 2
3. Cluster 3 in Site 3

A disaster can occur either in the entire site or just for that cluster. Site failure would also mean that the kubernetes/openshift cluster along with the Avi Controller in that site are down. Whereas, a cluster failure would mean that only the kubernetes/openshift cluster is down.

#### Site Failure Scenario

 ![Alt text](images/amko_fed_switch.png?raw=true "AMKO site failure switch")

The site where the leader AMKO was deployed and which hosted the Avi GSLB leader, fails. At this point, the user has to:
1. Choose a new Avi GSLB leader controller out of the GSLB followers by following [these](https://avinetworks.com/docs/20.1/how-to-select-a-leader-site-manually/) steps.
2. Choose a new follower AMKO to be the new leader, and perform the following steps on the cluster where this follower AMKO is deployed:
  a. Edit the `GSLBConfig` object and change the leader IP address:

        $ kubectl edit gslbconfig -n avi-system gc-1
        // Set the field spec.gslbLeader.controllerIP to the new leader's IP address

  b. Set the `isLeader` field in the `AMKOCluster` object to true on this cluster:

    $ kubectl edit amkocluster amkocluster-federation -n avi-system

This would reboot the new leader AMKO. Post reboot, the new `leader` will take over the responsibilities of the earlier leader.

#### Cluster Failure Scenario

 ![Alt text](images/amko_fed_cluster_failure.png?raw=true "AMKO cluster failure switch")

The cluster where the leader AMKO was deployed, fails. Since the Avi GSLB leader is still active, the user only has to choose a new AMKO leader out of the followers. The steps to recover AMKO and designation of the new `leader` remains the same as the above section:
```
$ kubectl edit amkocluster amkocluster-federation -n avi-system
// set the spec.leader field to true
```

#### Old leader AMKO boots up
At any given point in time, the architecture only allows a single AMKO `leader`. Conflicts leading from more than 1 leader must be resolved by the admin manually. This does not have any traffic impact on the existing GslbServices objects.
To resolve this situation, the admin must convert one of the leader AMKOs to follower by setting `spec.isLeader` field to `false` in the `AMKOCluster` object:
```
$ kubectl edit amkocluster amkocluster-federation -n avi-system
```
This is especially important for situations, when a cluster, which hosted a leader instance of AMKO, previously failed. The user has switched a follower AMKO to be the new leader. And, the failed cluster recovers and brings back the old leader AMKO. The user must set the old leader to follower.