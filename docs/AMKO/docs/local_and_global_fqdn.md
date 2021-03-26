# Deriving GslbService FQDNs

## FQDN Modes
AMKO decides the FQDNs for the GslbServices based on two modes:
1. Default Mode: Here, the hostname fields in the ingress, routes and service type load balancer become the deciding factor. If an ingress gets added with a hostname (and if its also selected via the GDP object), AMKO takes the hostname(s) present in the status field of the ingress and creates GslbService(s) with the respective hostnames. In this mode, no other objects are used to derive the mapping.

2. Custom Global Fqdn Mode: In this mode, AMKO checks the AKO HostRules to figure out the GslbService Fqdn. To expose an application via GSLB, the user must provide a mapping between the local fqdn and the global fqdn via AKO's HostRule object. See [this](https://github.com/avinetworks/avi-helm-charts/blob/master/docs/AKO/crds/hostrule.md#configure-gslb-fqdn) to configure a mapping between the local FQDN and a global FQDN. If the user wishes to map two application instances (which have different FQDNs) in two different clusters, they have to create HostRules in both clusters specifying the mapping.

![Alt text](../images/local_vs_global_fqdn.png?raw=true "local and global fqdn modes")

## How to specify the custom global fqdn mode
The user has to set the `useCustomGlobalFqdn` to `true` in the `GSLBConfig` object. This is a static operation and if changed while AMKO is already deployed, the changes won't take any effect.

## When to use the default mode
![Alt text](../images/local_fqdn.png?raw=true "local fqdn usage")

As mentioned, in the default mode, AMKO just looks at the ingress, routes and LB services and their hostnames to derive the FQDNs for the GslbServices. Users will find it useful when they have two instances of an application deployed on two different clusters which have the same FQDNs. And, they would want to expose all these applications to be exposed as GslbServices in one shot.

## When to use Custom Global Fqdn mode
![Alt text](../images/global_fqdn.png?raw=true "global fqdn usage")

For users who have specific site local FQDNs for their applications and these FQDNs are not common for applications deployed on two different clusters (sites), they can deploy AMKO in Custom Global Fqdn Mode. Users must create HostRule mapping their local FQDNs to a global FQDN in their respective clusters. AMKO then patches both the applications to the same GslbService with global FQDN.