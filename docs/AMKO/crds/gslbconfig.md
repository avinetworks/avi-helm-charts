## GSLBConfig for AMKO
A CRD has been provided to add the GSLB configuration. The name of the object is `GSLBConfig` (short name is `gc`) and it has the following parameters:

```yaml
apiVersion: "avilb.k8s.io/v1alpha1"
kind: "GSLBConfig"
metadata:
  name: "gslb-policy-1"
  namespace: "avi-system"
spec:
  gslbLeader:
    credentials: gslb-avi-secret
    controllerVersion: 18.2.7
    controllerIP: 10.79.171.1
  memberClusters:
    - clusterContext: cluster1-admin
    - clusterContext: cluster2-admin
  globalServiceNameSource: HOSTNAME
  domainNames:
    - avi-container-dns.internal
  refreshInterval: 60
  logLevel: "INFO"
```
1. `apiVersion`: The api version for this object has to be `avilb.k8s.io/v1alpha1`.
2. `kind`: the object kind is `GSLBConfig`.
3. `name`: Can be anything, but it has to be specified in the GDP object.
4. `namespace`: By default, this object is supposed to be created in `avi-system`.
5. `gslbLeader.credentials`: A secret object has to be created for (`helm install` does that automatically) the GSLB Leader cluster. The username and password have to be provided as part of this secret object. Refer to `username` and `password` in [parameters](#parameters).
6. `gslbLeader.controllerVersion`: The version of the GSLB leader cluster.
7. `gslbLeader.controllerIP`: The GSLB leader IP address or the hostname along with the port number, if any.
8. `memberClusters`: The kubernetes/openshift cluster contexts which are part of this GSLB cluster. See [here](#Multi-cluster kubeconfig) to create contexts for multiple kubernetes clusters.
9. `globalServiceNameSource`: The basis on which a GSLB service is created and named. For now, the supported type is `HOSTNAME`. This means that the any ingress which shares a host name across clusters will be placed on the same GSLB service.
10. `domainNames`: Supported GSLB subdomains.
11. `refreshInterval`: This is an internal cache refresh time interval, on which syncs up with the AVI objects and checks if a sync is required.
12. `logLevel`: Define the log level that the amko pod prints. The allowed levels are: `[INFO, DEBUG, WARN, ERROR]`.

### Notes
* Only one GSLBConfig object is allowed.
* If using `helm install`, the GSLB config object will be created for you, just provide the right parameters in `values.yml`.

### Editability
The only field that's editable today is the `logLevel` field. What this means is, a user can just change the logLevel in the runtime and the changes will take effect.

All other fields are non-editable. Once `GSLBConfig` object is defined and is accepted, it can't be changed (as of now). If changed, the changes will not take any effect. For the changes to take effect, the user has to restart the AMKO pod.
