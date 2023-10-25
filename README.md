
# RHACM

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

When RHACM is configured to use gitops operator for the deployment of applications, we can use ApplicationSet to deploy any kind of workload. In that case ArgoCD managed by gitops operator on the hub cluster is subscribing to git repos where deployable workloads are defined. These workloads are then deployed to any of the managed clusters using Placements.   


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

## Troubleshooting RHACM

In situations when you cannot import managed cluster due to stale managed cluster ACM services you need to do a cleanup of those services on the managed cluster in order to b able to reimport cluster to ACM. This happens when cluster was detached and ACM servies on managed clusters did not clean up automatically.

The typical error for this situation would be showing as a degraded kube-controller-manager cluster operator. Check the state of the operator using this command:
````
oc get co kube-controller-manager
NAME                      VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
kube-controller-manager   4.12.19   True        False         True       39d     GarbageCollectorDegraded: alerts firing: GarbageCollectorSyncFailed 
````

The solution for this problem might not be straight forward as it might involve multiple steps, especially if ACm was used to deploy certain operators on managed clusters. In that case those opertors need to be cleaned up first. In our example, additional issue was with the terminal-operator.

First part of the soultion is to cleanup the remains of Web Terminal Operator which is descibed in the following chapter.

### Uninstalling the web terminal

Uninstalling the Web Terminal Operator does not remove any of the custom resource definitions (CRDs) or managed resources that are created when the Operator is installed. For security purposes, you must manually uninstall these components. By removing these components, you save cluster resources because terminals do not idle when the Operator is uninstalled.

Uninstalling the web terminal is a two-step process:

1. Uninstall the Web Terminal Operator and related custom resources (CRs) that were added when you installed the Operator.

2. Uninstall the DevWorkspace Operator and its related custom resources that were added as a dependency of the Web Terminal Operator.


You can uninstall the web terminal by removing the Web Terminal Operator and custom resources used by the Operator.

#### Prerequisites

* Access to an OpenShift Container Platform cluster with cluster administrator permissions.

* oc CLI installed on the bastion host

#### Procedure
1. In the Administrator perspective of the web console, navigate to Operators → Installed Operators.

2. Scroll the filter list or type a keyword into the Filter by name box to find the Web Terminal Operator.

3. Click the Options menu kebab for the Web Terminal Operator, and then select Uninstall Operator.

In the Uninstall Operator confirmation dialog box, click Uninstall to remove the Operator, Operator deployments, and pods from the cluster. The Operator stops running and no longer receives updates.

Remove the custom resources using oc CLI:

````
oc delete devworkspaces.workspace.devfile.io --all-namespaces --selector 'console.openshift.io/terminal=true' --wait

oc delete devworkspacetemplates.workspace.devfile.io --all-namespaces  --selector 'console.openshift.io/terminal=true' --wait
````


### Removing the DevWorkspace Operator

To completely uninstall the web terminal, the DevWorkspace Operator and custom resources used by the Operator have to be removed.

The DevWorkspace Operator is a standalone Operator and may be required as a dependency for other Operators installed in the cluster. Follow the steps below only if you are sure that the DevWorkspace Operator is no longer needed.

#### Prerequisites
* Access to an OpenShift Container Platform cluster with cluster administrator permissions.

* oc CLI installed on the bastion host

#### Procedure

1. Remove the DevWorkspace custom resources used by the Operator, along with any related Kubernetes objects:

````
oc delete devworkspaces.workspace.devfile.io --all-namespaces --all --wait

oc delete devworkspaceroutings.controller.devfile.io --all-namespaces --all --wait
````

If this step is not complete, finalizers make it difficult to fully uninstall the Operator.

2. Remove the CRDs used by the Operator:

The DevWorkspace Operator provides custom resource definitions (CRDs) that use conversion webhooks. Failing to remove these CRDs can cause issues in the cluster.

````
oc delete customresourcedefinitions.apiextensions.k8s.io devworkspaceroutings.controller.devfile.io

oc delete customresourcedefinitions.apiextensions.k8s.io devworkspaces.workspace.devfile.io

$ oc delete customresourcedefinitions.apiextensions.k8s.io devworkspacetemplates.workspace.devfile.io

$ oc delete customresourcedefinitions.apiextensions.k8s.io devworkspaceoperatorconfigs.controller.devfile.io
````

3. Verify that all involved custom resource definitions are removed. The following command should not display any output:

````
oc get customresourcedefinitions.apiextensions.k8s.io | grep "devfile.io"
````

4. Remove the devworkspace-webhook-server deployment, mutating, and validating webhooks:

````
oc delete deployment/devworkspace-webhook-server -n openshift-operators

oc delete mutatingwebhookconfigurations controller.devfile.io

oc delete validatingwebhookconfigurations controller.devfile.io
````

If you remove the devworkspace-webhook-server deployment without removing the mutating and validating webhooks, you can not use oc exec commands to run commands in a container in the cluster. After you remove the webhooks you can use the oc exec commands again.

5. Remove any remaining services, secrets, and config maps. Depending on the installation, some resources included in the following commands may not exist in the cluster.

````
oc delete all --selector app.kubernetes.io/part-of=devworkspace-operator,app.kubernetes.io/name=devworkspace-webhook-server -n openshift-operators

oc delete serviceaccounts devworkspace-webhook-server -n openshift-operators

oc delete clusterrole devworkspace-webhook-server

oc delete clusterrolebinding devworkspace-webhook-server
````

6. Uninstall the DevWorkspace Operator:

In the Administrator perspective of the web console, navigate to Operators → Installed Operators.

Scroll the filter list or type a keyword into the Filter by name box to find the DevWorkspace Operator.

Click the Options menu with three dots for the Operator, and then select Uninstall Operator.

In the Uninstall Operator confirmation dialog box, click Uninstall to remove the Operator, Operator deployments, and pods from the cluster. The Operator stops running and no longer receives updates.

### Cleaning up stale RHACM resources

Now proceed with cleaning up stale RHACM resources on the managed cluster. If a cluster is intended to be a HUB cluster of RHACM but later gets added as a MANAGED cluster, the HUB resources need to be properly removed, while keeping the resources required for MANAGED cluster.

Keep in mind that reference cluster also had RHACM hub deployed because we were testing the difference between RHACM versions when we were troubleshooting the issue with no being able to deploy policies.



After properly following the documented steps to remove RHACM and KCS, review the further stale resources associated to RHACM.

1. Enlist all the correlated validatingwebhookconfigurations which were not removed following the above-documented steps. Proceed with removing them manually.

````
oc delete validatingwebhookconfiguration multiclusterengines.multicluster.openshift.io --ignore-not-found

oc delete validatingwebhookconfiguration multiclusterhub.validating-webhook.open-cluster-management.io --ignore-not-found

oc delete validatingwebhookconfiguration ocm.validating.webhook.admission.open-cluster-management.io --ignore-not-found
````

2. Enlist all managedclusterset and remove all finalizer from all the custom resources.

````
oc get managedclustersets

oc patch managedclusterset local-cluster  -p '{"metadata":{"finalizers":null}}' --type=merge
````

3. In case there is no such managedclustersetcustom resouces, proceed with removing the relevant CRDs.

````
oc delete crd managedclustersetbindings.cluster.open-cluster-management.io --ignore-not-found

oc delete crd managedclustersets.cluster.open-cluster-management.io --ignore-not-found
````



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

AAP is installed using policy day2/policies/policy-install-AAP.yaml. The policy installs AAP controller as well as Event Driven Automation (EDA) 

Current AAP controller web console URL is: https://example-ansible-automation-platform.apps.hub-dev-cci.refmobilecloud.ux.nl.tmo/

Current AAP controller login:
admin
mUgwXzWQ59sfvUoML8VwZyd298dQxgsD

Current EDA web console URL is: https://eda-ansible-automation-platform.apps.hub-dev-cci.refmobilecloud.ux.nl.tmo/

Current EDA login:
admin
Jx0Ot0D9btVcqohTZrj9s392dPc6LgVf



Both  URLs can be found using the command:  

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

The same procedure is valid for EDA, you only need to search for the eda-admin-password in the ansible-automation-platform namespace

### Requesting subscription manifest

Once loged in to AAP web console, there is a need to provide a subscription. Since no proxy is configured, and it cannot be configured before subscription is applied,you need to download a subscription manifest from RH portal.

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

To configure a proxy for your automation controller, add the proxy IP addresses to the extra_environment_variables  field in the job  settings page for your automation controller.

#### Procedure

Specify Environment Variables in the AAP UI. These can be found in the Job Settings area:

Settings>Jobs>Extra Environment Variables  
Proxy settings can be placed here as well, in JSON format:

````
{
  "HTTPS_PROXY": "http://10.254.102.198:3128",
  "HTTP_PROXY": "http://10.254.102.198:3128",
  "NO_PROXY": ".cluster.local,.svc,10.128.0.0/14,10.254.164.0/24,10.254.164.248,10.254.171.0/24,10.254.171.11,10.254.171.12,127.0.0.1,172.30.0.0/16,api-int.hub-dev-cci.refmobilecloud.ux.nl.tmo,localhost,refmobilecloud.ux.nl.tmo"
}
````
NOTE 1: Only append the proxy setting to the existing variables, do not delete the ones already defined! 
NOTE 2: Some programs using proxies will expect lower case vs. upper case env variable names. You can avoid this by setting both.
NOTE 3: Some applications will have different implementations of the proxy environment variables. Example: git does not support CIDR notation for no_proxy or some applications will support the domain such as "NO_PROXY": "internal.domain".
NOTE 4: Asterisk sign " * " is not supported while providing the domain information in the environment variable. For example, "*.example.com" will not work but ".example.com" will work for entire domain.
NOTE 5: Not specifying "http://" or "https://" in the proxy address will trigger errors asking for <SCHEMA> in the variable string on some versions of Tower.



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

