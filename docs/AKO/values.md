## Description of tunables of AKO

This document is intended to help the operator make the right choices while deploying AKO with the configurable settings.
The values.yaml in AKO affects a configmap that AKO's deployment reads to make adjustments as per user needs. Listed are detailed
explanation of various fields specified in the values.yaml. If the field is marked as "editable", it means that it can be edited without an AKO POD restart.


### AKOSettings.fullSyncFrequency

This field is used to set a frequency of consitency checks in AKO. Typically inconsistent states can arise if users make changes out
of band w.r.t AKO. For example, a pool is deleted by the user from the UI of the Avi Controller. The full sync frequency is used
to ensure that the models are re-conciled and the corresponding Avi objects are restored to the original state. 

### AKOSettings.logLevel *(editable)*

This flag defines the logLevel for logging and can be set to one of `DEBUG`, `INFO`, `WARN`, `ERROR` (case sensitive).
The logLevel value specified here gets populated in the ConfigMap and can be edited at any time while AKO is running. AKO picks up the change in the param value and sets the logLevel at runtime, so AKO pod restart is not required.

### AKOSettings.deleteConfig *(editable)*

This flag is intended to be used for deletion of objects in AVI Controller. The default value is false. 
If the value is set to true while booting up, AKO won't process any kubernetes object and stop regular operations. 

While AKO is running, this value can be edited to "true" in AKO configmap to delete all abjects created by AKO in AVI.
After that, if the value is set to "false", AKO would resume processing kubernetes objects and recreate all the objects in AVI. 

### AKOSettings.disableStaticRouteSync

This flag can be used in 2 scenarios:
 - If your POD CIDRs are routable either through an internal implementation or by default.
 - If you are working with multiple NICs on your kubernetes worker nodes and the default gateway is not from the same subnet as
   your VRF's PG network.

### AKOSettings.clusterName

The `clusterName` field primarily identifies your running AKO instance. AKO internally uses this field to tag all the objects it creates on Avi Controller. All objects created by a particular AKO instance have a prefix of `<clusterName>--` in their names and also populates the `created_by` like so `ako-<clusterName>`.

Each AKO instance mapped to a given Avi cloud should have a unique `clusterName` parameter. This would maintain uniqueness of object naming across Kubernetes clusters.

### AKOSettings.cniPlugin

Use this flag only if you are using `calico`/`openshift` as a CNI and you are looking to a sync your static route configurations automatically.
Once enabled, for `calico` this flag is used to read the `blockaffinity` CRD to determine the POD CIDR to Node IP mappings. If you are
on an older version of calico where `blockaffinity` is not present, then leave this field as blank. For `openshift` hostsubnet CRD is used to to determine the POD CIDR to Node IP mappings. 

AKO will then determine the static routes based on the Kubernetes Nodes object as done with other CNIs. 


### NetworkSettings.nodeNetworkList

The `nodeNetworkList` lists the Networks and Node CIDR's where the k8s Nodes are created. This is only used in the ClusterIP deployment of AKO and in vCenter cloud and only when disableStaticRouteSync is set to false.

If two Kubernetes clusters have overlapping POD CIDRs, the service engine needs to identify the right gateway for each of the overlapping CIDR groups. This is achieved by specifying the right placement network for the pools that helps the Service Engine place the pools appropriately. 

### NetworkSettings.subnetIP and NetworkSettings.subnetPrefix and NetworkSettings.networkName

AKO supports dual arm deployment where the Virtual IP network can be on a different subnet than the actual Port Groups on which the kubernetes nodes are deployed.

These fields are used to specify the Virtual IP network details on which the user wants to place the Avi virtual services on.

### L7Settings.shardVSSize

AKO uses a sharding logic for Layer 7 ingress objects. A sharded VS involves hosting multiple insecure or secure ingresses hosted by
one virtual IP or VIP. Having a shared virtual IP allows lesser IP usage since reserving IP addresses particularly in public clouds
incur greater cost. 

### L7Settings.passthroughShardSize

This is applicable only in openshift environment.
AKO uses a sharding logic for passthrough routes, these are distinct from the shared Virtual Services used for Layer 7 ingress or route objects. For all passthrough routes, a set of shared Virtual Services are created. The number of such Virtual Services is controlled by this flag.

### L7Settings.defaultIngController

This field is related to the ingress class support in AKO specified via `kubernetes.io/ingress.class` annotation specified on an
ingress object.

- If AKO is set as the default ingress controller, then it will sync everything other than the ones on which the ingress class is specified and is not equals to “avi”.
- If Avi is not set as the default ingress controller then AKO will sync only those ingresses which have the ingress class set to “avi”.

If you do not use ingress classes, then keep this knob untouched and AKO will take care of syncing all your ingress objects to Avi.

### L7Settings.l7ShardingScheme

AKO allows two types of sharding logic currently. These are `hostname` based or `namespace` based. The hostname based sharding uses
the hostname specified in the Ingress rules to determine the shard VS number while the `namespace` based sharding logic shards the
ingress object based on the namespace on which it is created.

### L4Settings.defaultDomain

If you have multiple sub-domains configured in your Avi cloud, use this knob to specify the default sub-domain.
This is used to generate the FQDN for the Service of type loadbalancer. If unspecified, the behavior works on a sorting logic.
The first sorted sub-domain in chosen, so we recommend using this parameter if you want to be in control of your DNS resolution for service of type LoadBalancer.


### ControllerSettings.controllerVersion

This field is used to specify the Avi controller version. While AKO is backward compatible with most of the 18.2.x Avi controllers,
the tested and preferred controller version is `18.2.8` 

### ControllerSettings.controllerIP

This field is usually not present in the `value.yaml` by default but can be provided with the `helm` install command to specify
the Avi Controller's IP address. If you are using a containerized deployment of the controller, pls use a fully qualified controler
IP address/FQDN. For example, if the controller is hosted on 8443, then controllerIP should: `x.x.x.x:8443`


### ControllerSettings.cloudName

This field is used to specify the name of the IaaS cloud in Avi controller. For example, if you have the VCenter cloud named as "Demo"
then specify the `name` of the cloud name with this field. This helps AKO determine the IaaS cloud to create the service engines on.


### ControllerSettings.serviceEngineGroupName

The `serviceEngineGroupName` field is used to specify the name of the IaaS cloud's Service Engine Group in Avi controller. This is only applicable for vCenter IaaS cloud in `ClusterIP` mode where POD CIDRs are not routable

Each AKO instance mapped to a unique `serviceEngineGroupName` parameter. This will be used to push the routes of the cluster to reach the POD CIDR's on the Service Engines.

#### AWS and Azure Cloud in NodePort mode of AKO
If the IaaS cloud is Azure then subnet name is specified in `networkName`. Azure IaaS cloud is supported only in `NodePort` mode of AKO.
If the IaaS cloud is AWS then subnet uuid is specified in `networkName`. AWS IaaS cloud is supported only in `NodePort` mode of AKO. 
The `subnetIP` and `subnetPrefix` are not required for AWS and Azure Cloud.

### avicredentials.username and avicredentials.password

The username/passwword of the Avi controller is specified with this flag. The username/password are base64 encoded by helm and a corresponding secret
object is used to maintain the same. Editing this field requires a restart (delete/re-create) of the AKO pod.

### image.repository

If you are using a private container registry and you'd like to override the default dockerhub settings, then this field can be edited
with the private registry name.

### L7Settings.serviceType

This option specifies whether the AKO functions in ClusterIP mode or NodePort mode. By default it is set to `ClusterIP`. Allowed values are `ClusterIP` and `NodePort`.

### nodeSelectorLabels.key and nodeSelectorLabels.value

It might not be desirable to have all the nodes of a kubernetes cluster to participate in becoming server pool members, hence key/value is used as a label based selection on the nodes in kubernetes to participate in NodePort. If key/value are not specified then all nodes are selected.

### persistentVolumeClaim
By default, AKO prints all the logs to stdout. Instead, persistentVolumeClaim(PVC) can be used for publishing logs of AKO pod to a file in PVC. To use this, the user has to create a PVC (and a persistent volume, if required) and specify the name of the PVC as the value of persistentVolumeClaim. 

### podSecurityContext

This can be used to set securityContext of AKO pod, if necessary. For example, in openshift environment, if a persistent storage with hostpath is used for logging, then securityContext must have privileged: true (Reference - https://docs.openshift.com/container-platform/4.4/storage/persistent_storage/persistent-storage-hostpath.html)