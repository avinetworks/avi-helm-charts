# AKO: Avi Kubernetes Operator

### Run AKO

AKO runs as a POD inside the kubernetes cluster.

 ![Alt text](Ako_k8s.png?raw=true "Title")
 
#### Pre-requisites

To Run AKO you need the following pre-requisites:
 - ***Step 1***: Configure an Avi Controller with a vCenter [cloud](https://avinetworks.com/docs/18.2/installing-avi-vantage-for-vmware-vcenter/). The Avi Controller should be versioned 18.2.10 or later. 

 - ***Step 2***: 
     - Make sure a PG network is part of the NS IPAM configured in the vCenter 

 - ***Step 3***: If your POD CIDRs are not routable:
    - Create a Service Engine Group dedicated for kubernetes cluster.
    - Routes to a given Kubernetes cluster is tagged with the `clusterName` which is specified by the user in `values.yaml`.
    
    Data path flow is as described below:
    
    ![Alt text](data_path_flow.png?raw=true "Title")
    
    The markers in the drawing are described below:
    
    1. Client makes a request to a specified hostname/path.
    2. The DNS VS returns an IP address corresponding to the hostname. 
    3. The request is forwarded to the resolved IP address that corresponds to a Virtual IP hosted on an Avi Service Engine.
       The destination IP in the packet is set as the POD IP address on which the application runs.
    4. Service Engines use the static route information to reach the POD IP via the next hop address of the host on which the pod is running.
    5. The pod responds and the request is sent back to the client. 
      
 - ***Step 3.1***: If your POD CIDRs are routable then you can skip step 2. Ensure that you skip static route syncing in this case using the `disableStaticRouteSync` flag in the `values.yaml` of your helm chart.
 - ***Step 4:*** Kubernetes 1.14+.
 - ***Step 5:*** `helm` cli pointing to your kubernetes cluster.
 NOTE: We only support `helm 3`

#### Install using *helm*

  Step 1: Create the `avi-system` namespace:

    kubectl create ns avi-system

  Step 2: Add this repository to your helm CLI
    
    helm repo add ako https://avinetworks.github.io/avi-helm-charts/charts/stable/ako

Use the `values.yaml` from this repository to edit values related to Avi configuration. Values and their corresponding index can be found [here](#parameters) 

  Step 3: Search the available charts for AKO

    helm search repo

    NAME                 	CHART VERSION	APP VERSION	DESCRIPTION
    ako/ako              	1.1.1        	1.1.1      	A helm chart for Avi Kubernetes Operator
    

 Step 4: Install AKO

    helm install  ako/ako  --generate-name --version 1.1.1 -f values.yaml  --set configs.controllerIP=<controller_ip> --set avicredentials.username=<avi-ctrl-username> --set avicredentials.password=<avi-ctrl-password> --namespace=avi-system
    
    NAME: ako-1593523840
    LAST DEPLOYED: Tue Jun 30 19:00:44 2020
    NAMESPACE: avi-system
    STATUS: deployed
    REVISION: 1
    NOTES:
    1. Get the application URL by running these commands:
      export POD_NAME=$(kubectl get pods --namespace avi-system -l "app.kubernetes.io/name=ako,app.kubernetes.io/instance=ako-1593523840" -o jsonpath="{.items[0].metadata.name}")
      echo "Visit http://127.0.0.1:8080 to use your application"
      kubectl --namespace avi-system port-forward $POD_NAME 8080:80
      
  Step 5: Check the installation
  
    helm list -n avi-system
    
    NAME          	NAMESPACE 	REVISION	UPDATED                             	STATUS  	CHART    	APP VERSION
    ako-1593523840	avi-system	1       	2020-06-30 19:00:44.134075 +0530 IST	deployed	ako-1.1.1	    1.1.1




#### Uninstall using *helm*

Simply run:


*Step1:*

    helm delete <ako-release-name> -n avi-system
    
 Note: the ako-release-name is obtained by doing helm list as shown in the previous step,
 
*Step 2:* 

    kubectl delete ns avi-system
    
#### Upgrade AKO using *helm*

If you are upgrading from an older AKO release then simply run the helm upgrade command. For example (assuming you are upgrading to 1.1.1):

*Step1*

    helm list -n avi-system
    
    NAME          	NAMESPACE 	REVISION	UPDATED                             	STATUS  	CHART    	APP VERSION
    ako-1593523840	avi-system	1       	2020-06-30 19:00:44.134075 +0530 IST	deployed	ako-0.9.1	    0.9.1

*Step2*

    helm upgrade ako-1593523840 ako/ako -f values.yaml --version 1.1.1 --set configs.controllerIP=<IP> --set avicredentials.password=<username> --set avicredentials.username=<username> --namespace=avi-system
    

## Parameters


The following table lists the configurable parameters of the AKO chart and their default values. Please refer to this link for more details on [each parameter](values.md).

| **Parameter** | **Description** | **Default** |
| --- | --- | --- |
| `configs.controllerVersion` | Avi Controller version | 18.2.8 |
| `configs.controllerIP` | Specify Avi controller IP | `nil` |
| `configs.shardVSSize` | Shard VS size enum values: LARGE, MEDIUM, SMALL | LARGE |
| `configs.fullSyncFrequency` | Full sync frequency | 1800 |
| `configs.ingressApi` | Support for default ingress API | corev1 |
| `configs.defaultIngController` | AKO is the default ingress controller | true |
| `configs.cloudName` | Name of the VCenter cloud managed in Avi | Default-Cloud |
| `configs.serviceEngineGroupName` | Name of the Service Engine Group | Default-Group |
| `configs.nodeNetworkList` | List of Networks and corresponding CIDR mappings for the K8s nodes. | `Empty List` |
| `configs.clusterName` | Unique identifier for the running AKO instance. AKO identifies objects it created on Avi Controller using this param. | **required** |
| `configs.subnetIP` | Subnet IP of the data network | **required** |
| `configs.subnetPrefix` | Subnet Prefix of the data network | **required** |
| `configs.networkName` | Network Name of the data network | **required** |
| `configs.defaultDomain` | Specify a default sub-domain for L4 LB services | First domainname found in cloud's dnsprofile |
| `configs.l7ShardingScheme` | Sharding scheme enum values: hostname, namespace | hostname |
| `configs.cniPlugin` | CNI Plugin being used in kubernetes cluster. Specify one of: calico, canal, flannel | **required** for calico setups |
| `configs.logLevel` | logLevel enum values: INFO, DEBUG, WARN, ERROR. logLevel can be changed dynamically from the configmap | INFO |
| `configs.deleteConfig` | set to true if user wants to delete AKO created objects from Avi. deleteConfig can be changed dynamically from the configmap | false |
| `configs.disableStaticRouteSync` | Disables static route syncing if set to true | false |
| `avicredentials.username` | Avi controller username | empty |
| `avicredentials.password` | Avi controller password | empty |
| `image.repository` | Specify docker-registry that has the AKO image | avinetworks/ako |

> `networkName`, `subnetIP` and `subnetPrefix` are required fields which are used for allocating VirtualService IP by IPAM Provider module

> Each AKO instance mapped to a given Avi cloud should have a unique clusterName parameter. This would maintain uniqueness of object naming across Kubernetes clusters.

### AKO objects

Please refer to this [page](objects.md) for details on how AKO interprets the kubernetes objects and translates them to Avi objects.

### FAQ

For some frequently asked question refer [here](faq.md) 


## *__Features in 1.2.1-9042-beta-2 Release__*

### AKO in Openshift Cluster

AKO can be used in openshift cluster to configure Routes and Services of type Loadbalancer.

#### Pre-requisites for running AKO in Openhsift Cluster

Follow the steps 1 to 2, given in section [Pre-requisites](https://github.com/avinetworks/avi-helm-charts/tree/master/docs/AKO#pre-requisites). Additionaly following points have to be noted for openshift environment.
1. Make Sure Openshift version is >= 4.4
2. Openshift routes and services of type load balancer are suppported in AKO
3. Ingresses, if created in the openshift cluster won't be handled by AKO.
4. cniPlugin should be set to **openshift**

#### Features of Openshift Route supported in AKO
AKO supports the following Layer 7 functions of the OpenShift Route object:
1. Insecure Routes.
2. Insecure Routes with alternate backends.
3. Secure routes with edge termination policy.
4. Secure Routes with InsecureEdgeterminationPolicy - Redirect or Allow.
5. Secure Routes of type passthrough
6. Secure Routes with reencrypt functionality

### AWS and Azure IaaS Cloud in NodePort mode of AKO
Supports AWS and Azure IaaS cloud with Avi DNS.

#### Installing Beta Release:
A beta release uses the incubator repository instead of stable. To install a beta release of AKO follow these steps:

Step 1: Create the avi-system namespace

    kubectl create ns avi-system

Step 2: Add ako incubator repository
    
    helm repo add ako-incubator https://avinetworks.github.io/avi-helm-charts/charts/incubator/ako


Step 3: Search for available charts

    helm search repo --devel

    NAME             	CHART VERSION    	APP VERSION      	DESCRIPTION
    ako-incubator/ako	1.2.1-9042-beta-2	1.2.1-9042-beta-2	A helm chart for Avi Kubernetes Operator


Step 4: Install AKO

    helm install  ako-incubator/ako  --generate-name --version 1.2.1-9042-beta-2 -f values.yaml  --set configs.controllerIP=<avi-controller-ip> --set avicredentials.username=<avi-ctrl-username> --set avicredentials.password=<avi-ctrl-password> --namespace=avi-system

Step 5: Check the installation

    helm ls -n avi-system

    NAME           	NAMESPACE 	REVISION	UPDATED                                	STATUS  	CHART                	APP VERSION
    ako-1599666759 	avi-system	1       	2020-09-09 15:52:42.41565938 +0000 UTC 	deployed	ako-1.2.1-9042-beta-2	1.2.1-9042-beta-2

