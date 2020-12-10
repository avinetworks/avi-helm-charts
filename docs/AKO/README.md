# AKO: Avi Kubernetes Operator

### Run AKO

AKO runs as a POD inside the Kubernetes cluster.

#### Pre-requisites

To Run AKO you need the following pre-requisites:

* <i>**Step 1**</i>: Configure an Avi Controller with a vCenter [cloud](https://avinetworks.com/docs/18.2/installing-avi-vantage-for-vmware-vcenter/). The Avi Controller should be versioned 18.2.10 / 20.1.2 or later.
* <i>**Step 2**</i>:
    * Make sure a PG network is part of the NS IPAM configured in the vCenter
* <i>**Step 3**</i>: If your POD CIDRs are not routable:
Data path flow is as described below:
![Alt text](data_path_flow.png?raw=true)
The markers in the drawing are described below:
    1. The client requests a specified hostname/path.
    2. The DNS VS returns an IP address corresponding to the hostname.
    3. The request is forwarded to the resolved IP address that corresponds to a Virtual IP hosted on an Avi Service Engine.
    The destination IP in the packet is set as the POD IP address on which the application runs.
    4. Service Engines use the static route information to reach the POD IP via the next-hop address of the host on which the pod is running.
    5. The pod responds and the request is sent back to the client.
    * Create a Service Engine Group dedicated to a Kubernetes cluster.
* <i>**Step 3.1**</i>: If your POD CIDRs are routable then you can skip step 2. Ensure that you skip static route syncing in this case using the `disableStaticRouteSync` flag in the `values.yaml` of your helm chart.
* <i>**Step 4:**</i> Kubernetes 1.16+.
* <i>**Step 5:**</i> `helm` cli pointing to your kubernetes cluster.
NOTE: We only support `helm 3`

#### Install using *helm*

Step 1: Create the `avi-system` namespace:

```
kubectl create ns avi-system
```

Step 2: Add this repository to your helm CLI

```
helm repo add ako https://avinetworks.github.io/avi-helm-charts/charts/stable/ako
```

Use the `values.yaml` from this repository to edit values related to Avi configuration. Values and their corresponding index can be found [here](#parameters)

Step 3: Search the available charts for AKO

```
helm search repo

NAME                 	CHART VERSION	APP VERSION	DESCRIPTION
ako/ako              	1.2.1        	1.2.1      	A helm chart for Avi Kubernetes Operator
```

Step 4: Install AKO

```
helm install  ako/ako  --generate-name --version 1.2.1 -f values.yaml  --set ControllerSettings.controllerIP=<controller_ip> --set avicredentials.username=<avi-ctrl-username> --set avicredentials.password=<avi-ctrl-password> --namespace=avi-system
```

Step 5: Check the installation

```
helm list -n avi-system

NAME          	NAMESPACE 	
ako-1593523840	avi-system
```

#### Uninstall using *helm*

Simply run:

*Step1:*

```
helm delete <ako-release-name> -n avi-system
```

Note: the ako-release-name is obtained by doing helm list as shown in the previous step,

*Step 2:*

```
kubectl delete ns avi-system
```

#### Upgrade AKO using *helm*

If you are upgrading from an older AKO release then simply run the helm upgrade command. For example:

*Step1*

```
helm list -n avi-system

NAME          	NAMESPACE 	REVISION	UPDATED                             	STATUS  	CHART    	APP VERSION
ako-1593523840	avi-system	1       	2020-09-16 13:44:31.609195757 +0000 UTC	deployed	ako-1.1.1	1.1.1-9032
```

*Step2*

```
helm upgrade ako-1593523840 ako/ako -f values.yaml --version 1.2.1 --set ControllerSettings.controllerIP=<IP> --set avicredentials.password=<username> --set avicredentials.username=<username> --namespace=avi-system
```

Note:

* Seamless upgrade to version 1.2.1 is not supported. Before upgrading to 1.2.1, uninstall ako, delete all objects from AVI and install ako 1.2.1 using helm.
* Do not delete the configmap avi-k8s-config manually, unless you are doing a complete helm uninstall. AKO pod has to be rebooted if you delete and recreate the avi-k8s-config configmap.

## Parameters

The following table lists the configurable parameters of the AKO chart and their default values. Please refer to this link for more details on [each parameter](values.md).

| **Parameter** | **Description** | **Default** |
| --------- | ----------- | ------- |
| `ControllerSettings.controllerVersion` | Avi Controller version | 18.2.10 |
| `ControllerSettings.controllerIP` | Specify Avi controller IP | `nil` |
| `ControllerSettings.cloudName` | Name of the cloud managed in Avi | Default-Cloud |
| `ControllerSettings.tenantsPerCluster` | Set to true if user want to map each kubernetes cluster uniquely to a tenant in Avi | false |
| `ControllerSettings.tenantName` | Name of the tenant where all the AKO objects will be created in AVI. | admin |
| `L7Settings.shardVSSize` | Shard VS size enum values: LARGE, MEDIUM, SMALL | LARGE |
| `AKOSettings.fullSyncFrequency` | Full sync frequency | 1800 |
| `L7Settings.defaultIngController` | AKO is the default ingress controller | true |
| `ControllerSettings.serviceEngineGroupName` | Name of the Service Engine Group | Default-Group |
| `NetworkSettings.nodeNetworkList` | List of Networks and corresponding CIDR mappings for the K8s nodes. | `Empty List` |
| `AKOSettings.clusterName` | Unique identifier for the running AKO instance. AKO identifies objects it created on Avi Controller using this param. | **required** |
| `NetworkSettings.subnetIP` | Subnet IP of the data network | **required** |
| `NetworkSettings.subnetPrefix` | Subnet Prefix of the data network | **required** |
| `NetworkSettings.networkName` | Network Name of the data network | **required** |
| `NetworkSettings.enableRHI` | Publish route information to BGP peers | false |
| `L4Settings.defaultDomain` | Specify a default sub-domain for L4 LB services | First domainname found in cloud's dnsprofile |
| `L7Settings.l7ShardingScheme` | Sharding scheme enum values: hostname, namespace | hostname |
| `AKOSettings.cniPlugin` | CNI Plugin being used in kubernetes cluster. Specify one of: calico, canal, flannel | **required** for calico setups |
| `AKOSettings.logLevel` | logLevel enum values: INFO, DEBUG, WARN, ERROR. logLevel can be changed dynamically from the configmap | INFO |
| `AKOSettings.deleteConfig` | set to true if user wants to delete AKO created objects from Avi. deleteConfig can be changed dynamically from the configmap | false |
| `AKOSettings.disableStaticRouteSync` | Disables static route syncing if set to true | false |
| `avicredentials.username` | Avi controller username | empty |
| `avicredentials.password` | Avi controller password | empty |
| `image.repository` | Specify docker-registry that has the AKO image | avinetworks/ako |

> `networkName`, `subnetIP` and `subnetPrefix` are required fields which are used for allocating VirtualService IP by IPAM Provider module

> Each AKO instance mapped to a given Avi cloud should have a unique clusterName parameter. This would maintain the uniqueness of object naming across Kubernetes clusters.

## AKO Operator
If you want to install and manage the AKO Controller from one place, you should use the AKO Operator. The AKO Operator deploys and manages an instance of the AKO Controller.
### Install and Manage AKO using the AKO operator
Please follow [this](AKO_Operator.md) to:
1. Install the AKO operator
2. Deploy and manage the AKO controller using the AKO operator

### Caveats with existing AKO installations
AKO operator is not supported to work with the standalone deployments of AKO, via helm or otherwise (non-operator based installations). The reason for this is, the AKO Operator  completely manages the AKO controller and it's related artifacts. If users want to switch from a non-operator based deployment of AKO to AKO operator, follow these steps:
1. Uninstall the existing AKO installation.
2. Install the AKO operator (follow [this](AKO_Operator.md#Install-using-helm) for helm based installation).
3. Deploy and manage the AKO controller via the operator (follow [this](AKO_Operator.md#AKOConfig-Custom-Resource)).

### AKO objects

Please refer to this [page](objects.md) for details on how AKO interprets the Kubernetes objects and translates them to Avi objects.

### FAQ

For some frequently asked question refer [here](faq.md)

### AKO in Openshift Cluster

AKO can be used in openshift cluster to configure Routes and Services of type Loadbalancer.

#### Pre-requisites for running AKO in Openshift Cluster

Follow the steps 1 to 2, given in section [Pre-requisites](https://github.com/avinetworks/avi-helm-charts/tree/master/docs/AKO#pre-requisites). Additionally, the following points have to be noted for openshift environment.

1. Make Sure Openshift version is >= 4.4
2. Openshift routes and services of type load balancer are supported in AKO
3. Ingresses, if created in the openshift cluster won't be handled by AKO.
4. cniPlugin should be set to **openshift**
5. Set `state_based_dns_registration` to false in AVI cloud configuration. Follow the instructions mentioned in https://avinetworks.com/docs/20.1/dns-record-additions-independent-of-vs-state/.

#### Features of Openshift Route supported in AKO

AKO supports the following Layer 7 functions of the OpenShift Route object:

1. Insecure Routes.
2. Insecure Routes with alternate backends.
3. Secure routes with edge termination policy.
4. Secure Routes with InsecureEdgeTerminationPolicy - Redirect or Allow.
5. Secure Routes of type passthrough
6. Secure Routes with re-encrypt functionality

### Node Port

Service of type `NodePort` can be used to send traffic to the pods using nodeports.

This feature supports Ingress/Route attached to Service of type `NodePort`. Service of type LoadBalancer is also supported, since kubernetes populates `NodePort` by default. AKO will function either in `NodePort` mode or in `ClusterIP` mode.

A new parameter serviceType has been introduced as config option in AKO's values.yaml. To use this feature, set the value of the parameter to **NodePort**.

| **Parameter** | **Description** | **Default** |
| --------- | ----------- | ------- |
| `configs.serviceType` | Type of Service to be used as backend for Routes/Ingresses | ClusterIP |
| `nodeSelectorLabels.key` | Key used as a label based selection for the nodes in NodePort mode. | empty |
| `nodeSelectorLabels.value` | Value used as a label based selection for the nodes in NodePort mode. | empty |

Kubernetes populates NodePort by default for service of type LoadBalancer. If config.serviceType is set to NodePort, AKO would use NodePort as backend for service of type Loadbalancer instead of using Endpoints, which is the default behaviour with config.serviceType set as ClusterIP.

### Tenancy support in AKO

This feature allows AKO to map each kubernetes / OpenShift cluster uniquely to a tenant in Avi. `ControllerSettings.tenantsPerCluster` needs to be set to `true` to enable this feature.

#### Steps to enable Tenancy in AKO

* Create seperate tenant for each cluster in AVI. For the below steps, lets assume `billing` tenant is created by the Avi controller admin.
![Alt text](images/tenant_path.png?raw=true)
* Click `create`
![Alt text](images/new_tenant.png?raw=true)
* Create the [`ako-admin`](roles/ako-admin.json) and[`ako-tenant`](roles/ako-tenant.json) roles which gives appropriate previliges to the ako user in `admin` and `billing` tenant.
![Alt text](images/role_list.png?raw=true)
* Create a new user for AKO in AVI under `Administration->Accounts->Tenants`
![Alt text](images/user_path.png?raw=true)
* Click `create`
![Alt text](images/new_user.png?raw=true)
* Assign [`ako-admin`](roles/ako-admin.json) and [`ako-tenant`](roles/ako-tenant.json) roles to admin and billing tenant respectively.
![Alt text](images/new_user_role.png?raw=true)
* In **AKO**, Set the `ControllerSettings.tenantsPerCluster` to `true` and `ControllerSettings.tenantName` to the tenant created in the earlier steps.
* In **AKO**, Set the `avicredentials.username` and `avicredentials.password` to the user credentials created above.

With the above settings AKO will map the `billing` cluster to the `billing` tenant and all the objects will be created in that tenant.

Note:
- In `NodePort` mode of AKO (when `L7Settings.serviceType` is set to `NodePort`), VRFContext permissions are not required in `admin` tenant in AVI Controller. 

### AWS and Azure IaaS Cloud in NodePort mode of AKO

Supports AWS and Azure IaaS cloud with Avi DNS.