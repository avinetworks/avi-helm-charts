# AKO in Public Clouds

## AKO in GCP

This section covers the support for `ClusterIP` mode for GCP IaaS cloud in Avi Controller. AKO will be managing the pod routing in the GCP.

### Deployment Modes

Network Configuration modes supported by AKO in GCP Full Access IaaS cloud are:

* Inband
* Two arm mode with Dedicated management

In both of the deployments there is a one to one mapping between VPC and a kubernetes cluster. So In case of Inband each cluster should have a corresponding cloud created in AVI.

In Two Arm mode we can have multiple clusters syncing to the same cloud. Details to configure the same will follow in this section.

Below are some of the terminologies used to describe the network deployment in AVI.

* Frontend Data Network - This network connects the VIP to the Service Engines. All the VIP traffic reaches Service Engines through this network
* Backend Data Network - This network connects the Service Engines to the Application Servers. All the traffic between the Service Engines and Application Servers flows through this network.
* Management Network - This network connects the Service Engines with the Controller for all management operations.

This [document](https://avinetworks.com/docs/20.1/configuring-gcp-cloud-network/) has more details on the GCP Cloud network configuration in AVI.

#### Inband Management

![Alt text](images/public_cloud/inband.png?raw=true)

* Service Engines will be connected to only 1 VPC Subnet
* No network isolation between frontend and backend data and management traffic as both will go through same VPC Subnet

#### Two-Arm Mode with the same backend subnet

![Alt text](images/public_cloud/two_arm.png?raw=true)

* Service Engines will be connected to three VPC Subnets, one each for the frontend data traffic, management traffic and backend data traffic.
* Provides isolation between management, frontend data and backend data networks.
* First Interface of the SE will be connected to the frontend data network. GCP supports shared VPC only on first NIC, therefore the data VPC can be a shared VPC.
* The second Interface of the SE will be connected to the management network. This interface cannot be connected to a shared VPC because GCP allows shared VPC only on first NIC.
* The third Interface of the SE will be connected to the Kubernetes cluster where AKO is running.
* In this mode, the backend network of the SE is the same as the one where the Kubernetes / OpenShift nodes are connected.

For more details refer to this [document](https://avinetworks.com/docs/20.1/configuring-gcp-cloud-network/)

##### Routing in GCP Two-Arm Mode with the same backend subnet

When multiple clusters are syncing to the same cloud the POD CIDR’s can overlap. Currently, In AKO for each cluster, SE Group is created. In addition to Service Engine Group each cluster should be in unique VPC in GCP cloud. This config is supplied in the Service Engine Group during the cloud setup by the admin.

![Alt text](images/public_cloud/gcp_routing.png?raw=true)

* AKO configures the routes for `cluster1` and `cluster2` in the **GCP** routing tables for `vpc1` and `vpc2` respectively. Since there is unique mapping for cluster and SE Group the overlapping pod cidr's routes will be in their respective VPC's.

### AKO in GCP Day 0 Preparation

Following lists the Day 0 preparation work required to set up AKO in GCP

#### GCP Side Preparation

* Kubernetes / Openshift clusters running in their own dedicated VPC.
* Kubernetes and Openshift cluster node VM’s should have **IP forwarding** enabled in the [GCP VM's](https://cloud.google.com/vpc/docs/using-routes#canipforward).
* Create a dedicated backend VPC for each of the clusters.

#### AVI side preparation

* Create a GCP cloud in Avi. Skip this step if the IaaS cloud is already created.
* Create a Service Engine group for each cluster.
* Override the backend vpc subnet in each of the SEG.
  * If there are two clusters `cluster1` and `cluster2`
    * Run `configure serviceenginegroup cluster1-seg`
    * Override the `gcp_config` in the SE Group.
    * ![Alt text](images/public_cloud/gcp_segroup.png?raw=true)
    * Similarly override the `gcp_config` for the `cluster2` with its VPC and Subnet details.

## AKO in AZURE

This section covers the support for ClusterIP mode support for Azure IaaS cloud in Avi Controller. AKO will be managing the pod routing in the Azure cloud.

### Routing in Azure with two Kubernetes clusters syncing to the same Azure Cloud

When multiple clusters are syncing to the same cloud the POD CIDR’s can overlap. Currently, In AKO for each cluster, SE Group is created. For Azure cloud in addition to the creation of an SE group, SE Network/ subnet needs to be overridden in the SE group which will be a unique per cluster

![Alt text](images/public_cloud/azure_routing.png?raw=true)

* AKO configures the routes for `cluster1` and `cluster2` in the **Azure** routing tables for `subnet1` and `subnet2` respectively.

### AKO in Azure Day 0 Preparation

Following lists the Day 0 preparation work required to set up AKO in Azure

#### Azure Side Preparation

* Kubernetes / Openshift clusters running in Azure.
  * The clusters need to be in same **VNET** as the SE.
* Create a dedicated subnet for each of the clusters in **VNET** for SE to be provisioned in.
    * `subnet1` and `subnet2` are created for `cluster1` and `cluster2` respectively
* Create Route Table in Azure for each subnet created above and associate it to the SE subnet.
  * Create `RouteTable1` and `RouteTable2` and associate to `subnet1` and `subnet2` respectively.
* Configure NSG Rules in kubernetes cluster’s subnet to allow traffic from SE Subnet.
* Give permissions to tha AVI cloud credential to write on route tables.
  * Avi Controller role for the AKO [`avi-controller-ako`](roles/avicontroller-ako.json)
  * Use the above role to configure the cloud in the Avi. Details for the same are in this [link](https://avinetworks.com/docs/20.1/role-setup-for-installation-into-microsoft-azure/).

#### AVI side preparation

* Create an Azure cloud in Avi. Skip this step if the IaaS cloud is already created.
* Create a Service Engine group for each cluster.
* Override the backend vpc subnet in each of the SEG.
  * If there are two clusters `cluster1` and `cluster2`
  * Run `configure serviceenginegroup seg1`
  * Override the `se_subnet_id` in the SE Group by running below commands.
    * `se_subnet_id subnet1`
    * `save`
  * Similarly override the `se_subnet_id` for the `cluster2` .
