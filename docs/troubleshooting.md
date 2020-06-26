## Troubleshooting guide for Avi Kubernetes Operator

### AKO POD is not running

#### Possible Reasons/Solutions:

##### Check the reason why the POD didn't come up by doing the following:

    kubectl get pods -n avi-system
    NAME                 READY   STATUS             RESTARTS   AGE
    ako-f776577b-5zpxh   0/1     ImagePullBackOff   0          15s

##### Solution:

    Ensure that you have your docker registry configured properly or the image is configured locally.

#### AKO is not responding to my ingress object creations.

#### Possible Reasons/Solutions:

##### Look into the AKO container logs and see if you find a reason on why the sync is disabled like this:
    
    2020-06-26T10:27:26.032+0530	INFO	lib/lib.go:56	Setting AKOUser: ako-my-cluster for Avi Objects
    2020-06-26T10:27:26.337+0530	ERROR	cache/controller_obj_cache.go:1814	Required param networkName not specified, syncing will be disabled.
    2020-06-26T10:27:26.337+0530	WARN	cache/controller_obj_cache.go:1770	Invalid input detected, syncing will be disabled.

### My Ingress object didn't sync in Avi

#### Possible Reasons/Solutions:

    1. The ingress class is set as something other than "avi". defaultIngController is set to true. 
    2. For TLS ingress, the `Secret` object does not exist. Please ensure that the Secret object is pre-created.
    3. Check the connectivity between your AKO POD and the Avi Controller.

### My virtualservice returns a CONNECTION REFUSED after sometime.
 
#### Possible Reasons/Solutions:
 
    Check if your virtualservice IP is in use somewhere else in your network.

### My out-of-band virtualservice setting just got overwritten.

#### Possible Reasons/Solutions:

    You don't recommend changing properties of a shared virtualservice out-of-band.  If AKO has an ingress update 
    that related to this shared VS, then AKO would overwrite the configuration.
    
### Static routes are populated, but my pools are down.

#### Possible Reasons/Solutions:

    Check if you have a dual nic kubernetes worker node setup. In case of a dual nic setup, AKO would populate the static
    routes using the default gateway network. However, the default gateway network might not be the port group network that
    you want to use as the data network. Hence service engines may not be able to reach the POD CIDRs using the default gateway
    network. 
    
    If it's impossible to make your data networks routable via the default gateway, disableStaticRoute sync in AKO and edit your
    static routes with the correct network.

## Log Collection

For every log collection, also collect the following information:

    - What kubernetes distribution are you using? For example: RKE, PKS etc.
    - What is the CNI you are using with versions? For example: Calico v3.15
    - What is the Avi Controller version you are using? For example: 18.2.8

#### How do I gather the AKO logs?


#### How do I gather the controller tech support?



    
