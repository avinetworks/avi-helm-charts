## GlobalDeploymentPolicy CRD for AMKO
A CRD called GlobalDeploymentPolicy allows users to select kubernetes/openshift objects based on certain rules. This GDP object has to be created on the same system wherever the GSLBConfig object was created and `amko` is running. The selection policy applies to all the clusters which are mentioned in the GDP object. A typical GlobalDeploymentPolicy looks like this:

```yaml
apiVersion: "amko.k8s.io/v1alpha1"
kind: "GlobalDeploymentPolicy"
metadata:
  name: "global-gdp"
  namespace: "avi-system"   // a cluster-wide GDP
spec:
  matchRules:
    appSelector:
      label:
        app: gslb
    namespaceSelector:
      label:
        ns: gslb
 
  matchClusters:
    - cluster: cluster1-admin    // cluster names are kubernetes cluster contexts
    - cluster: cluster2-admin
 
  trafficSplit:
    - cluster: cluster1
      weight: 8
    - cluster: cluster2
      weight: 2
```
1. `namespace`: an important piece here, as a GDP object created in `avi-system` namespace is recognised and all other GDP objects created in other namespaces are ignored.
2. `matchRules`: List of selection policy rules. If a user wants to select certain objects in a namespace (mentioned in `namespace`), they have to add those rules here. A typical `matchRule` looks like:
```yaml
matchRules:
    appSelector:                       // application selection criteria
      label:
        app: gslb                       // kubernetes/openshift label key-value
    namespaceSelector:                 // namespace selection criteria
      label:
        ns: gslb                        // kubernetes/openshift label key-value
```
A combination of appSelector and namespaceSelector will decide which objects will be selected for GSLB service consideration.
- appSelector: Selection criteria only for applications:
  * label: will be used to match the ingress/service type load balancer labels (key:value pair).
- namespaceSelector: Selection criteria only for namespaces:
  * label: will be used to match the namespace labels (key:value pair).

AMKO supports the following combinations for GDP matchRules:
| **appSelector** | **namespaceSelector** | **Result**                                                                                         |
| --------------- | --------------------- | -------------------------------------------------------------------------------------------------- |
| yes             | yes                   | Select all objects satisfying appSelector and from the namespaces satisfying the namespaceSelector |
| no              | yes                   | Select all objects from the selected namespaces (satisfying namespaceSelector)                     |
| yes             | no                    | Select all objects satisfying the appSelector criteria from all namespaces                         |
| no              | no                    | No objects selected (default action)                                                               |

Example Scenarios:

> Select objects with label `app:gslb` from all the namespaces:
```yaml
  matchRules:
    appSelector:
      label:
        app: gslb
```

> Select objects with label `app:gslb` and from namespaces labelled `ns:prod`:
```yaml
matchRules:
    appSelector:
      label:
        app: gslb
    namespaceSelector:
      label:
        ns: prod
```

3. `matchClusters`: List of clusters on which the above `matchRules` will be applied on. The member object of this list are cluster contexts of the individual k8s/openshift clusters.

4. `trafficSplit` is required if we want to route a certain percentage of traffic to certain objects in a certain cluster. These are weights and the range for them is 1 to 20.

### Notes
* A GDP object must be created in the `avi-system` namespace. GDP objects in all ther namespaces will *not* be considered. For now, AMKO supports only one GDP object in the entire cluster. Any other additonal GDP objects will be ignored.
* A GDP object is created as part of `helm install`. User can then edit this GDP object to modify their selection of objects.
* Deletion of a GDP rule will trigger all the objects to be again checked against the remaining set of rules.
* Deletion of a cluster member from the `matchClusters` will trigger deletion of objects selected from that cluster in AVI.

### Editability
All fields in the GDP spec are editable. This means that, any GDP change made in runtime *will* be reflected by AMKO.