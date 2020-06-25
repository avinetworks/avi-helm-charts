## Troubleshooting guide for Avi Kubernetes Operator

### My Ingress object didn't sync in Avi

#### Possible Reasons/Solutions:

    1. The ingress class is set as something other than "avi". defaultIngController is set to true. 
    2. For TLS ingress, the `Secret` object does not exist. Please ensure that the Secret object is pre-created.
 
### My virtualservice returns a CONNECTION REFUSED after sometime.
 
#### Possible Reasons/Solutions:
 
    Check if your virtualservice IP is in use somewhere else in your network.

### My out-of-band virtualservice setting just got overwritten.

#### Possible Reasons/Solutions:

    You don't recommend changing properties of a shared virtualservice out-of-band.  If AKO has an ingress update 
    that related to this shared VS, then AKO would overwrite the configuration.
    
### What logs should i collect when I want to report a problem with AKO?

    
