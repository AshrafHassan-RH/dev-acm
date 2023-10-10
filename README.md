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
