## FAQ

This document answers few of the frequenty asked questions w.r.t AKO.

#### How do i clean up all my configs?

Simply delete the AKO configmap to delete the virtualservices in Avi. Re-create the same configmap to re-create your setup.

#### How do i alter the shard VS number?

Altering the shard VS number is considered as disruptive. This is because dynamic re-adjustment of shard numbers may re-balance
the ingress to VS mapping. Hence if you want to alter the shard VS number, first delete the older configmap and trigger a complete
cleanup of the VSes in the controller. Followed by a edit of the configmap and restart of AKO.

#### What happens if i have the same SNI host across multiple namespaces?

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

Then the behavior of the SNI virtualservice would be indeterministic since the secrets for the same SNI are different. This is not
supported.

#### What out of band operations can i do on the objects created by AKO?

AKO runs a refresh cycle that currently just refreshes the cloud object parameters. However, if some out of band operations are performed on objects created by AKO via directly interacting with the Avi APIs, AKO may not always be able to remediate
an error caused due to this.

AKO has a best effort, retry layer implementation that would try to detect a problem (For example: an SNI VS deleted from the Avi UI), but it is not guranteed to work for all such manual operations.

Upon reboot of AKO - a full reconcilliation loop is run and  most of the out-of-band changes are overwritten with AKO's view of the intended model. This does not happen in every full sync cycle.

