## Troubleshooting guide for Avi Multi Kubernetes Operator

#### AMKO POD is not running

#### Possible Reasons/Solutions:

##### Check the reason why the POD didn't come up by doing the following:

    kubectl get pods -n avi-system
    NAME                 READY   STATUS             RESTARTS   AGE
    amko-f776577b-5zpxh   0/1     ImagePullBackOff   0          15s

##### Solution:

    Ensure that you have your docker registry configured properly or the image is configured locally.

#### AMKO Pod is up, but no GSLB object created

##### Possible Reasons/Solutions:

Verify the namespaceSelector or appSelector filters on the GlobalDeploymentPolicy is able to select a
valid ingress object(s).


## Log Collection

For every log collection, also collect the following information:

    1. What kubernetes distribution are you using? For example: RKE, PKS etc.
    2. What is the CNI you are using with versions? For example: Calico v3.15
    3. What is the Avi Controller version you are using? For example: 18.2.8

### How do I gather the AMKO logs?

Get the script from [here](https://github.com/avinetworks/devops/tree/master/openshift/ako/log_collector.py)

The script is used to collect all relevant information for the AMKO pod.

**About the script:**

1. Collects log file of AMKO pod
2. Collects configmap  in a yaml file
3. Zips the folder and returns

_For logs collection, 3 cases are considered:_

Case 1 : A running AMKO pod logging into a Persistent Volume Claim, in this case the logs are collected from the PVC that the pod uses.

Case 2 : A running AMKO pod logging into console, in this case the logs are collected from the pod directly.

Case 3 : A dead AMKO pod that uses a Persistent Volume Claim, in this case a backup pod is created with the same PVC attached to the AMKO pod and the logs are collected from it.

**Configuring PVC for the AMKO pod:**

We recommend using a Persistent Volume Claim for the amko pod. Refer this [link](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/) to create a persistent volume(PV) and a Persistent Volume Claim(PVC). 

Below is an example of hostpath persistent volume. We recommend you use the PV based on the storage class of your kubernetes environment. 

    #persistent-volume.yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: amko-pv
      namespace : avi-system
      labels:
        type: local
    spec:
      storageClassName: manual
      capacity:
        storage: 10Gi
      accessModes:
        - ReadWriteOnce
      hostPath:
        path: <any-host-path-dir>  # make sure that the directory exists
        
A persistent volume claim can be created using the following file

    #persistent-volume-claim.yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: amko-pvc
      namespace : avi-system
    spec:
      storageClassName: manual
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 3Gi
          
Add PVC name into the amko/helm/amko/values.yaml before the creation of the amko pod like 

    persistentVolumeClaim: amko-pvc
    mountPath: /log
    logFile: avi.log

**How to use the script for AMKO**

Usage:

1. Case 1: With PVC, (Mandatory) --amkoNamespace (-amko) : The namespace in which the AMKO pod is present.

    `python3 log_collections.py -amko avi-system`

2. Case 2: Without PVC (Optional) --since (-s) : time duration from present time for logs.

    `python3 log_collections.py -amko avi-system -s 24h`

**Sample Run:**

At each stage of execution, the commands being executed are logged on the screen.
The results are stored in a zip file with the format below:

    amko-<helmchart name>-<current time>

Sample Output with PVC :

    2020-06-25 13:20:37,141 - ******************** AMKO ********************
    2020-06-25 13:20:37,141 - For AMKO : helm list -n avi-system
    2020-06-25 13:20:38,974 - kubectl get pod -n avi-system -l app.kubernetes.io/instance=my-amko-release
    2020-06-25 13:20:41,850 - kubectl describe pod amko-56887bd5b7-c2t6n -n avi-system
    2020-06-25 13:20:44,019 - helm get all my-amko-release -n avi-system
    2020-06-25 13:20:46,360 - PVC name is my-pvc
    2020-06-25 13:20:46,361 - PVC mount point found - /log
    2020-06-25 13:20:46,361 - Log file name is avi.log
    2020-06-25 13:20:46,362 - Creating directory amko-my-amko-release-2020-06-25-132046
    2020-06-25 13:20:46,373 - kubectl cp avi-system/amko-56887bd5b7-c2t6n:log/avi.log amko-my-amko-release-2020-06-25-132046/amko.log
    2020-06-25 13:21:02,098 - kubectl get cm -n avi-system -o yaml > amko-my-amko-release-2020-06-25-132046/config-map.yaml
    2020-06-25 13:21:03,495 - Zipping directory amko-my-amko-release-2020-06-25-132046
    2020-06-25 13:21:03,525 - Clean up: rm -r amko-my-amko-release-2020-06-25-132046

    Success, Logs zipped into amko-my-amko-release-2020-06-25-132046.zip

### How do I gather the controller tech support?


It's recommended we collect the controller tech support logs as well. Please follow this [link](https://avinetworks.com/docs/18.2/collecting-tech-support-logs/)  for the controller tech support.
    
