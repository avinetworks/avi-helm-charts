### HostRule 

HostRule CRD is primarily targeted to be used by the Operator. This CRD can be used to express additional virtual host
properties. The virtual host FQDN is matched from either Kubernetes Ingress or OpenShift Route based objects. 

A sample HostRule CRD looks like this:

    apiVersion: ako.k8s.io/v1alpha1
    kind: HostRule
    metadata:
       name: my-host-rule
       namespace: red
    spec:
      virtualhost:
        fqdn: foo.com # mandatory
        tls: # optional
          sslKeyCertificate:
           name:"avi-ssl-key-cert"
           type: ref
          termination: edge
        httpPolicy: 
          policySets:
          - "my-secure-policy-ref"
          overwrite: false
        networkSecurityPolicy: "nspolicy"
        wafPolicy: "my-waf-policy"
        applicationProfile: "my-app-ref"


### Specific usage of HostRule CRD

HostRule CRD can be created in a given namespace where the operator desires to have more control. 
The section below walks over the details and associated rules of using each field of the HostRule CRD.

#### Express httpolicyset object refs.

HostRule CRD can be used to express httppolicyset references. These httppolicyset objects should be pre-created in the Avi controller.

        httpPolicy: 
          policySets:
          - "my-secure-policy-ref"
          overwrite: false

The httppolicyset currently is only applicable for secure FQDNs and cannot be applied for insecure routes.
The order of evaluation of the httpolicyset rules is in the same order they appear in the CRD definition. The list of httpolicyset rules are
always intepreted as an `AND` operation.

AKO currently uses httppolicyset objects on the SNI virtualservices to route traffic based on host/path matches. These rules are always at
a lower index than the httppolicyset objects specified in the CRD object. In case, a user would want to overwrite all httppolicyset objects
on a SNI virtualservice with the ones specified in the HostRule CRD, the `overwrite` flag can be set to `true`.


#### Express WAF policy object refs.

HostRule CRD can be used to express WAF policy references. The WAF policy object should have been created in the Avi Controller prior to this
CRD creation.

        wafPolicy: "my-waf-policy"

 This property can be applied only for secure FQDNs and cannot be applied for insecure routes.
 WAF policies are useful when deep layer 7 packet filtering is required.
 
#### Express custom application profiles

HostRule CRD can be used to express application profile references. The application profile reference should have been created in the Avi Controller 
prior to this CRD creation. The application profile should be of `TYPE` of `APPLICATION_PROFILE_TYPE_HTTP`.

        applicationProfile: "my-app-ref"
 
 This property can be applied only for secure FQDNs and cannot be applied for insecure routes.
 The application profiles can be used for various HTTP/HTTP2 protocol settings.

#### Express SSL Key and Certs

If the kubernetes operator wants to control the TLS termination from a privileged namespace then the HostRule CRD can be created in such a namespace.

        tls:
          sslKeyCertificate:
           name:"avi-ssl-key-cert"
           type: ref
          termination: edge

The `name` field refers to an Avi object if `type` specifies the value as `ref`. Alternatively in the future, we will be able to support a kubernetes
`Secret` as a `type` where the sslkeyandcertificate object can be created by AKO using the Secret. 

Currently only one of type of termination is supported viz. `edge`. In the future, we should be able to support other types of termination policies.


#### Status Messages

The status messages are used to give instanteneous feedback to the users about the reference objects specified in the HostRule CRD.

Following are some of the sample status messages:

##### Accepted HostRule object

    $ kubectl get hr
    NAME                 HOST                     STATUS     AGE
    secure-waf-policy    myhost273.avi.internal   Accepted   3d3h

A HostRule is accepted only when all the reference objects specified inside it exist in the Avi Controller.

##### A Rejected HostRule object

    $ kubectl get hr
    NAME                     HOST                     STATUS     AGE
    secure-waf-policy-alt    myhost273.avi.internal   Rejected   2d23h
    
The detailed rejection reason can be obtained from the status:

    status:
    error: duplicate fqdn myhost273.avi.internal found in default/secure-waf-policy
    status: Rejected
    
#### Conditions and Caveats

##### Converting insecure FQDNs to secure

The HostRule CRD can be used to convert an insecure host fqdn to a secure one. This is done by specifying a `tls` section in the CRD object.
Whatever `sslKeyCertificate` is provided for the FQDN, will override all sslkeyandcertificates generated for the FQDN. This maybe useful if:

* The operator wants to convert an insecure ingress FQDN to secure.

* The operator wants to override any existing secrets for a given host fqdn and define tls termination semantics. 

##### HostRule deletion

If a HostRule is deleted, all the settings for the FQDNs are withdrawn from the Avi controller.

##### HostRule admission

A HostRule CRD is only admitted if all the objects referenced in it, exist in the Avi Controller. If after admission the object references are
deleted out-of-band, then AKO does not re-validate the associated HostRule CRD objects. The user needs to manually edit or delete the object
for new changes to take effect.

##### Duplicate FQDN rules

Two HostRule CRDs cannot be used for the same FQDN information across namespaces. 
