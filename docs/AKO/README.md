# Helm charts by Avi Networks

This repository is used to host helm charts from Avi Networks (A VMware Company)

## Helm Charts for AKO - Avi Kubernetes Operator


### Run AKO

AKO runs as a POD inside the kubernetes cluster.

 ![Alt text](Ako_k8s.png?raw=true "Title")
 
#### Pre-requisites

To Run AKO you need the following pre-requisites:
 - ***Step 1***: Configure an Avi Controller with a vCenter [cloud](https://avinetworks.com/docs/18.2/installing-avi-vantage-for-vmware-vcenter/). 

 - ***Step 2***: 
     - Make sure a PG network is part of the NS IPAM configured in the vCenter 

 - ***Step 3***: If your POD CIDRs are not routable:
    - Create a VRF context object in Avi for the kubernetes controller.
    - Get the name the PG network which the kubernetes nodes are part of. 
    - Configure the PG network in step 2 with the vrf mentioned in the previous step using the Avi CLI.
    
    Data path flow is as described below:
    
    ![Alt text](data_path_flow.png?raw=true "Title")
    
    The markers in the drawing are described below:
    
    1. Client makes a request to a specified hostname/path.
    2. The DNS VS returns an IP address corresponding to the hostname. 
    3. The request is forwarded to the resolved IP address that corresponds to a Virtual IP hosted on an Avi Service Engine.
       The destination IP in the packet is set as the POD IP address on which the application runs.
    4. Service Engines use the static route information to reach the POD IP via the next hop address of the host on which the pod is running.
    5. The pod responds and the request is sent back to the client. 
      
      *NOTE: If you are using AKO for test puposes you can use the `global` vrf but you cannot manage multiple kubernetes clusters in the same cloud with this setting.*

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
    ako/ako              	0.9.1        	0.9.1      	A helm chart for Avi Kubernetes Operator
    

 Step 4: Install AKO

    helm install  ako/ako  --generate-name --version 0.9.1 -f values.yaml  --set configs.controllerIP=<controller_ip> --set avicredentials.username=<avi-ctrl-username> --set avicredentials.password=<avi-ctrl-password> --namespace=avi-system
    
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
    ako-1593523840	avi-system	1       	2020-06-30 19:00:44.134075 +0530 IST	deployed	ako-0.9.1	    0.9.1




#### Uninstall using *helm*

Simply run:


*Step1:*

    helm delete <ako-release-name> -n avi-system
    
 Note: the ako-release-name is obtained by doing helm list as shown in the previous step,
 
*Step 2:* 

    kubectl delete ns avi-system

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

## *__Features in 1.2.1-beta Release__*
> All the features mentioned in the subsequent sections are only part of the beta release and not present in the stable releases.

### Node Port

Service of type node port can be used to send traffic to the pods using nodeports. This can be used where the option of static IP in VRF Context is not feasible.

A new parameter serviceType has been introduced as config option in AKO's values.yaml. To use this feature, set the value of the parameter to **NodePort**.

| **Parameter** | **Description** | **Default** |
| --- | --- | --- |
| `configs.serviceType` | Type of Service to be used as backend for Routes/Ingresses | ClusterIP |
| `nodeSelectorLabels.key` | Key used as a label based selection for the nodes in NodePort mode. | empty |
| `nodeSelectorLabels.value` | Value used as a label based selection for the nodes in NodePort mode. | empty |

Kubernetes populates NodePort by default for service of type LoadBalancer too. If config.serviceType is set to NodePort, AKO would use NodePort as backend for service of type Loadbalancer instead of using Endpoints, which is the default behaviour with config.serviceType set as ClusterIP.

### AKO in Openshift Cluster

AKO can be used in openshift cluster to configure Routes and Services of type Loadbalancer.

#### Pre-requisites for running AKO in Openhsift Cluster

Follow the steps 1 to 2, given in section [Pre-requisites](https://github.com/avinetworks/avi-helm-charts/tree/master/docs/AKO#pre-requisites). Additionaly following points have to be noted for openshift environment.
1. Make Sure Openshift version is >= 4.4
2. Openshift routes and services of type load balancer are suppported in AKO
3. Ingresses, if created in the openshift cluster won't be handled by AKO.
4. serviceType should be **NodePort** in values.yaml. Routes should use services of type NodePort as backends.

#### Installing Beta Release:


Step 1: Create the avi-system namespace

    kubectl create ns avi-system

Step 2: Add ako incubator repository
    
    helm repo add ako-incubator https://avinetworks.github.io/avi-helm-charts/charts/incubator/ako


Step 3: Search for available charts

    helm search repo --devel

    NAME                 	CHART VERSION	        APP VERSION	        DESCRIPTION
    ako-incubator/ako	    1.2.1-5140-beta	        1.2.1-5140-beta	    A Helm chart for Kubernetes


Step 4: Install AKO

    helm install  ako-incubator/ako  --generate-name --version 1.2.1-5140-beta -f values.yaml --set configs.serviceType=NodePort --set configs.networkName=<network-name> --set configs.clusterName=<cluster-name>  --set configs.controllerIP=<avi-controller-ip> --set avicredentials.username=<avi-ctrl-username> --set avicredentials.password=<avi-ctrl-password> --namespace=avi-system

Step 5: Check the installation

    helm ls -n avi-system

    NAME          	NAMESPACE 	REVISION	UPDATED                                	STATUS  	CHART              	APP VERSION
    ako-1595250583	avi-system	1       	2020-07-20 13:09:45.958991153 +0000 UTC	deployed	ako-1.2.1-5140-beta	1.2.1-5140-beta

#### Features of Openshift Route supported in AKO
AKO supports the following features of openshift route. These features are currently in **beta** phase.
1. Insecure Routes.
2. Insecure Routes with alternate backends.
3. Secure routes with edge termination policy.

#### Features of Openshift Route to be supported in upcoming releases of AKO
Following feature of openshift routes are not currently supported in AKO. They will be supported in upcoming releases.
1. Secure Routes with InsecureEdgeterminationPolicy - Redirect or Allow.
2. Secure Routes of type passthrough
3. Secure Routes with reencrypt functionality

