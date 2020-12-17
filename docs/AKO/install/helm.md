## Install using *helm*

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
helm install  ako/ako  --generate-name --version 1.2.1 -f values.yaml  --set ControllerSettings.controllerHost=<controller IP or Hostname> --set avicredentials.username=<avi-ctrl-username> --set avicredentials.password=<avi-ctrl-password> --namespace=avi-system
```

Step 5: Check the installation

```
helm list -n avi-system

NAME          	NAMESPACE 	
ako-1593523840	avi-system
```

## Uninstall using *helm*

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

## Upgrade AKO using *helm*

Follow these steps if you are upgrading from an older AKO release.

*Step1*

Helm does not upgrade the CRDs during a release upgrade. Before you upgrade a release, run the following command to upgrade the CRDs:

```
kubectl apply -f https://github.com/avinetworks/avi-helm-charts/tree/master/charts/stable/ako/crds/
```

*Step2*

```
helm list -n avi-system

NAME          	NAMESPACE 	REVISION	UPDATED                             	STATUS  	CHART    	APP VERSION
ako-1593523840	avi-system	1       	2020-09-16 13:44:31.609195757 +0000 UTC	deployed	ako-1.1.1	1.1.1-9032
```

*Step3*

```
helm upgrade ako-1593523840 ako/ako -f values.yaml --version 1.2.1 --set ControllerSettings.controllerHost=<IP or Hostname> --set avicredentials.password=<username> --set avicredentials.username=<username> --namespace=avi-system
```

Note:

* Seamless upgrade to version 1.2.1 is not supported. Before upgrading to 1.2.1, uninstall ako, delete all objects from AVI and install ako 1.2.1 using helm.
* Do not delete the configmap avi-k8s-config manually, unless you are doing a complete helm uninstall. AKO pod has to be rebooted if you delete and recreate the avi-k8s-config configmap.
* In order to update CRDs, make sure to run the `kubect apply` as mentioned above, before running helm upgrade. 
