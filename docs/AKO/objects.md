This document outlines the object translation logic between AKO and the Avi controller. It's assumed that the reader is minimally versed with both
Kubernetes object semantics and the Avi Object semantics.

### Service of type loadbalancer

AKO creates a Layer 4 virtualservice object in Avi corresponding to a service of type loadbalancer in Kubernetes. Let's take an example of such a service object in Kubernetes:

```
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
```

AKO creates a dedicated virtual service for this object in kubernetes that refers to reserving a virtual IP for it. The layer 4 virtual service uses a pool section logic based on the ports configured on the service of type loadbalancer. In this case, the incoming port is port `80` and hence the virtual service listens on this ports for client requests. AKO selects the pods associated with this service as pool servers associated with the virtualservice.

#### Service of type loadbalancer with preferred IP

Kubernetes' service objects allow controllers/cloud providers to create services with user-specified IPs using the `loadBalancerIP` field. AKO supports the `loadBalancerIP` field usage where-in the corresponding Layer 4 virtualservice objects are created with the user provided IP. Example usage for this could look something like this:

```
apiVersion: v1
kind: Service
metadata:
  name: avisvc-lb
  namespace: red
spec:
  loadBalancerIP: 10.10.10.11
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
    name: eighty
  selector:
    app: avi-server
```

Avi does not allow users to update preferred virtual IPs bound to a particular virtualservice. Therefore in order to update the user preferred IP, it is required to re-create the Service object, failing which Avi/AKO throws an error. The following transition cases should be kept in mind, and for these, an explicit Service re-create with changed configuration is required.
 - updating loadBalancerIP value, from `loadBalancerIP: 10.10.10.11` to `loadBalancerIP: 10.10.10.22`.
 - adding `loadBalancerIP` value after the Service is assigned an IP from Avi.
 - removing `loadBalancerIP` value after the Service is assigned an IP from Avi.
Recreating the Service object deletes the Layer 4 virtualservice in Avi, frees up the applied virtual IP and post that the Service creation with update configuration should result in the intended virtualservice configuration.

### Insecure Ingress.

Let's take an example of an insecure hostname specification from a Kubernetes ingress object:

```
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
```

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

```
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
```

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

```
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
```

In order to use this service across 2 paths, with each routing to a different port, the Ingress spec should look like this:

```
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
```

As you may note that the service ports in case of multi-port `Service` inside the ingress file are `strings` that match the port names of
the `Service`. This is mandatory for this feature to work.

### Namespace Sync in AKO

Namespace Sync feature allows the user to sync Ingresses/Routes from specific namespace/s to Avi controller.

New parameters has been introduced as config options in AKO's values.yaml. To use this feature, set the value of these parametes to a non-empty string.

| **Parameter** | **Description** | **Default** |
| --------- | ----------- | ------- |
| `AKOSettings.namespaceSelector.labelKey` | Key used as a label based selection for the namespaces. | empty |
| `AKOSettings.namespaceSelector.labelValue` | Value used as a label based selection for the namespaces. | empty |

Empty value to any one of the mentioned parameters will result in disabling namespace sync functionality and will result in syncing up ingresses/routes from all namespace with Avi controller. Any changes in values of these parameters will require AKO reboot.

Once user boots up AKO with this setting, user has to label a namespace with same key:value pair mentioned in values of labelKey and labelValues. For example, if user has specified values as labelKey:"app" and labelValue: "migrate" in values.yaml, then user has to label namespace with "app: migrate".

```
    apiVersion: v1
    kind: Namespace
    metadata:
      creationTimestamp: "2020-12-04T13:20:42Z"
      labels:
        app: migrate
      name: red
      resourceVersion: "14055620"
      selfLink: /api/v1/namespaces/red
      uid: a424bf13-2f4a-4005-a84d-f2fb65acfda0
    spec:
      finalizers:
      - kubernetes
    status:
      phase: Active
```

Valid labelling of a namespace will sync ingresses/routes from that namespace with Avi controller.

Ingresses/Routes from namespaces, with no labels or invalid labels, will not be synched with Avi Controller.

If user changes a label of a namespace from valid to invalid, then it will result in pool deletion for an insecure ingress/route and a SNI VS deletion for a secure ingress/route from Avi controller. For example, let say label of 'red' namespace is changed from "app: migrate" (valid) to "app: migrate1" (invalid), then pools associated with, insecure ingresses/routes and SNI VSes associated with secure ingresses/routes of a namespace 'red', will be deleted from Avi Controller.

If user changes label of a namespace from invalid to valid, then it will result in adding ingresses/routes of that namespace with Avi controller.

### AKO created object naming conventions

In the current AKO model, all kubernetes cluster objects are created on the `admin` tenant in Avi. This is true even for multiple kubernetes clusters managed through a single IaaS cloud in Avi (for example - vcenter cloud). This poses a challenge where each VS/Pool/PoolGroup is expected to be unique to ensure no conflicts between similar object types.

AKO uses a combination of elements from each kubernetes objects to create a corresponding object in Avi that is unique for the cluster.

##### L4 VS names

The formula to derive a VirtualService (vsName) is as follows:

```
vsName = clusterName + "--" + namespace + "-" + svcName`
```

`clusterName` is the value specified in values.yaml during install.
`svcName` refers to the service object's name in kubernetes.
`namespace` refers to the namespace on which the service object is created.

##### L4 pool names

The following formula is used to derive the L4 pool names:

```
poolName = vsName + "-" + listener_port`
```

Here the `listener_port` refers to the service port on which the virtualservice listens on. As it can be intepreted that the number of pools will be directly associated with the number of listener ports configured in the kubernetes service object.

##### L4 poolgroup names

The poolgroup name formula for L4 virtualservices is as follows:

```
poolgroupname = vsName + "-" + listener_port`
```

##### Shared VS names

The shared VS names are derived based on a combination of fields to keep it unique per kubernetes cluster. This is the only object in Avi that does not derive it's name from any of the kubernetes objects.

```
ShardVSName = clusterName + "--Shared-L7-" + <shardNum>
```

`clusterName` is the value specified in values.yaml during install. "Shared-L7" is a constant identifier for Shared VSes
`shardNum` is the number of the shared VS generated based on either hostname or namespace based shards.

##### Shared VS pool names

The formula to derive the Shared VS pool is as follows:

```
poolName = clusterName + "--" + priorityLabel + "-" + namespace + "-" + ingName
```

Here the `priorityLabel` is a combination of the host/path combination specified in each rule of the kubernetes ingress object. `ingName` refers to the name of the ingress object while `namespace` refers to the namespace on which the ingress object is found in kubernetes.

##### Shared VS poolgroup names

The following is the formula to derive the Shared VS poolgroup name:

```
poolgroupname = vsName
```

Name of the Shared VS Poolgroup is the same as the Shared VS name.

##### SNI child VS names

The SNI child VSes namings vary between different sharding options.

###### Hostname shard

```
vsName = clusterName + "--" + sniHostName
```

###### Namespace shard

```
vsName = clusterName + "--" + ingName + "-" + namespace + "-" + secret
```

The difference in naming is done because with namespace based sharding only one SNI child is created per ingress/per secret object while in hostname based sharding each SNI VS is unique to the hostname specified in the ingress object.

##### SNI pool names

The formula to derive the SNI virtualservice's pools is as follows:

```
poolName = clusterName + "--" + namespace + "-" + host + "_" + path + "-" + ingName
```

Here the `host` and `path` variables denote the secure hosts' hostname and path specified in the ingress object.

##### SNI poolgroup names

The formula to derive the SNI virtualservice's poolgroup is as follows:

```
poolgroupname = clusterName + "--" + namespace + "-" + host + "_" + path + "-" + ingName
```

Some of these naming conventions can be used to debug/derive corresponding Avi object names that could prove as a tool for first level trouble shooting.

##### Pool pkiprofile names

The formula to derive the pool's PKIprofile is as follows:

```
pkiprofilenam = poolName + "-pkiprofile"
```

### NodePort Mode

#### Insecure and Secure Ingress/Routes in NodePort mode

```
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
```

In `NodePort` mode, Service `service1` should be of type `NodePort`. There is no change in naming of the objects of VS and pool. AKO populates the Pool server object with `node_ip:nodeport`. If there are 3 nodes in the cluster with Internal IP being `10.0.0.100, 10.0.0.101, 10.0.0.102` and assuming that there’s no node label selectors used, AKO populates pool server as: `10.0.0.100:31013, 10.0.0.101:31013, 10.0.0.101:31013`.

If `service1` is of type `ClusterIP` in NodePort mode. Pool servers will be empty for ingres/route referring to the service.

#### Service of type loadbalancer In NodePort mode

Service of type `LoadBalancer` automatically creates a NodePort. AKO populates the pool server object with `node_ip:nodeport`.

```
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
```

In the above example, AKO creates a dedicated virtual service for this object in kubernetes that refers to reserving a virtual IP for it. If there are 3 nodes in the cluster with Internal IP being `10.0.0.100, 10.0.0.101, 10.0.0.102` and assuming that there’s no node label selectors used, AKO populates pool server as: `10.0.0.100:31013, 10.0.0.101:31013, 10.0.0.101:31013`.

For the Openshift objects: [Openshift](https://github.com/avinetworks/avi-helm-charts/tree/master/docs/AKO/openshift/objects.md)
