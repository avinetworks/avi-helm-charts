## GSLBHostRule CRD for AMKO
The `GSLBHostRule` CR allows users to override certain properties of a specific GslbService object on the Avi Controller created by AMKO.

A typical `GSLBHostRule` looks like this:
```yaml
apiVersion: amko.vmware.com/v1alpha1
kind: GSLBHostRule
metadata:
  name: gslb-host-rule-1
  namespace: avi-system
spec:
  fqdn: foo.avi.internal
  sitePersistence:
    enabled: true
    profileRef: "gap-1"  # only enabled for secure ingresses/routes
  thirdPartyMembers:
  - site: non-avi-site
    vip: 10.10.10.10
  healthMonitorRefs:
  - hm1
  - hm2
  trafficSplit:
  - cluster: k8s
    weight: 15
  - cluster: oshift
    weight: 5
  ttl: 30
```
1. `namespace`: namespace of this object must be `avi-system`.

2. `fqdn`: FQDN of the GslbService.

3. `sitePersistence`: Enable Site Persistence for client requests. Set the `enabled` flag as `true` and add a `profileRef` for a pre-created Application Persistence Profile created on the Avi Controller. Please follow the steps [here](https://avinetworks.com/docs/20.1/gslb-site-cookie-persistence/#outline-of-steps-to-be-taken) to create a federated Application Persistence Profile on the Avi Controller.

**Note** that site persistence is **disabled** on GslbServices created for **insecure** ingresses/routes, irrespective of this field.
If this field is not provided in `GSLBHostRule`, the site persistence property will be inherited from the GDP object.

4. `thirdPartyMembers`: To add one or more third party members to a GS from a non-avi site (third party site) for the purpose of maintenance, specify a list of those members. For each member, provide the site name in `site` and IP address in `vip`. Please refer [here](https://avinetworks.com/docs/20.1/gslb-third-party-site-configuration-and-operations/#associating-third-party-services-with-third-party-sites) to see how to add third party sites to existing Gslb configuration. **Note** that, to add third party members, set the `enable` flag in `sitePersistence` to false for this object. If site persistence is enabled for a GSLB Service, third party members can't be added.

**Note** that the site must be added to the GSLB leader as a 3rd party site before adding the member here.

5. `healthMonitorRefs`: If a GslbService requires some custom health monitoring, the user can create a federated custom health monitor in the Avi Controller and provide the ref(s) here. To add a custom health monitor, follow the steps [here](https://avinetworks.com/docs/16.3/avi-gslb-service-and-health-monitors/#configuring-health-monitoring). If no custom health monitor refs have been added, the custom health monitors from the GDP object will be inherited.

6. `trafficSplit`: Specify traffic steering to member clusters/sites. The traffic is then split proportionately between two different clusters. Weight for each cluster must be provided between 1 to 20. If not added, GDP object's traffic split applies on this GslbService.

7. `ttl`: Override the default `ttl` value specified on the GDP object using this field.

## Caveats:
* Site Persistence cannot be enabled for the GslbServices which have insecure ingresses or routes as the members.
