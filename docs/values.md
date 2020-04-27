## Description of tunables of AKO

This document is intended to help the operator make the right choices while deploying AKO with the configurable settings.
The values.yaml in AKO affects a configmap that AKO's deployment reads to make adjustments as per user needs. Listed are detailed
explanation of various fields specified in the values.yaml.

### configs.controllerVersion

This field is used to specify the Avi controller version. While AKO is backward compatible with most of the 18.2.x Avi controllers,
the tested and preferred controller version is `18.2.8` 

### configs.controllerIP

This field is usually not present in the `value.yaml` by default but can be provided with the `helm` install command to specify
the Avi Controller's IP address. If you are using a containerized deployment of the controller, pls use a fully qualified controler
IP address/FQDN. For example, if the controller is hosted on 8443, then controllerIP should: `x.x.x.x:8443`

### configs.shardVSSize

AKO uses a sharding logic for Layer 7 ingress objects. A sharded VS involves hosting multiple insecure or secure ingresses hosted by
one virtual IP or VIP. Having a shared virtual IP allows lesser IP usage since reserving IP addresses particularly in public clouds
incur greater cost. 

#### How are VSes sharded?

If you create an ingress with an insecure host/path combination then AKO creates a corresponding Avi Pool object and patches the pool
on one of the existing shard virtual services. The shard VS has a datascript associated with it that reads the host/path of the incoming
request and appropriately selects a pool by matching it with the priority label specified for each pool member (corresponding to a host/path
combination).

For secure ingresses, a SNI virtual service is created which although is a dedicated virtualservice, does not have any IP addresses
associated with it. The SNI virtual service is a child to a parent virtual service and is created based on the secret object specified
in the ingress file against the host/path that is meant to be accessed securely.

#### How do you decide the Shard VS size?

In the current AKO model, the Shard VS size is an enum. It allows 3 pre-fixed set of values viz. `LARGE`, `MEDIUM` and `SMALL`. They
respectively correspond to 8, 4 and 1 virtual service. The decision of selecting one of these sizes for Shard VS is driven by the
size of the kubernetes cluster's ingress requirements. Typically, it's advised to always go with the highest possible Shard VS number
that is - `LARGE` to account for future expansion. 

#### Can I change the Shard VS number?

In order to Shard to virtual services, AKO uses a sharding mechanism that is driven either by the `namespace` on which the ingress object
is created or the `hostname` of each individual rule within an ingress object. The latter is marked as default because it ensures that a unique
hostname is always sharded consistently to the same virtual service. 

Since the sharding logics are determined by the number of Shard virtualservices, changing the Shard VS number has the potential hazard
of messing up an existing cluster's already synced objects. Hence it's recommended that the Shard VS numbers are not changed once fixed.


### configs.fullSyncFrequency

This field is used to set a frequency of consitency checks in AKO. Typically inconsistent states can arise if users make changes out
of band w.r.t AKO. For example, a pool is deleted by the user from the UI of the Avi Controller. The full sync frequency is used
to ensure that the models are re-conciled and the corresponding Avi objects are restored to the original state. 

### configs.ingressApi

If your kubernetes APIs don't support the newly introduced ingress `networking.k8s.io/v1beta1` Api version but instead only supports
the deprecated (1.16 onwards) `extensionv1` APIs - then this field can be used to tune AKO accordingly. This feature will be soon
deprecated in AKO as well as the customer based move towards newer kubernetes version 1.16 and beyond.

### configs.defaultIngController

This field is related to the ingress class support in AKO specified via `kubernetes.io/ingress.class` annotation specified on an
ingress object.

- If AKO is set as the default ingress controller, then it will sync everything other than the ones on which the ingress class is specified and is not equals to “avi”.
- If Avi is not set as the default ingress controller then AKO will sync only those ingresses which have the ingress class set to “avi”.

If you do not use ingress classes, then keep this knob untouched and AKO will take care of syncing all your ingress objects to Avi.

### configs.l7ShardingScheme

AKO allows two types of sharding logic currently. These are `hostname` based or `namespace` based. The hostname based sharding uses
the hostname specified in the Ingress rules to determine the shard VS number while the `namespace` based sharding logic shards the
ingress object based on the namespace on which it is created.

### configs.cniPlugin

Use this flag only if you are using calico as a CNI and you are looking to a sync your static route configurations automatically.
Once enabled, this flag is used to read the `blockaffinity` CRD in calico to determine the POD CIDR to Node IP mappings. If you are
on an older version of calico where `blockaffinity` is not present, then leave this field as blank. AKO will then determine the static
routes based on the Kubernetes Nodes object as done with other CNIs.

### configs.cloudName

This field is used to specify the name of the IaaS cloud in Avi controller. For example, if you have the VCenter cloud named as "Demo"
then specify the `name` of the cloud name with this field. This helps AKO determine the IaaS cloud to create the service engines on.

### configs.subnetIP and configs.subnetPrefix and configs.networkName

AKO supports dual arm deployment where the Virtual IP network can be on a different subnet than the actual Port Groups on which the kubernetes nodes are deployed.

These fields are used to specify the Virtual IP network details on which the user wants to place the Avi virtual services on.
   
### configs.vrfRefName

#### What is the use of static routes?

Static routes are created within a VRF context. While deploying AKO the admin or the operator decides a VRF context object for a given
kubernetes cluster. This VRF context object is created in the Avi controller for the cluster. The VRF contexts are associated with a 
Port Group in VCenter and the same Port Group (PG) is used by the Service Engines to communicate with the backend servers. 
Hence the static routes map each POD CIDR with the kubernetes node's IP address. However, for static routes to work, the Service Engines must
be L2 adjacent to your kubernetes nodes.

#### Use of the flag

AKO needs to know the VRF context to use for the virtual services associated with a kubernetes cluster. This flag is provided to
the AKO operator to specify the same.

### configs.disableStaticRouteSync

This flag can be used in 2 scenarios:
 - If your POD CIDRs are routable either through an internal implementation or by default.
 - If you are working with multiple NICs on your kubernetes worker nodes and the default gateway is not from the same subnet as
   your VRF's PG network.

### avicredentials.username and avicredentials.password

The username/passwword of the Avi controller is specified with this flag. The username/password are base64 encoded by helm and a corresponding secret
object is used to maintain the same. Editing this field requires a restart (delete/re-create) of the AKO pod.

### image.repository

If you are using a private container registry and you'd like to override the default dockerhub settings, then this field can be edited
with the private registry name.
