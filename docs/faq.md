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

`Ingress 1 (default namespace) --> SNI hostname --> foo.com path: /foo, Secret: foo

Ingress 1 (foo namespace) --> SNI hostname --> foo.com path: /bar, Secret: foo`

In the above case, only 1 SNI virtualservice will be created with a sslkeyandcertificate as `foo`.

However if the following happens:

`Ingress 1 (default namespace) --> SNI hostname --> foo.com path: /foo, Secret: foo

Ingress 1 (foo namespace) --> SNI hostname --> foo.com path: /bar, Secret: bar`

Then the behavior of the SNI virtualservice would be indeterministic since the secrets for the same SNI are different. This is not
supported.

