## GlobalDeploymentPolicy CRD for AMKO
A CRD called GlobalDeploymentPolicy allows users to select kubernetes/openshift objects based on certain rules. The selection policy applies to all the clusters which are mentioned in the GDP object. A typical GlobalDeploymentPolicy looks like this:

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
1. `namespace`: namespace of this object must be `avi-system`.
2. `matchRules`: This allows users to select objects using either application labels (configured as labels on Ingress/Route objects) or via namespace labels (configured as labels on the namespace objects). `matchRules` are defined as:
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

4. `trafficSplit` is required if we want to route a percentage of traffic to objects in a given cluster. Weights for these clusters range from 1 to 20.

### Notes
* Only one `GDP` object is allowed.
* If using `helm install`, a `GDP` object is created by picking up values from `values.yaml` file. User can then edit this GDP object to modify their selection of objects.
