# AMKO: Avi Multicluster Kubernetes Operator

### Run AMKO

AMKO is a kubernetes operator which orchestrates application deployments across multiple kubernetes/openshift clusters.
 ![Alt text](images/amko_ss.png?raw=true "amko architecture")

AMKO is aware of the following object types:
* Kubernetes Ingresses
* Openshift Routes
* Service type Load Balancer

Based on a couple of CRDs `GSLBConfig` and `GlobalDeploymentPolicy`, AMKO manages the GSLB service configuration on a AVI leader site.
 
#### Pre-requisites
To kick-start AMKO, we need:
1. Atleast one kubernetes/openshift cluster.
2. Atleast one AVI controller which manages the kubernetes/openshift cluster(s) designated as the GSLB leader site. Additional controllers managing openshift/kubernetes clusters can be added as follower sites.
3. AMKO assumes that it has connectivity to all the member clusters' kubernetes API servers. Without this, AMKO won't be able to watch over the ingress/route/services in the member kubernetes clusters.
4. Choose one of the kubernetes/openshift clusters where AMKO should be deployed. All the configs for `amko` will be added to this cluster. Let's call this cluster `cluster-amko` for further references.
5. Create a namespace `avi-system` in `cluster-amko`:
   ```
   kubectl create ns avi-system
   ```

6. Create a kubeconfig file with the permissions to read the service and the ingress/route objects for all the member clusters. Follow [this tutorial](kubeconfig.md) to create a kubeconfig file with multi-cluster access. Name this file `gslb-members` and generate a secret with the kubeconfig file in `cluster-amko` by following:
   ```
   kubectl create secret generic gslb-config-secret --from-file gslb-members -n avi-system
   ```
   *Note* that the permissions provided in the kubeconfig file for all the clusters must have atleast the permissions to `[get, list, watch]` on services, ingresses and routes.

#### Install using helm
*Note* that only helm v3 is supported.

1. Create the `avi-system` namespace:
   ```
   kubectl create ns avi-system
   ```
2. Add this repository to your helm client:
   ```
   helm repo add amko https://avinetworks.github.io/avi-helm-charts/charts/incubator/amko
   ```
   Use the `values.yaml` from this repository to provide values related to Avi configuration. Check [here](#parameters) for the required values.
4. Search the available charts for AMKO:
   ```
   helm search repo

    NAME                 	CHART VERSION	APP VERSION	DESCRIPTION
    amko/amko               0.9.1        	0.9.1      	A helm chart for Avi Multicluster Kubernetes Operator
   ```
5. Install AMKO:
   ```
   helm install  ako/ako  --generate-name --version 0.9.1 -f values.yaml  --set configs.controllerIP=<controller_ip> --namespace=avi-system

   NAME: amko-1593523840
   LAST DEPLOYED: Tue Jun 30 19:00:44 2020
   NAMESPACE: avi-system
   STATUS: deployed
   REVISION: 1
   NOTES:
   1. Get the application URL by running these commands:
     export POD_NAME=$(kubectl get pods --namespace avi-system -l "app.kubernetes.io/name=amko,app.kubernetes.io/instance=amko-1593523840" -o jsonpath="{.items[0].metadata.name}")
     echo "Visit http://127.0.0.1:8080 to use your application"
     kubectl --namespace avi-system port-forward $POD_NAME 8080:80
   ```
6. Check the installation:
   ```
   helm list -n avi-system

   NAME          	NAMESPACE 	REVISION	UPDATED                             	STATUS  	CHART    	APP VERSION
   amko-1593523840	avi-system	1       	2020-06-30 19:00:44.134075 +0530 IST	deployed	amko-0.9.1	    0.9.1
   ```

#### Uninstall using helm
```
helm uninstall -n avi-system <amko-release-name>
```
If a user needs to remove the already created GSLB services, one has to remove the GDP object first. This will remove all the GSLB services selected via the GDP object.
```
kubectl delete gdp -n avi-system global-gdp
```
Also, delete the `avi-system` namespace:
```
kubectl delete ns avi-system
```

#### parameters
| **Parameter**                                    | **Description**                                                                                                          | **Default**                           |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ | ------------------------------------- |
| `configs.controllerVersion`                      | GSLB leader controller version                                                                                           | 18.2.9                                |
| `configs.gslbLeaderHost`                         | GSLB leader site URL                                                                                                     | Nil                                   |
| `configs.gslbLeaderSecret`                       | GSLB leader credentials secret name in `avi-system` namespace                                                            | `gslb-avi-secret`                     |
| `configs.gslbLeaderCredentials.username`         | GSLB leader controller username                                                                                          | `admin`                               |
| `configs.gslbLeaderCredentials.password`         | GSLB leader controller password                                                                                          | `avi123`                              |
| `configs.memberClusters.clusterContext`          | K8s member cluster context for GSLB                                                                                      | `cluster1-admin` and `cluster2-admin` |
| `configs.refreshInterval`                        | The time interval which triggers a AVI cache refresh                                                                     | 120 seconds                           |
| `configs.clusterMembers.secret`                  | The name of the secret which is created with kubeconfig as the data                                                      | `gslb-config-secret`                  |
| `configs.clusterMembers.key`                     | The name of the field (key) inside the secret `configs.clusterMembers.secret` data                                       | `gslb-members`                        |
| `gdpConfig.appSelector.label{.key,.value}`       | Selection criteria for applications, label key and value are provided                                                    | Nil                                   |
| `gdpConfig.namespaceSelector.label{.key,.value}` | Selection criteria for namespaces, label key and value are provided                                                      | Nil                                   |
| `gdpConfig.matchClusters`                        | List of clusters (names must match the names in configs.memberClusters) from where the objects will be selected          | Nil                                   |
| `gdpConfig.trafficSplit`                         | List of weights for clusters (names must match the names in configs.memberClusters), each weight must range from 1 to 20 | Nil                                   |
