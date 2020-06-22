## FAQ

This document answers few of the frequenty asked questions w.r.t AKO.

#### How do I clean up all my configs?

Simply delete the AKO configmap to delete the virtualservices in Avi. Re-create the same configmap to re-create your setup.

#### How is the Shared VS lifecycle controlled?


In hostname based sharding, when an ingress object is created with multiple hostnames, AKO generates a md5 hash using the hostname and the Shard VS number. This uniquely maps a FQDN to a given Shared VS and avoids DNS conflicts. During initial clean bootup, if the Shared VS does not exist in Avi - AKO creates the same and then patches the ingress FQDN to it either in the form of a pool (for insecure routes) or in the form of a SNI child virtualservice (in case of secure routes).

The Shared VSes aren't deleted if all the FQDNs mapped to it are removed from kubernetes. However, if the user wants AKO to delete ununsed shared VSes - a pod restart is required that would evaluate the VS and delete it appropriately. 


#### How do I alter the Shard VS number?

Altering the shard VS number is considered as disruptive. This is because dynamic re-adjustment of shard numbers may re-balance
the ingress to VS mapping. Hence if you want to alter the shard VS number, first delete the older configmap and trigger a complete
cleanup of the VSes in the controller. Followed by a edit of the configmap and restart of AKO.

#### What is the use of static routes?

Static routes are created within a VRF context. While deploying AKO the admin or the operator decides a VRF context object for a given
kubernetes cluster. This VRF context object is created in the Avi controller for the cluster. 
The static routes map each POD CIDR with the kubernetes node's IP address. However, for static routes to work, the Service Engines must
be L2 adjacent to your kubernetes nodes.


#### What happens if I have the same SNI host across multiple namespaces?

The ingress API does not prohibit the user from creating the same SNI hostname across multiple namespaces. In the hostname sharding
mode, AKO will create 1 SNI virtualservice and gather all paths associated with it across namespaces to create corresponding switching
rules. However, the user needs to denote each ingress with the TLS secret for a given hostname to qualify the host for the SNI virtualservice.

Consider the below example:

    Ingress 1 (default namespace) --> SNI hostname --> foo.com path: /foo, Secret: foo

    Ingress 1 (foo namespace) --> SNI hostname --> foo.com path: /bar, Secret: foo

In the above case, only 1 SNI virtualservice will be created with a sslkeyandcertificate as `foo`.

However if the following happens:

    Ingress 1 (default namespace) --> SNI hostname --> foo.com path: /foo, Secret: foo

    Ingress 1 (foo namespace) --> SNI hostname --> foo.com path: /bar, Secret: bar

Then the behavior of the SNI virtualservice would be indeterministic since the secrets for the same SNI are different. This is not supported.

#### What out of band operations can I do on the objects created by AKO?

AKO runs a refresh cycle that currently just refreshes the cloud object parameters. However, if some out of band operations are performed on objects created by AKO via directly interacting with the Avi APIs, AKO may not always be able to remediate
an error caused due to this.

AKO has a best effort, retry layer implementation that would try to detect a problem (For example: an SNI VS deleted from the Avi UI), but it is not guranteed to work for all such manual operations.

Upon reboot of AKO - a full reconcilliation loop is run and  most of the out-of-band changes are overwritten with AKO's view of the intended model. This does not happen in every full sync cycle.

#### What is the expected behaviour for same host/path combination across different secure/insecure ingresses?

The ingress API allows users to add duplicate hostpaths bound to separate backend services. Something like this:

    Ingress1 (default namespace) --> foo.com path: /foo, Service: svc1

    Ingress2 (default namespace) --> foo.com path: /foo, Service: svc2

Also, ingress allows you to have a mix of secure and insecure hostpath bound to same backend services like so:

    Ingress1 (default namespace) --> SNI hostname --> foo.com path: /foo, Secret: secret1

    Ingress2 (default namespace) --> foo.com path: /foo, Service: svc2

AKO doesnot explicitly handle these conditions and would continue syncing these objects on the Avi controller, but this may lead to traffic issues.
AKO does a best effort of detecting some of these conditions by printing them in logs. A sample log statement looks like this:

`key: Ingress/default/ingress2, msg: Duplicate entries found for hostpath default/ingress2: foo.com/foo in ingresses: ["default/ingress1"]`

#### What happens to static routes if the kubernetes nodes are rebooted/shutdown?

AKO programs a static route for every node IP and the POD CIDR associated with it. Even though node state changes to `NotReady` in kubernetes this configuration is stored in the node object and does not change when the node rebooted/shutdown.

Hence AKO will not remove the static routes until the kubernetes node is completely removed from the cluster.

#### Can I point my ingress objects to a service of type Loadbalancer?

The short answer is No. 
The ingress objects should point to service of type clusterIP. Loadbalancer services either point to an ingress controller POD if one is using an in cluster ingress controller or they can directly point to application PODs that need layer 4 loadbalancing.

If you have such a configuration where the ingress objects are pointing to services of type loadbalancer, AKO's behavior would be indeterministic. 

#### What happenns when AKO fails to connect to AVI controller while booting up ?

AKO would stop processing kubernetes objects and no update would be made to the AVI Controller. After the connection to AVI Controller is restored, AKO pod has to be rebooted. This can be done by deleting the exiting POD and ako deployment would bring up a new POD, which would start procesing kubernetes objects after verifying connectivity to AVI Controller.  
