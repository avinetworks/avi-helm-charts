## FAQ

This document answers few of the frequenty asked questions w.r.t AKO.

#### How do I clean up all my configs?

Simply delete the AKO configmap to delete the virtualservices in Avi. Re-create the same configmap to re-create your setup.

#### How do I alter the shard VS number?

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
