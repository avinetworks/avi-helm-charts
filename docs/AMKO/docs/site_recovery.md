## Recover a failed instance of AMKO

There can be scenarios where a cluster which is running the AMKO pod fails, or the user loses control over the cluster due to some network disruption
It is always recommended to take backups of the following configuration objects for AMKO at regular intervals:
1. `GSLBConfig`
2. `GDP`
3. `GSLBHostRule`

To get a list of all such objects, use the following commands:
```
kubectl get gslbconfig -n avi-system
kubectl get gdp -n avi-system
kubectl get gslbhostrule --all-namespaces
```

**Note** that AMKO needs only the above configuration for recovery scenarios.

### Deployment Scenario
Now, let's consider a scenario where there are 3 sites, and each site has some clusters:
1. Site 1:
    - clusterA (AMKO is currently hosted here)
    - clusterB
2. Site 2:
    - clusterC
3. Site 3:
    - clusterD

There can be a couple of failures where AMKO is running:
1. `Site failure`: Entire site `Site 1` fails.
2. `Cluster failure`: Cluster `clusterA` fails inside `Site 1`.

#### Step 1: Disable previous AMKO logins
To avoid any conflicting scenarios (multiple AMKOs running together with same the configuration), change the password in the Avi Controller which was getting used by the previous AMKO instance in `cluster A`.

#### Step 2: Switch to a different cluster
Select a new cluster from the [Deployment Scenario](#Deployment-Scenario). In the event of `Site failure`, the user can choose a cluster from `Site 2` or `Site 3`. In the event of `Cluster failure`, the user can choose `clusterB` as the new cluster to deploy AMKO. For this example, let's consider that `clusterD` in `site 3` was chosen.

#### Step 3: Install AMKO on the new cluster
Edit the `values.yaml` for AMKO (if installing via `helm`) and update the new Avi Controller password for this instance. Also, keep the `gslbLeaderController` field **empty**. This field will get filled up later. Use the steps [here](../README.md#Install-using-helm) to then install AMKO.

#### Step 4: Restore the AMKO configs to the new cluster
Once AMKO is installed, the user can then restore the backed up configuration `GSLBConfig`, `GDP` and `GSLBHostRule` objects on this cluster `clusterD`:
```
kubectl apply -f gc.yaml
kubectl apply -f gdp.yaml
kubectl apply -f gslbhostrule.yaml  // done for the required number of GSLBHostRules
```

#### Step 5: Update configuration (if required)
Since we are using the existing objects as is, AMKO may not restart if the member clusters are in failed state. In that case, remove the member clusters which are in failed state, from:
1. `GSLBConfig` object `gc-1`
2. `GDP` object `global-gdp`

So, if `site 1` failed, remove `clusterA` and `clusterB` from the above objects. If just `clusterA` failed, remove only `clusterA` from the above objects.

Once the above objects are updated, AMKO should then be restarted:
```
kubectl delete pod -n avi-system amko-0
```

**Note** that the user can perform Step 3 before-hand on a certain set of chosen clusters as standby clusters. In case of failures, the config objects for AMKO can then be restored on one of these standy clusters, updated and AMKO can be restarted.