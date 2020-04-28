# Helm charts by Avi Networks

This repository is used to host helm charts from Avi Networks (A VMware Company)

## Helm Charts for AKO - Avi Kubernetes Operator


### Run AKO

AKO runs as a POD inside the kubernetes cluster.

 ![Alt text](docs/AKO.png?raw=true "Title")
 
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
    
    ![Alt text](docs/data_path_flow.png?raw=true "Title")
    
      *NOTE: If you are using AKO for test puposes you can use the `global` vrf but you cannot manage multiple kubernetes clusters in the same cloud with this setting.*

 - ***Step 3.1***: If your POD CIDRs are routable then you can skip step 2. Ensure that you skip static route syncing in this case using the `disableStaticRouteSync` flag in the `values.yaml` of your helm chart.
 - ***Step 4:*** Kubernetes 1.14+.
 - ***Step 5:*** `helm` cli pointing to your kubernetes cluster.

#### Install using *helm*

  Step 1: Create the `avi-system` namespace:

    kubectl create ns avi-system


  Step 2: Add this repository to your helm CLI
    
    helm repo add ako https://avinetworks.github.io/avi-helm-charts/ako-beta

Use the `values.yaml` from this repository to edit values related to Avi configuration. Values and their corresponding index can be found [here](#parameters) 

  Step 3: Search the available charts for AKO

    helm search repo --devel
    
    NAME      	CHART VERSION  	APP VERSION    	DESCRIPTION
    ako/ako   	0.9.1-5047-beta	0.9.1-5047-beta	A Helm chart for Kubernetes

 Step 4: Install AKO

    helm install  ako/ako  --generate-name --devel -f values.yaml --set configs.controllerIP=<avi-controller-ip> --version 0.9.1-5047-beta -n avi-system
    
    NAME: ako-1588049417
    LAST DEPLOYED: Tue Apr 28 10:20:21 2020
    NAMESPACE: avi-system
    STATUS: deployed
    REVISION: 1
    NOTES:
    1. Get the application URL by running these commands:
      export POD_NAME=$(kubectl get pods --namespace avi-system -l "app.kubernetes.io/name=ako,app.kubernetes.io/instance=ako-1588049417" -o jsonpath="{.items[0].metadata.name}")
      echo "Visit http://127.0.0.1:8080 to use your application"
      kubectl --namespace avi-system port-forward $POD_NAME 8080:80
      
  Step 5: Check the installation
  
      helm list -n avi-system
    NAME          	NAMESPACE 	REVISION	UPDATED                            	STATUS  	CHART              	APP VERSION
    ako-1588049417	avi-system	1       	2020-04-28 10:20:21.61093 +0530 IST	deployed	ako-0.9.1-5047-beta	0.9.1-5047-beta
    
#### Uninstall using *helm*

Simply run:


*Step1:*

    helm delete <ako-release-name> -n avi-system
    
 Note: the ako-release-name is obtained by doing helm list as shown in the previous step,
 
*Step 2:* 

    kubectl delete ns avi-system

## Parameters


The following table lists the configurable parameters of the AKO chart and their default values. Please refer to this link for more details on [each parameter](docs/values.md).

| **Parameter**                                   | **Description**                                         | **Default**                                                           |
|---------------------------------------------|-----------------------------------------------------|-------------------------------------------------------------------|
| `configs.controllerVersion`                      | Avi Controller version                       | 18.2.8                                                            |
| `configs.controllerIP`                         | Specify Avi controller IP    | `nil`      |
| `configs.shardVSSize`                   | Shard VS size enum values: LARGE, MEDIUM, SMALL     | LARGE      |
| `configs.fullSyncFrequency`                       | Full sync frequency       | 1800                                                            
| `configs.ingressApi`                      | Support for default ingress API                      | corev1                                                           |
| `configs.defaultIngController`                         | AKO is the default ingress controller   | true      |               |
| `configs.subnetIP`                   | Subnet IP of the data network     | empty      | |
| `configs.subnetPrefix`                       | Subnet Prefix of the data network       | empty |                   
| `configs.networkName`                         | Network Name of the data network    | empty      |
| `configs.l7ShardingScheme`                   | Sharding scheme to use. Choices: hostname, namespace     | hostname      |
| `configs.cniPlugin`                       | Specify the CNI used, only in case of calico      | empty                                                            |
| `configs.cloudName`                            | Name of the VCenter cloud managed in Avi                              | Default-Cloud                                                       |
| `configs.disableStaticRouteSync`                          | Disable static route sync                                  | false                                                 |
| `configs.vrfRefName`                          | VRF context name to be used for the kubernetes cluster                                  | global                                                 |
| `avicredentials.username`                                 | Avi controller username                                  | empty                                                      |
| `avicredentials.password`                          | Avi controller password                          | empty                                                    |
| `image.repository`                         | Specify docker-registry that has the ako image    | avinetworks/ako     |


