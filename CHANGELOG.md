# Change log:

All notable changes to this project will be documented in this file. The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## AKO-0.9.1

### Changed:
 - SNI naming.  Requires full deletion of the SNI VS names in beta-1.
 - Retry for status updates for ingress.
 - Log levels. Transitioned some logs from INFO —> DEBUG.
 - Logging mechanism. Uses Uber-zap now.
 - Removed SDK logging.
 - Caching improvements. Fixes in race conditions.
 - Reduction of controller API calls during full sync.
 - Full sync fixes.
 - Enchanced Retry logic.
 - Removal of regular object cache syncs - only periodic refresh of cloud config parameters.
 
 
### Added:
 - Dynamic logging on the fly by editing the ConfigMap.
 - AKO API server - for liveness probe and basic controller debugging.
 - SNI VS sharing on the basis of hostnames. Same hostname will create only 1 VS across namespaces.
 - Option to disable full sync. Change fullSyncFrequency to 0.
 - Unused shared VS deletion on reboot of AKO.
 - Multiple sub-domain support with a specification of default sub-domain in `values.yaml` for service of type LB.
 
 ### Removed:
 - VRF context is now removed from `values.yaml` and instead is read from the network subnet.


## AKO-1.1.1

### Added:
 - HostRule/HTTPRule support for Kubernetes


## AKO-1.2.1

### Changed:
 - Liveness probe enhancements.
 - Stability fixes around pod restarts.
 - Retry layer improvements.
 - Cleanup fixes.
 - SDK bug fixes.
 - Logging improvements.

### Added:
 - Full OpenShift 4.x support for NodePort and ClusterIP
 - Per Cluster SE group support. Label based routing support.
 - NodePort Support for Kubernetes.
 - HostRule/HTTPRule support for Openshift.
 - Minimal public cloud support.

 ### Removed:
 - VRF context support deprecated.
 
 ## AKO-1.3.1

### Changed:
 - AKO support for IPAM without specification of networkName.
 - AKO support for controller credential change.

### Added:
 - AKO tenancy support. 
 - AKO operator feature.
 - AKO public cloud with ClusterIP support for GCP/Azure.
 - AKO support for GKE/AKS/EKS.
 - AKO selective namespace sync for Ingress/Route.
 - AKO support for static IP using LoadbalancerIP for L4.
 - OpenShift wildcard certificate support.
 - Global RHI support.
 - AKO support for avi controller object deletion updates via statefulset conditions.
 - AKO support for multiple new fields in HTTPRule/HostRule CRD.
 - Tolerance support for networking/v1 Ingress in k8s 1.19
 
