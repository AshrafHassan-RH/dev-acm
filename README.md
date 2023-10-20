


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

### Issue with Policies deployed using Subscription on a cluster with internet proxy

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

3. Add proxyPolicy under specification for "applicationManager". It should resemble this example:
```
applicationManager:
    enabled: true
    proxyPolicy: CustomProxy
```

4. Save the file and exit.

After this fix Policies are successfully deplyed using Applications>Subscription.

Note: This procedure needs to be done for addons on all managed clusters. The procedure is done on a hub cluster where klusterletaddonconfig exists in a namespace named after each managed cluster.

For example:
```
oc -n <cluster-name> edit klusterletaddonconfig <cluster-name>
``` 
Where <cluster-name> is the name which can be found in RHACM web console unser Infrastructure>Clusters under "Name" column.
   
  
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

 
      
## Using GitOps operator to deploy applications via RHACM 

### Setting up GitOps with RHACM

To integrate GitOps with RHACM, all RHACM managed clusters need to be registered to an instance of GitOps operator running on the hub cluster. Once this is configured, applications can be deployed to those clusters using ArgoCD. To achieve this run the following commands on the hub cluster:

```
oc apply -f /bootstrap/rhacm-gitops/01_managedclustersetbinding.yaml  
oc apply -f /bootstrap/rhacm-gitops/02_placement.yaml  
oc apply -f /bootstrap/rhacm-gitops/03_gitopscluster.yaml  
```

### Deploying workloads with gitops  

When RHACM is configured to use gitops operator for the deployment of applications, we can use ApplicationSet to deploy any kind of workload. In that case ArgoCD managed bygitops operator on the hub cluster is subscribing to git repos where deployable workloads are defined. These workloads are then deployed to any of the managed clusters using Placements.   


#### Deploying multiclusterobservability using gitops

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
URL: https://github.com/AshrafHassan-RH/dev-acm.git
Revision: feature-test
Path: day2/apps/observe/

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



#### Deploy compliance operator using gitops

In RHACM web console go to:  
Applications>Create Application>Applicationset  

Follow wizard and enter these details in the forms:  

**General**  
Name: compliance  
Argo server: openshift-gitops  
Requeue time: 180  

**Source**  
Repository type: Git  
URL: https://github.com/AshrafHassan-RH/dev-acm.git
Revision: feature-test
Path: day2/apps/compliance/  

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


To remove compliance operator, first remove the aplication from RHACM. Then login to each cluster where compliance operator is installed and run these commands:  
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

These commands need to be run on each cluster where compliance operator was deployed. 


## RHACS
The policy named policy-advanced-cluster-security-central installs ACS operator and central. The placements.yaml is so configured to install this policy on hub cluster only. An additional policy named policy-advanced-cluster-security-managed install only ACS oprator on all managed clusters. Such setup is needed since central component is only installed on the hub cluster whereas secured services are installed on all clusters manually. The requirement for secured services is ACS operator installed.   

The admin password for central services is contained in secret called central-htpasswd in namespace stackrox on the hub cluster.

URL for central Admin console can be found in namespace stackrox>routes on the hub cluster.

Admin console: https://central-stackrox.apps.hub-dev-cci.refmobilecloud.ux.nl.tmo/main/clusters


User: admin
Password: G6s1jsJLZmZyPjYIaz0f9EjA3


### Generating init-bundle

You must have the Admin user role to create an init bundle.

#### Procedure

* On the RHACS portal, navigate to Platform Configuration → Integrations.
* Navigate to the Authentication Tokens section and click on Cluster Init Bundle.
* Click Generate bundle.
* Enter a name for the cluster init bundle and click Generate.
* Click Download Kubernetes Secret File to download the generated bundle.
* Either copy the file or copy it's content and create a new file on the host with access to oc client.
* Using the Red Hat OpenShift CLI, run the following command to create the resources:

To configure init-bundle on hub cluster:
````
$ oc create -f rhacs-init-cluster-init-secrets.yaml -n stackrox
````
To configure init-bundle on managed clusters:
````
oc apply -f RHACS/rhacs-init-cluster-init-secrets.yaml -n rhacs-operator
````

### Installing secured services

Secured services are installed on all managed clusters that need to be monitore. This usually includes all clusters together with hub cluster.

#### Prerequisites

* RHACS Operator installed
* init bundle generated  and applied to the cluster.


#### Procedure

* On the OpenShift Container Platform web console, navigate to the Operators → Installed Operators page.
* Click the RHACS Operator.
* Click Secured Cluster from the central navigation menu in the Operator details page.
* Click Create SecuredCluster.
* Make sure Cluster Name is unique across all managed clusters
* Fill in Central Endpoint: central-stackrox.apps.hub-dev-cci.refmobilecloud.ux.nl.tmo:443  
* Click Create.


### Integrating compliance operator with RHACS
 

### RHACS cleanup

Sometimes there is a need to have multiple attempts when using policies to deploy operators, especially when developing and testing new policies.
In order to deploy operators using policies we need to cleanup previous deployments by hand.

The cleanup usually encompasses uninstalling the operator from the web console and cleaning up namespace created during operator installation.
When deleting namespace rhacs-operator, it iwll be stuck in termination phase. The cause of this is that not all resources in the namespace are deleted during operator uninstall.

To address this issue execute these commands:
````
oc api-resources --verbs=list --namespaced -o name | xargs -t -n 1 oc get --show-kind --ignore-not-found -n rhacs-operator
````
The output will show that this resource is not deleted: 
````
NAME                                                     AGE
central.platform.stackrox.io/stackrox-central-services   4d3h
````
Delete that resource
````
oc delete central.platform.stackrox.io/stackrox-central-services -n rhacs-operator
````
Delete the namespace if not already deleted
````
oc delete project rhacs-operator
````

## AAP

AAP is installed using policy day2/policies/policy-install-AAP.yaml

Current AAP web console URL is: https://example-ansible-automation-platform.apps.hub-dev-cci.refmobilecloud.ux.nl.tmo/

Current login:
admin
mUgwXzWQ59sfvUoML8VwZyd298dQxgsD

The AAP URL can be found using the command:  

````
oc get route -n ansible-automation-platform
````

The admin user and password can be fetched from the secret in ansible-automation-platform namespace using web console:

* Go to Workloads>Serets in the left side menu
* In the Project dropdown choose ansible-automation-platform
* Scroll down to secret named example-admin-password
Note: In our case the name of the instance iscalled example. If any other name instane name was used during AAP deployment the name will follow this rule: <instance-name>-admin-password
* Click on the secret to reveal secret details
* Scroll down and click Reveal values
* Copy the secret and use it to login to AAP web console
* Built in user for accessing the web console is admin 

### Requesting subscription manifest

One loged in to AAP web console, there is a need to provide a subscription. Since no proxy is configured, and it cannot be configured before subscription is applied,you need to download a subscription manifest from RH portal.

* Follow the link provided in "subscription allocations" hyperlink
* Login using your Red Hat Customer Portal credentials
* From the Subscription Allocations page, click New Subscription Allocation.
* Enter a name for the allocation so that you can find it later.
* Select Type: Satellite 6.8 as the management application.
* Click Create, This will create the subscription allcation named by the provided name.
* Click on the tab Subscriptions. 
* If AAP subscription is not already in the list click Add Subscriptions
* In the Search box under "Add Subscriptions <Subscription Allocation Name> type "ansible"
* For subscription named "60 Day Product Trial of Red Hat Ansible Automation Platform, Self-Supported (100 Managed Nodes)" enter number of entitlements: 50
* Click "Submit"
* Click "Export Manifest". This will start the download of the zip file containing the manifest.
* On your AAP instance web console click "Browse" to upload a downloaded manifest.
* Select the downloaded manifest
* Click next
* Under "User and Automation Analytics, deselect all chekboxes and click next
* Under "End user license agreement" click Submit
* You should see the empty AAP dashboard using web console

### Configuring proxy for AAP

To configure a list of known proxies for your automation controller, add the proxy IP addresses to the PROXY_IP_ALLOWED_LIST field in the settings page for your automation controller.

#### Procedure

On your automation controller, navigate to Settings → Miscellaneous System.
In the PROXY_IP_ALLOWED_LIST field, enter IP addresses that are allowed to connect to your automation controller, following the syntax in the example below:

Example PROXY_IP_ALLOWED_LIST entry

[
  "example1.proxy.com:8080",
  "example2.proxy.com:8080"
]
Important:

PROXY_IP_ALLOWED_LIST requires proxies in the list are properly sanitizing header input and correctly setting an X-Forwarded-For value equal to the real source IP of the client. Automation controller can rely on the IP addresses and hostnames in PROXY_IP_ALLOWED_LIST to provide non-spoofed values for the X-Forwarded-For field.
Do not configure HTTP_X_FORWARDED_FOR as an item in `REMOTE_HOST_HEADERS`unless all of the following conditions are satisfied:

* You are using a proxied environment with ssl termination;
* The proxy provides sanitization or validation of the X-Forwarded-For header to prevent client spoofing;
* /etc/tower/conf.d/remote_host_headers.py defines PROXY_IP_ALLOWED_LIST that contains only the originating IP addresses of trusted proxies or load balancers. 


### Adding credentials to AAP

#### Openshift API credentials

* In the AAP web console go to Resources>Credentials
* Click Add
* Enter the credential name (arbitrary name)
* Under "Credential Type" select "Openshift or Kubernetes API Bearer Token"
* Under "OpenShift or Kubernetes API Endpoint" enter the API endpoint for the hub cluster: https://api.hub-dev-cci.refmobilecloud.ux.nl.tmo:6443
* For bearer token you can either create a new service account or use a token from any OpenShift user. To obtain the token for existing user, log in ti OpenShift web console, in the top left corner click on the username, in the dropdown click "Copy login command". Copy the API token.
* Paste the copied token in AAP dialogue under "API authentication bearer token"  
* Uncheck "Verify SSL" in case you are not using "Certificate Authority data" certificate, otherwise paste the CA certificate in the "Certificate Authority data" textbox and leave "Verify SSL" checked

#### Git credentials

In the AAP web console go to Resources>Credentials
* Click Add
* Enter the credential name (arbitrary name)
* Under "Credential Type" select "Source control credential"
* Under "OpenShift or Kubernetes API Endpoint" enter the API endpoint for the hub cluster: https://api.hub-dev-cci.refmobilecloud.ux.nl.tmo:6443
* For bearer token you can either create a new service account or use a token from any OpenShift user. To obtain the token for existing user, log in ti OpenShift web console, in the top left corner click on the username, in the dropdown click "Copy login command". Copy the API token.
* Paste the copied token in AAP dialogue under "API authentication bearer token"
* Uncheck "Verify SSL" in case you are not using "Certificate Authority data" certificate, otherwise paste the CA certificate in the "Certificate Authority data" tex
tbox and leave "Verify SSL" checked

#### Adding bastion host to AAP

In order to execute automation manifests form AAP against OpenShift clusters we ned to add bastion host to AAP.

We first need to create an inventory. In AAP web console:

* Go to Resources>Inventory
* Click Add
* Under Name enter: bation inventory
* Under Labels enter: bastion
* Click Save

To add host to created inventory:
* Go to Resources>Hosts
* Click Add
* Under Name enter: Bastion host
* In Inventory dropdown box click magnifier and select bastion inventory
* Under Variables add th IP address of the bastion host:  




## RHACM RBAC

The use case should setup a user "user1" using htpasswd identity provider. The user should be able to deploy apps on all clusters but should not be able to do anything to the infrastructure. user1 account is a prerequisite  


oc new-project  apps-user1

oc adm groups new group1

oc adm groups add-users group1 user1

oc create clusterrolebinding hub-cluster-view  --clusterrole=open-cluster-management:view:local-cluster --user=user1
oc create clusterrolebinding prod-dev-cluster-admin  --clusterrole=open-cluster-management:admin:prod-dev-cci --user=user1
oc create clusterrolebinding ref-cluster-view  --clusterrole=open-cluster-management:view:reference-cluster  --user=user1

oc create rolebinding apps-user1-admin -n apps-user1 --clusterrole=admin --user=user1 - Allows user1 to create subscriptions in namespace apps-user1

oc create clusterrolebinding  view-global-clusterset --clusterrole=open-cluster-management:managedclusterset:bind:global --user

oc apply -f clusterrole.yaml

oc apply -f rolebinding.yaml

Check who can do what

oc adm policy who-can create cluster.open-cluster-management.io/placements



#REMOved 
removed oc create clusterrolebinding managedclusterset-global-view  --clusterrole=open-cluster-management:managedclusterset:view:global --user=user1

this one is removed:oc create clusterrolebinding subscription-admin --clusterrole=open-cluster-management:subscription-admin --user=user1

removed oc create clusterrolebinding create-placements --clusterrole=placements.cluster.open-cluster-management.io:create --user=user1

removed oc create clusterrolebinding create-apps --clusterrole=applications.app.k8s.io:create --user=user1

removed oc create clusterrolebinding create-deployments --clusterrole=deployables.apps.k8s.io:create --user=user1


applicationsets.argoproj.io is forbidden: User "user1" cannot create resource "applicationsets" in API group "argoproj.io" in the namespace "openshift-gitops": RBAC: [clusterrole.rbac.authorization.k8s.io "open-cluster-management:managedclusterset:admin:global" not found, clusterrole.rbac.authorization.k8s.io "placements.cluster.open-cluster-management.io:edit" not found]

[root@cci-dev-eng ~]# oc edit profilebundle rhcos4^C
[root@cci-dev-eng ~]# oc create clusterrolebinding user1-create-apps  --clusterrole=open-cluster-management:cluster-manager-admin --user=user1
clusterrolebinding.rbac.authorization.k8s.io/user1-create-apps created
[root@cci-dev-eng ~]# oc delete clusterrolebinding user1-create-apps
clusterrolebinding.rbac.authorization.k8s.io "user1-create-apps" deleted
[root@cci-dev-eng ~]# oc create clusterrolebinding user1-create-apps  --clusterrole=open-cluster-management:cluster-manager-admin --user=user1
clusterrolebinding.rbac.authorization.k8s.io/user1-create-apps created
[root@cci-dev-eng ~]# oc create clusterrolebinding <role-binding-name> --clusterrole=open-cluster-management:subscription-admin --user=<username>
-bash: syntax error near unexpected token `newline'
[root@cci-dev-eng ~]# oc create clusterrolebinding <role-binding-name> --clusterrole=open-cluster-management:subscription-admin --user=<username>
-bash: syntax error near unexpected token `newline'
[root@cci-dev-eng ~]# oc create clusterrolebinding user1-create-apps --clusterrole=open-cluster-management:subscription-admin --user=user1
error: failed to create clusterrolebinding: clusterrolebindings.rbac.authorization.k8s.io "user1-create-apps" already exists
[root@cci-dev-eng ~]# oc delete clusterrolebinding user1-create-apps
clusterrolebinding.rbac.authorization.k8s.io "user1-create-apps" deleted
[root@cci-dev-eng ~]# oc create clusterrolebinding user1-create-apps --clusterrole=open-cluster-management:subscription-admin --user=user1
clusterrolebinding.rbac.authorization.k8s.io/user1-create-apps created
[root@cci-dev-eng ~]# oc delete clusterrolebinding user1-create-apps
clusterrolebinding.rbac.authorization.k8s.io "user1-create-apps" deleted
[root@cci-dev-eng ~]# oc create clusterrolebinding user1-create-apps --clusterrole=open-cluster-management:admin:prod-dev-cci --user=user1
clusterrolebinding.rbac.authorization.k8s.io/user1-create-apps created
[root@cci-dev-eng ~]# oc delete clusterrolebinding user1-create-apps
clusterrolebinding.rbac.authorization.k8s.io "user1-create-apps" deleted
[root@cci-dev-eng ~]# oc create clusterrolebinding user1-prod-dev-cluster --clusterrole=open-cluster-management:admin:prod-dev-cci --user=user1
clusterrolebinding.rbac.authorization.k8s.io/user1-prod-dev-cluster created

