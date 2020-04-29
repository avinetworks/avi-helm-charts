This document outlines the object translation logic between AKO and the Avi controller. It's assumed that the reader is minimally versed with both
Kubernetes object semantics and the Avi Object semantics. 

### Insecure Ingress.

Let's take an example of an insecure hostname specification from a Kubernetes ingress object:

    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
      name: my-ingress
    spec:
      rules:
        - host: myinsecurehost.avi.internal
          http:
            paths:
            - path: /foo
              backend:
                serviceName: service1
                servicePort: 80
              
For insecure host/path combinations, AKO uses a Sharded VS logic where based on either the `namespace` of this ingress or the `hostname`
value (`myhost.avi.internal`), a pool object is created on a Shared VS. A shared VS typically denotes a virtualservice in Avi that
is shared across multiple ingresses. A priority label is associated on the poolgroup against it's member pool (that is created as a part of
this ingress), with priority label of `myhost.avi.internal/foo`.

An associated datascript object with this shared virtual service is used to interpret the host fqdn/path combination of the incoming
request and the corresponding pool is chosen based on the priority label as mentioned above.

The paths specified are interpreted as `STARTSWITH` checks. This means for this particular host/path if pool X is created then, the matchrule can
be interpreted as - If the host header equals `myhost.avi.internal` and path `STARTSWITH` `foo` then route the request to pool X.

### Secure Ingress

Let's take an example of a secure ingress object:

    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
      name: my-ingress
    spec:
      tls:
      - hosts:
        - myhost.avi.internal
        secretName: testsecret-tls
      rules:
        - host: myhost.avi.internal
          http:
            paths:
            - path: /foo
              backend:
                serviceName: service1
                servicePort: 80

##### SNI VS per secure hostname

AKO creates an SNI child VS to a parent shared VS for the secure hostname. The SNI VS is used to bind the hostname to an sslkeycert object.
The sslkeycert object is used to terminate the secure traffic on Avi's service engine. In the above example the `secretName` field denotes the
secret asssociated with the hostname `myhost.avi.internal`. AKO parses the attached secret object and appropriately creates the sslkeycert
object in Avi. The SNI virtualservice does not get created if the secret object does not exist in Kubernetes corresponding to the
reference specified in the ingress object.

##### Traffic routing post SSL termination

On the SNI VS, AKO creates httppolicyset rules to route the terminated (insecure) traffic to the appropriate pool object using the host/path
specified in the `rules` section of this ingress object.

##### Redirect secure hosts from http to https

Additionally - for these hostnames, AKO creates a redirect policy on the shared VS (parent to the SNI child) for this specific secure hostname.
This allows the client to automatically redirect the http requests to https if they are accessed on the insecure port (80).

