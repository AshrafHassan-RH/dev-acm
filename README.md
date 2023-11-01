
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

> **Note** 
> This procedure needs to be done for addons on all managed clusters. The > procedure is done on a hub cluster where klusterletaddonconfig exists in a namespace named after each managed cluster.

For example:
```
oc -n <cluster-name> edit klusterletaddonconfig <cluster-name>
``` 
Where <cluster-name> is the name which can be found in RHACM web console unser Infrastructure>Clusters under "Name" column.
   
  
### Deploying Policies using RHACM Subscription

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

Note: This has been implemented using AAP, job template > integrate gitops with ACM

To integrate GitOps with RHACM, all RHACM managed clusters need to be registered to an instance of GitOps operator running on the hub cluster. Once this is configured, applications can be deployed to those clusters using ArgoCD. To achieve this run the following commands on the hub cluster:

```
oc apply -f /bootstrap/rhacm-gitops/01_managedclustersetbinding.yaml  
oc apply -f /bootstrap/rhacm-gitops/02_placement.yaml  
oc apply -f /bootstrap/rhacm-gitops/03_gitopscluster.yaml  
```

### Deploying RHACM policies with gitops  

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


### RHACM degraded due to stale RHACM repurces on managed cluster 

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

oc delete customresourcedefinitions.apiextensions.k8s.io devworkspacetemplates.workspace.devfile.io

oc delete customresourcedefinitions.apiextensions.k8s.io devworkspaceoperatorconfigs.controller.devfile.io
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



After properly following the documented steps to remove [RHACM](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.8/html/install/installing#uninstalling), review the further stale resources associated to RHACM.

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

### General RHACM troubleshooting




# RHACS
The policy named policy-advanced-cluster-security-central installs ACS operator and central. The placements.yaml is so configured to install this policy on hub cluster only. An additional policy named policy-advanced-cluster-security-managed install only ACS oprator on all managed clusters. Such setup is needed since central component is only installed on the hub cluster whereas secured services are installed on all clusters manually. The requirement for secured services is ACS operator installed.   

The admin password for central services is contained in secret called central-htpasswd in namespace stackrox on the hub cluster.

URL for central Admin console can be found in namespace stackrox>routes on the hub cluster.

Admin console: https://central-stackrox.apps.hub-dev-cci.refmobilecloud.ux.nl.tmo/main/clusters


User: admin
Password: 0VsIj3u75RjY4SVZWdHurv9Gi


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

To configure init-bundle on all  clusters:
````
$ oc create -f rhacs-init-cluster-init-secrets.yaml -n rhacs-operator
````


### Installing secured services

Secured services are installed on all managed clusters that need to be monitored. This usually includes all clusters together with hub cluster.

#### Prerequisites

* RHACS Operator installed
* init bundle generated  and applied to the cluster.


#### Procedure

* On the OpenShift Container Platform web console, navigate to the Operators → Installed Operators page.
* Select "All Projects in the top toolbar
* Click the RHACS Operator.
* Click Secured Cluster from the central navigation menu in the Operator details page.
* Click Create SecuredCluster.
* Make sure Cluster Name is unique across all managed clusters
* Fill in Central Endpoint: central-stackrox.apps.hub-dev-cci.refmobilecloud.ux.nl.tmo:443  
* Click Create.


### Integrating compliance operator with RHACS

Note: This has already been covered by the Ansible Automation Platform in Job Template: rhacs-compliance.

RHACS can be configured to use the Compliance Operator for compliance reporting and remediation with OpenShift Container Platform clusters. That way the results from the Compliance Operator are  reported in the RHACS Compliance Dashboard.

Before applying ScanSettingBinding, the default scan setting needs to be adjusted so receiver pods are not scheduled on the master nodes which cannot provision storage due to missconfiguration/bug in the compliance operator: https://bugzilla.redhat.com/show_bug.cgi?id=1963040

1. Patch the default scan setting to remove previous node selector:
````
oc patch -n openshift-compliance  scansetting/default  --type=merge  -p '{"rawResultStorage": {"nodeSelector": null}}'
````
2. Patch it once again to configure the nodeselector for worker nodes:
````
oc patch -n openshift-compliance  scansetting/default  --type=merge  -p '{"rawResultStorage": {"nodeSelector": {"node-role.kubernetes.io/worker": ""}}}'
```` 
3. Create a ScanSettingBinding object in the openshift-compliance namespace to scan the cluster by using the cis and cis-node profiles:
````
oc apply -f  bootstrap/rhacs-compliance/sscan.yaml -n openshift-compliance
````
4. If compliance-operator was installed after RHACS, redeploy the sensor pod:
````
oc -n stackrox delete pod -lapp=sensor
````

After performing these steps, run a compliance scan in RHACS and ensure that ocp4-cis and ocp4-cis-node results are displayed:
  
1. In RHACS web UI go to Compliance 
2. Click "Manage"standards"
3. Select ocp4-cis and ocp4-cis-node, for this exercise, deselect all other standdards. 


### RHACS cleanup

Sometimes there is a need to have multiple attempts when using policies to deploy operators, especially when developing and testing new policies.
In order to deploy operators using policies we need to cleanup previous deployments by hand.

````
oc delete namespace stackrox

oc get clusterrole,clusterrolebinding,role,rolebinding -o name | grep stackrox | xargs oc delete --wait

oc delete scc -l "app.kubernetes.io/name=stackrox"

oc delete ValidatingWebhookConfiguration stackrox

for namespace in $(oc get ns | tail -n +2 | awk '{print $1}'); do     oc label namespace $namespace namespace.metadata.stackrox.io/id-;     oc label namespace $namespace namespace.metadata.stackrox.io/name-;     oc annotate namespace $namespace modified-by.stackrox.io/namespace-label-patcher-;   done

oc delete namespace rhacs-operator
````

The cleanup usually encompasses uninstalling the operator from the web console and cleaning up namespace created during operator installation.
When deleting namespace rhacs-operator, it it will be stuck in termination phase. The cause of this is that not all resources in the namespace are deleted during operator uninstall.

To address this issue execute these commands:
````
oc api-resources --verbs=list --namespaced -o name | xargs -t -n 1 oc get --show-kind --ignore-not-found -n rhacs-operator
````
The output will show that these resource is not deleted: 
````
NAME                                                                   AGE
central.platform.stackrox.io/stackrox-central-services                 4d3h
securedcluster.platform.stackrox.io/stackrox-secured-cluster-services  4d3h 
````
Delete these resources if found:
````
oc delete central.platform.stackrox.io/stackrox-central-services -n rhacs-operator

oc delete securedcluster.platform.stackrox.io/stackrox-secured-cluster-services 
````

If these resources are also stuck in termination phase, you need to patch them to remove dependency to finalizers:
````
oc patch -n rhacs-operator central.platform.stackrox.io/stackrox-central-services --type=merge -p '{"metadata": {"finalizers":null}}'

oc patch -n rhacs-operator securedcluster.platform.stackrox.io/stackrox-secured-cluster-services --type=merge -p '{"metadata": {"finalizers":null}}'
````



Note: This procedure needs to be done on all clusters.

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

1. Go to Workloads>Serets in the left side menu
2. In the Project dropdown choose ansible-automation-platform
3. Scroll down to secret named example-admin-password
Note: In our case the name of the instance iscalled example. If any other name instane name was used during AAP deployment the name will follow this rule: <instance-name>-admin-password
4. Click on the secret to reveal secret details
5. Scroll down and click Reveal values
6. Copy the secret and use it to login to AAP web console
7. Built in user for accessing the web console is admin 

The same procedure is valid for EDA, you only need to search for the eda-admin-password in the ansible-automation-platform namespace

### Requesting subscription manifest

Once loged in to AAP web console, there is a need to provide a subscription. Since no proxy is configured, and it cannot be configured before subscription is applied,you need to download a subscription manifest from RH portal.

1. Follow the link provided in "subscription allocations" hyperlink
2. Login using your Red Hat Customer Portal credentials
3. From the Subscription Allocations page, click New Subscription Allocation.
4. Enter a name for the allocation so that you can find it later.
5.  Select Type: Satellite 6.8 as the management application.
6. Click Create, This will create the subscription allcation named by the provided name.
7. Click on the tab Subscriptions. 
8. If AAP subscription is not already in the list click Add Subscriptions
9. In the Search box under "Add Subscriptions <Subscription Allocation Name> type "ansible"
10. For subscription named "60 Day Product Trial of Red Hat Ansible Automation Platform, Self-Supported (100 Managed Nodes)" enter number of entitlements: 50
11. Click "Submit"
12. Click "Export Manifest". This will start the download of the zip file containing the manifest.
13. On your AAP instance web console click "Browse" to upload a downloaded manifest.
14. Select the downloaded manifest
15. Click next
16. Under "User and Automation Analytics, deselect all chekboxes and click next
17. Under "End user license agreement" click Submit
18, You should see the empty AAP dashboard using web console

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

### Creating service account on OCP for AAP

In order to allow AAP doing configurations on OCP hub cluster we need to create a service account and clusterrolebinding for that service account.

On bastion host run these commands on th hub cluster:

1. Create service account and clusterrolebinding

````
oc apply -f bootstrap/aap-prerequisites/service-account.yaml
````

2. Get the token for the service account 

````
oc describe secrets aap-service-account-token-qm8dg -n default
````
Note: Secret is created in the default namespace because service account is created in that namespace. Check the AAP/service-account.yaml about the details.
Secrets are created accordign to service account name: <account_name>-token-<random_id>. In our case, secret name is aap-service-account-token-qm8dg 

Copy the token and paste it in the appropriate field descibed in the chapter "Openshift API credentials"

### Adding credentials to AAP

#### Openshift API credentials


1. In the AAP web console go to Resources>Credentials
2. Click Add
3. Enter the credential name under "Name"
4. Under "Credential Type" select "Openshift or Kubernetes API Bearer Token"
5. Under "OpenShift or Kubernetes API Endpoint" enter the API endpoint for the hub cluster: https://api.hub-dev-cci.refmobilecloud.ux.nl.tmo:6443
6. For bearer token, use the one you created by following the procedure in section "Creating service account on OCP for AAP"
7. Paste the copied token in AAP dialogue under "API authentication bearer token"  
8. Uncheck "Verify SSL" in case you are not using "Certificate Authority data" certificate, otherwise paste the CA certificate in the "Certificate Authority data" textbox and leave "Verify SSL" checked
9. Click "Save"
Note: Use this comand to extract ca.crt from the Secret, in case needed
````
oc get secret aap-service-account-token-qm8dg -n default  -o jsonpath="{.data['ca\.crt']}" |base64 -d
````

Note: This procedure needs to be done on all clusters which need to be accessed by AAP.

#### Git credentials

In the AAP web console go to Resources>Credentials
1. Click Add
2. Enter the credential name under "Name" 
3. Under "Credential Type" select "Source control credential"
4. Under "Username", enter your github username.
5. Under "Password"  paste the personal github token.
6. Click "Save"

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

#### Creating a Project 

Ansible Ansible Platform manages different automation tasks using Projects. Project combines and puts into relation Credentials, Inventories/Hosts, and Ansible playbooks in git repos. Project can also be assigned to specific organization/team.

In order to create a Project follow the procedure:
1. In AAP web console go to Resources > Projects
2. Click Add
3. Under "Name" add the name for the project.
4. Click magnifier under "Organization", choose "Infrastructure"
5. Click magnifier under "Execution Environment", choose "Default execution environment"
6. Under "Source Control Type" select "Git". This will open additional configuration texboxes
7. Under "Source Control URL" enter the URL for your repo. Use https URL
8. Under "Source Control Branch/Tag/Commit" provide the name of the branch
9. Click magnifier under  "Source Control Credential", choose the credential name you created following the procedure in "Git credentials" section.
10. Click "Save"       

Note: There is already a project created named AAP Demo. The project should provide a showcase for a couple of usecases.

#### Creating Job Templates

Job Templates 

In the Ansible Automation Platform (AAP), a job template is a fundamental concept that defines how Ansible should execute automation tasks. It provides a way to configure and control the execution of Ansible playbooks, job runs, and tasks. Job templates are used to streamline and automate the process of running Ansible automation on various systems, making it easier to execute tasks and enforce consistency in execution.


To configure Job Template follow the procedure:
1. In AAP web console go to Resources > Templates
2. Under "Name" add the name for the Job Template
3. Click magnifier under "Inventory", choose localhost
4. Click magnifier under "Project", choose the project you created following the procedure in "Creating a project" section.
5. Click magnifier under "Playbook", choose the playbook you want to use with your Template. In our example we use the playbook named acm_gitops.yaml which covers one of our use-cases. The playbook is fetched from github repo using the credentials created.
6. Click magnifier under "Credentials"
7. For "Selected Category" choose Openshift or Kubernetes API Token.
8. Choose the credential created by following the proceure in "Openshift API credentials" section
9. Click "Save"
   

Note: There is already a Job Template named "integrate gitops with ACM" created. This job template is configuring the integration for RHACM and gitops. It applies a couple of yaml manifests in order to enable Application set for Applications in RHACM. This feature allows us to deploy applications to managed clsuters using ArgoCD. 

#### Running Jobs

Once all above configuration have bee applied, we can now run the defined Job template

To run a Job follow the procedure:
1. In AAP web console go to Resources > Templates
2. On the right side in a row for a template you want to run, click the rocket icon
3. The output dialog will open. You can follow the output of the job execution

Hint: You can increase verboseness of the output:
1. In AAP web console go to Resources > Templates
2. On the right side in a row for a template you want to edit, click pen icon
3. Under "Verbosity" , select "Debug" in the dropdown menu
4. Click "Save"

 
### Configuring EDA (Event Driven Ansible)


#### Creating decision environmrnt

We will use the default decsision environment container image

1. In EDA click on Resources > Decision Environments
2. Click "Create decision Environment"
3. Enter the following details in the dialog:
Name: default 
Description: default
Image: quay.io/ansible/ansible-rulebook
4. Click "Save"


#### Creating access token on AAP for EDA

1. Go to Administration --> Applications

2. Click "Add Application"

3. Fill in the details as follows: 

Name: Personal Access Token for EDA
Description: Personal Access Token
Organization: Default
Authorization Grant Type: Resource owner password-based
Client Type: Public

4. Click "Save".  A token will be created.
5. Go to Users --> Admin --> Tokens
6. Click "Add"
7. Click the magnifier icon and choose "Personal Access Token for EDA"
8. Under "Scope" choose > Write
9. Click "Save".  A token and refresh token will be shown.  Save this as it will be needed later.




#### Connecting EDA to AAP Controller using access token 

1. On the EDA, go to Users --> Admin --> Controller Tokens.  
2. Click "Create controller token"
3. Fill in the following information
Name: Connection to AAP
Desription: Connection to AAP
Token: Copied/pasted from output in step 9. under "Creating access token on AAP for EDA"
4. Click "Create controller token"

#### Create Project in EDA

We will use sample provided in https://github.com/kcalliga/pvcresize-eda

1. In EDA got to Resources > Projects
2. Click "Create Project"
3. Enter the following information in the dialog:
Name: resizepvc
Description: resizepvc
SCM Type: Git
SCM URL: https://github.com/kcalliga/pvcresize-eda
Credential: No credential since this is a public repo
4. Click "Create Project"
