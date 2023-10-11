


## Policies

Folder /day2/policies contains all the policies which are deployed using Applications>Subscription
When using subscriptions to deploy policies, the placement for subscriptions should be only the hub cluster. Subscription manager deploys policies only to the hub cluster. Then these are passed to the policy controller on the hub cluster which then distributes them to all themanaged clusters. To achieve this, two placement definitions are involved:  

* placement for the subcription which should point to the hub cluster only.
* placement for the policies which can be seen in the "day2/policies/placements.yaml" file. 

The day2/policies/placements.yaml also contains PlacementBinding definition which links Placement to specific policies.  

```
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: binding-policies-all-clusters
  namespace: default
placementRef:
  name: placement-policies-all-clusters
  apiGroup: cluster.open-cluster-management.io
  kind: Placement
subjects:
  - name: policy-push-all-htpasswd
    apiGroup: policy.open-cluster-management.io
    kind: Policy
  - name: install-quay
    apiGroup: policy.open-cluster-management.io
    kind: Policy
  - name: policy-configure-clusterlevel-rbac
    apiGroup: policy.open-cluster-management.io
    kind: Policy
  - name: policy-push-identity-provider
    apiGroup: policy.open-cluster-management.io
    kind: Policy
  - name: policy-remove-kubadmin
    apiGroup: policy.open-cluster-management.io
    kind: Policy
```

In the above PlacementBinding we are telling the Policy controller to to link the placement defined in "placementRef" which is in our case"placement-policies-all-clusters" to the policies defined under subjects. These policies are defined in the files within /day2/policies folder.

## Issue with Policies deployed using Subscription on a cluster with internet proxy

We encountered the issue when trying to deploy Policies using subscriptions. After creating Subscription the policies were always in pending state. The problem was in that the http_proxy was not configured for the applicationManager which needs access to the git repo defined for policy deployments.  

This issue as fixed using the following procedure:

1. Edit the KlusterletAddonConfig file that is in the namespace of the cluster that needs the proxy. You can use the console to find the resource, or you can edit from the terminal with the following command:
```
oc -n local-cluster edit klusterletaddonconfig local-cluster
```

2. Add the .spec.proxyConfig section of the file so it resembles the following example. The spec.proxyConfig is an optional section:
```
spec
  proxyConfig:
    httpProxy: http://10.254.102.198:3128
    httpsProxy: http://10.254.102.198:3128
    noProxy: .cluster.local,.svc,10.128.0.0/14,10.254.164.0/24,10.254.164.248,10.254.171.0/24,10.254.171.11,10.254.171.12,127.0.0.1,172.30.0.0/16,api-int.hub-dev-cci.refmobilecloud.ux.nl.tmo,localhost,refmobilecloud.ux.nl.tmo  
```

3. Add proxyPolicy under specification for "pplicationManager". It should resemble this example:
```
applicationManager:
    enabled: true
    proxyPolicy: CustomProxy
```

4. Save the file and exit.

After this fix Policies are successfully deplyed using Applications>Subscription.


   
  
### Deploying Policies

In RHACM web console go to:  
Applications>Create Application>Subscription

Enter the name for the subscription, can be any arbitrary name which will describe the policies.
Enter the target Namespace on the local cluster where Policies will be deployed.  
Under "Repository location for resources" click Git to choose the repository type.  
Fill in this information:  

URL: https://github.com/AshrafHassan-RH/dev-acm.git
Branch: feature-test
Path: day2/policies  

Expand "Select clusters for application deployment".  
Select radio button "Deploy application resources on clusters with all specified labels".  
Under "cluster sets" click the dropdown menu and choose global. Global is the default readonly clusterset which contains all managed clusters and hub cluster.  
In the textbox "Label" enter "local-cluster"
In the textbox "Value" enter "true"

Once all the information is filled in click "Create" in the header of the dialog.

 
      
 

## Setting up gitops with rhacm:


oc apply -f /bootstrap/rhacm-gitops/01_managedclustersetbinding.yaml  
oc apply -f /bootstrap/rhacm-gitops/02_placement.yaml  
oc apply -f /bootstrap/rhacm-gitops/03_gitopscluster.yaml  


## Deploying workloads with gitops  

Use filter, click checkbox for  Application sets and Subscriptions in the Applications dashboard.  


### Deploying multiclusterobservability using gitops

For observability, namespace called open-cluster-management-observability has to be created prior to creating Application using Applicationset. The objet bucket claim has to be created also, and information like access, key, secret key, endpoint and bucket name have to be provided to gitops/observe/observability.yaml  

In RHACM web console go to:
Applications>Create Application>Applicationset

Follow wizard and enter these details in the forms:
      
**General**  
Name: observability  
Argo server: openshift-gitops  
Requeue time: 180  

**Source**  
Repository type: Git  
URL: https://github.com/dsimonov1/rh-acm.git  
Revision: master  
Path: gitops/observe  

**Destination**  
Remote namespace: default  

**Sync policy**  
Make sure that these are checked:  
Delete resources that are no longer defined in the source repository  
Delete resources that are no longer defined in the source repository at the end of a sync operation  
Automatically sync when cluster state changes  
Automatically create namespace if it does not exist  

**Placement**  
Existing placement: local-cluster  

To remove observability, just delete created Application. 



### Deploy compliance operator using gitops

In RHACM web console go to:  
Applications>Create Application>Applicationset  

Follow wizard and enter these details in the forms:  

**General**  
Name: compliance  
Argo server: openshift-gitops  
Requeue time: 180  

**Source**  
Repository type: Git  
URL: https://github.com/dsimonov1/rh-acm.git  
Revision: master  
Path: gitops/compliance  

**Destination**  
Remote namespace: default  

**Sync policy**  
Make sure that these are checked:  
Delete resources that are no longer defined in the source repository
Delete resources that are no longer defined in the source repository at the end of a sync operation.  
Automatically sync when cluster state changes  
Automatically create namespace if it does not exist  

**Placement**  
Existing placement: all-openshift-clusters  


To remove compliance operator, first remove the aplication from RHAC. Then login to each cluster where compliance operator is installed and run these commands:  
````
oc project openshift-compliance
oc edit profilebundle ocp4  
````
Find the below section and delete the marked line and save the file.
````
spec:
  finalizers:
  - kubernetes   <-- delete this line
````
Do the same with
````
oc edit profilebundle rhcos4
````



On each cluster run these commands 
