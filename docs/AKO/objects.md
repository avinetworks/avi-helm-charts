This document outlines the object translation logic between AKO and the Avi controller. It's assumed that the reader is minimally versed with both
Kubernetes object semantics and the Avi Object semantics. 

### Service of type loadbalancer

AKO creates a Layer 4 virtualservice object in Avi corresponding to a service of type loadbalancer in Kubernetes. Let's take an example of such a service object in Kubernetes:

    apiVersion: v1
    kind: Service
    metadata:
      name: avisvc-lb
      namespace: red
    spec:
      type: LoadBalancer
      ports:
      - port: 80
        targetPort: 8080
        name: eighty
      selector:
        app: avi-server

AKO creates a dedicated virtual service for this object in kubernetes that refers to reserving a virtual IP for it. The layer 4 virtual service uses a pool section logic based on the ports configured on the service of type loadbalancer. In this case, the incoming port is port `80` and hence the virtual service listens on this ports for client requests. AKO selects the pods associated with this service as pool servers associated with the virtualservice.

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

##### Multi-Port Service Support

A kubernetes service can have multiple ports. In order for a `Service` to have multiple ports, kubernetes mandates them to have a `name`.
Users could choose their ingress paths to route traffic to a specific port of the Service using this `name`. In order to understand the utility,
consider the following Service:

        apiVersion: v1
        kind: Service
        metadata:
          labels:
            run: my-svc
        spec:
          ports:
          - name: myport1
            port: 80
            protocol: TCP
            targetPort: 80
          - name: myport2
            port: 8080
            protocol: TCP
            targetPort: 8090
          selector:
            run: my-svc
          sessionAffinity: None
          type: ClusterIP


In order to use this service across 2 paths, with each routing to a different port, the Ingress spec should look like this:

        spec:
          rules:
          - host: myhost.avi.internal
            http:
              paths:
              - backend:
                  serviceName: service1
                  servicePort: myport1
                path: /foo
          - host: myhost.avi.internal
            http:
              paths:
              - backend:
                  serviceName: service1
                  servicePort: myport2
                path: /bar


As you may note that the service ports in case of multi-port `Service` inside the ingress file are `strings` that match the port names of
the `Service`. This is mandatory for this feature to work. 

### AKO created object naming conventions

In the current AKO model, all kubernetes cluster objects are created on the `admin` tenant in Avi. This is true even for multiple kubernetes clusters managed through a single IaaS cloud in Avi (for example - vcenter cloud). This poses a challenge where each VS/Pool/PoolGroup is expected to be unique to ensure no conflicts between similar object types.

AKO uses a combination of elements from each kubernetes objects to create a corresponding object in Avi that is unique for the cluster.

##### L4 VS names

The formula to derive a VirtualService (vsName) is as follows:

    vsName = clusterName + "--" + namespace + "-" + svcName`

`clusterName` is the value specified in values.yaml during install.
`svcName` refers to the service object's name in kubernetes.
`namespace` refers to the namespace on which the service object is created.

##### L4 pool names

The following formula is used to derive the L4 pool names:

    poolname = vsName + "-" + listener_port`

Here the `listener_port` refers to the service port on which the virtualservice listens on. As it can be intepreted that the number of pools will be directly associated with the number of listener ports configured in the kubernetes service object.

##### L4 poolgroup names

The poolgroup name formula for L4 virtualservices is as follows:

    poolgroupname = vsName + "-" + listener_port`


##### Shared VS names

The shared VS names are derived based on a combination of fields to keep it unique per kubernetes cluster. This is the only object in Avi that does not derive it's name from any of the kubernetes objects.

    ShardVSName = clusterName + "--Shared-L7-" + <shardNum>
 
`clusterName` is the value specified in values.yaml during install. "Shared-L7" is a constant identifier for Shared VSes
`shardNum` is the number of the shared VS generated based on either hostname or namespace based shards.

##### Shared VS pool names

The formula to derive the Shared VS pool is as follows:

    poolgroupname = clusterName + "--" + priorityLabel + "-" + namespace + "-" + ingName

Here the `priorityLabel` is a combination of the host/path combination specified in each rule of the kubernetes ingress object. `ingName` refers to the name of the ingress object while `namespace` refers to the namespace on which the ingress object is found in kubernetes.

##### Shared VS poolgroup names

The following is the formula to derive the Shared VS poolgroup name:

    poolgroupname = vsName

Name of the Shared VS Poolgroup is the same as the Shared VS name.

##### SNI child VS names

The SNI child VSes namings vary between different sharding options.

###### Hostname shard

    vsName = clusterName + "--" + sniHostName

###### Namespace shard

    vsName = clusterName + "--" + ingName + "-" + namespace + "-" + secret

The difference in naming is done because with namespace based sharding only one SNI child is created per ingress/per secret object while in hostname based sharding each SNI VS is unique to the hostname specified in the ingress object.

##### SNI pool names

The formula to derive the SNI virtualservice's pools is as follows:

    poolname = clusterName + "--" + namespace + "-" + host + "_" + path + "-" + ingName

Here the `host` and `path` variables denote the secure hosts' hostname and path specified in the ingress object.

##### SNI poolgroup names

The formula to derive the SNI virtualservice's poolgroup is as follows:

    poolgroupname = clusterName + "--" + namespace + "-" + host + "_" + path + "-" + ingName

Some of these naming conventions can be used to debug/derive corresponding Avi object names that could prove as a tool for first level trouble shooting.

## Features in 1.2.1-beta release

### NodePort Mode 

#### Insecure and Secure Ingress/Routes in NodePort mode

    apiVersion: v1
    kind: Service
    metadata:
      name: service1
      namespace: default
    spec:
      type: NodePort
      ports:
      - port: 80
        targetPort: 8080
        nodePort: 31013
        name: eighty
      selector:
        app: avi-server

In `NodePort` mode, Service `service1` should be of type `NodePort`. There is no change in naming of the objects of VS and pool. AKO populates the Pool server object with `node_ip:nodeport`. If there are 3 nodes in the cluster with Internal IP being `10.0.0.100, 10.0.0.101, 10.0.0.102` and assuming that there’s no node label selectors used, AKO populates pool server as: `10.0.0.100:31013, 10.0.0.101:31013, 10.0.0.101:31013`.

If `service1` is of type `ClusterIP` in  NodePort mode. Pool servers will be empty for ingres/route referring to the service.

#### Service of type loadbalancer In NodePort mode

Service of type `LoadBalancer` automatically creates a NodePort. AKO populates the pool server object with `node_ip:nodeport`. 

    apiVersion: v1
    kind: Service
    metadata:
      name: avisvc-lb
      namespace: red
    spec:
      type: LoadBalancer
      ports:
      - port: 80
        targetPort: 8080
        nodePort: 31013
        name: eighty
      selector:
        app: avi-server

In the above example, AKO creates a dedicated virtual service for this object in kubernetes that refers to reserving a virtual IP for it. If there are 3 nodes in the cluster with Internal IP being `10.0.0.100, 10.0.0.101, 10.0.0.102` and assuming that there’s no node label selectors used, AKO populates pool server as: `10.0.0.100:31013, 10.0.0.101:31013, 10.0.0.101:31013`.

### Openshift Route
In Openshift cluster, AKO can be used to configure routes. Ingress configuration is not supported. Currently the shard mode supported for openshift route is hostname. 

### Insecure Route

    apiVersion: v1
    kind: Route
    metadata:
      name: route1
    spec:
      host: routehost1.avi.internal
      path: /foo
      to:
        kind: Service
        name: avisvc1

For insecure route, AKO creates a Shared VS, Poolgroup and Datascript like insecure ingress. For poolName, route configuration differs from ingress configuration. Service name is appended at the end of poolname, as illustrated bellow.

#### Shared VS pool names for route

The formula to derive the Shared VS pool name for route is as follows:

    poolgroupname = clusterName + "--" + priorityLabel + "-" + namespace + "-" + routeName + "-" + serviceName


### Insecure Route with Alternate Backends

A route can also be associated with multiple services denoted by alternateBackends. The requests that are handled by each service is governed by the service weight.



    apiVersion: v1
    kind: Route
    metadata:
      name: route1
    spec:
      host: routehost1.avi.internal
      path: /foo
      to:
        kind: Service
        name: avisvc1
        weight: 20
      alternateBackends:
      - kind: Service
        name: avisvc2
        weight: 10


For each backend of a route, a new pool is added. All such pools are added with same priority label - 'hostname/path'. For the above example, two pools would be added with priority - 'routehost1.avi.internal/foo'. Ratio for a pool is same as the weight specified for the service in the route.


### Secure Route with Edge Termination

    apiVersion: v1
    kind: Route
    metadata:
      name: secure-route1
    spec:
      host: secure1.avi.internal
      path: /bar
      to:
        kind: Service
        name: avisvc1
      tls:
        termination: edge
        key: |-
        -----BEGIN RSA PRIVATE KEY-----
        ...
        ...
        -----END RSA PRIVATE KEY-----

        certificate: |-
        -----BEGIN CERTIFICATE-----
        ...
        ...
        -----END CERTIFICATE-----

Secure route is configured in Avi like secure ingress. An SNI VS is created for each hostname and for each hostpath one poolgroup is created. However, for alternate backends, multiple pools are added in each poolgroup. Also, unlike Secure Ingress, no redirect policy is configured for Secure Route for insecure traffic.

##### SNI pool names for route

The formula to derive the SNI virtualservice's pools for route is as follows:

    poolname = clusterName + "--" + namespace + "-" + host + "_" + path + "-" + routeName + "-" serviceName

