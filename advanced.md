# Advanced

## Table of contents

[Overview](#overview)

[practice 1: Network Policies](#practice-1-network-policies)

[Practice 2: ClusterRole,ClusterRoleBinding-Role,RoleBinding](#practice-2-clusterrole,-clusterrolebinding-role,-rolebinding)

[practice 3: Cluster Autoscaler on Azure](#practice-3-cluster-autoscaler-on-azure)

[practice 4:  Qudos](#practice-4-qudos)

## Overview

**Network Plugin**

By default, Kubernetes has an open network where every pod can talk to every pod.
Network policies are implemented by the network plugin, so you must use networking solution which supports NetworkPolicy - simply creating the resource without a controller to implement it will have no effect.

**Network Policies**

By default, Kubernetes has an open network where every pod can talk to every pod.

Create network policies that control traffic from one pod to another or from an IP outside of the cluster. Enable this to control your network using network policies.

**ClusterRole,ClusterRoleBinding-Role,RoleBinding**

If we want to apply the ClusterRole, ClusterRoleBinding, Role, RoleBinding, the cluster should be RBAC enabled.

RBAC (Role Based Access Cluster) is used to restrict the access to the user on Cluster. It only allows admins to configure the policies using Kubernetes API *rbac.authorization.k8s.io*.

By default AKS(Azure Kubernetes Service) cluster comes with RBAC enabled.

And to apply these ClusterRole, ClusterRoleBinding, Role, RoleBinding, integrate the cluster with Azure Active Directory (AAD) while creating.

**Role and ClusterRole**

Role is used to grant access to resources within a single namespace.

ClusterRole is used to grant the same permissions as a role, but it can be applied in the cluster level not just in namespace level.

**RoleBinding and ClusterRoleBinding**

Once a role is defined to grant permissions to resources, you can assign those permissions using a rolebinding for a given namespace.

A ClusterRoleBinding works in the same way to bind roles to users, but it can be applied to resources across the entire cluster, not just a specific namespace.

**cluster autoscaler on Azure**

The cluster autoscaler on Azure scales worker nodes within any specified autoscaling group. It will run as a Kubernetes deployment in your cluster.

**Qudos**

 **What is Qudos?**

* Qloudable Developer operating name spaces `QuDoS` solves the problem of automating recurring tasks for developers, before code goes into integration environment.

**What Qudos can automate?**

* Changing environment variables to point to required dependencies like db or other services.
* Pushing local changes to remote service.
* Forwarding cluster services to appear to be local services.
* Deploying changes after a save to the environment in cluster.
* Pulling other peoples code so conflicts are resolved early.
* Committing changes on each save to local git repo.
* But connecting to the debugged environment.
* Running some node processes.
* Restarting node process, restarting debugger, checking the old break points and variables.
    
**What Qudos cannot automate?**

* Creating a local workspace, by cloning and checking out a branch from GitHub may be manual.
* Pushing changes to GitHub has to be manual.
* Debugging has to be manual. 



## practice 1: Network Policies

By default, Kubernetes has an open network where every pod can talk to every pod.
Network policies are implemented by the network plugin, so you must use networking solution which supports NetworkPolicy - simply creating the resource without a controller to implement it will have no effec

### Create the Namespace and a Sample Service

1. Create a sample namespace.

    ` kubectl create namespace test`

2. Create a deployment with 2 replicas using nginx image.

    ` kubectl -n test run nginx --replicas=2 --image=nginx`

3. Verify whether the pods are created or not
 
    ` kubectl -n test get pods`

4. Create a service and expose it with port 80.

    ` kubectl expose -n test deployment nginx --port=80`

5. You can see the service using the below command.

    ` kubectl -n test get services`

**Network Policies**

By default, Kubernetes has an open network where every pod can talk to every pod.

Create network policies that control traffic from one pod to another or from an IP outside of the cluster. Enable this to control your network using network policies.

### step 1: Deny all ingress and egress traffic

1. **Deny all ingress traffic:** Block all ingress traffic on the namespace by deploying the below YAML code.

``` yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: test
  name: deny-ingress
spec:
  podSelector: 
    matchLabels: {}
  policyTypes:
  - Ingress
```

2. Save the YAML code in file and deploy it.

    ` kubectl create -f <file path>`


3. **Deny all egress traffic:** Block all egress traffic on the namespace by deploying the below YAML code.

``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  namespace: test
  name: deny-egress
spec:
  podSelector: 
    matchLabels: {}
  policyTypes:
  - Egress
```

4. Save the YAML code in file and deploy it.

    ` kubectl create -f <file path>`

### step 2: Verify access - Denied all Ingress and Denied all Egress

Now all Ingress and Egress traffic is blocked on all the pods in the namespace.

1. Open a new shell and create a busybox pod to test the policies access.

    `kubectl -n test run busybox --rm -ti --image=busybox /bin/sh`

2. This will open up a shell session and verify the ingress and egress traffic.

3. **Verify Ingress:** Now you are in the busybix pod shell, send the ingress traffic to the nginx service using the below command.

    `wget -q --timeout=5 nginx -O -`

    It should return 
    ` wget: bad address 'nginx'`

4. **Verify egress:** Send the egress traffic to the pod using the below command.

    `wget -q --timeout=5 google.com -O -`

    It should return 
    `wget: bad address 'google.com'`

### step 3: Allow DNS egress traffic

1. **Allow DNS egress traffic:** Run the following command to create a label of name: kube-system on the kube-system namespace and a  NetworkPolicy, which allows DNS egress traffic from any pods in your namespace to the kube-system namespace.

2. Execute the following command before applying this network policy

    `kubectl label ns kube-system kubesystem=kubesystem`

3. Deploy the below YAML file to allow egress traffic from your namspace to kube-system namespace.

```  yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: test
  name: allow-dns
spec:
  podSelector: 
    matchLabels: {}
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kube-system: kube-system
    - podSelector:  
        matchLabels:
          k8s-app: kube-dns
```

### step 4: Verify Access - Allowed DNS access

Now all egress traffic is allowed to DNS.

1. In the new shell, exec into the busybox pod and test the egress traffic.
    
    `kubectl -n test run busybox --rm -ti --image=busybox /bin/sh`

2. Now your in busy box pod shell, verify the egress traffic.

    `nslookup nginx`

### step 5: Allow ingress and egress traffic within the Namespace:

1. **Allow ingress traffic within the Namespace:** Run the following to create a? NetworkPolicy, ?which allows ingress traffic from any pods within the namespace using the following file.

```  yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: test
  name: allow-from-same-namespace-and-ingress
spec:
  podSelector:
    matchLabels: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          run: nginx
    - podSelector: {}
```

2. **Allow egress traffic within the Namespace:** Run the following to create a ?NetworkPolicy, which allows egress traffic from any pods within the Namespace using the following YAML file.

```  yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: test
  name: allow-from-same-namespace-and-egress
spec:
  podSelector:
    matchLabels: {}
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          run: nginx
    - podSelector: {}
```

### step 6: Verify Access - Allowed ingress and egress traffic within the Namespace

Now all the ingress and egress is allowd within the namespace.

1. In the new shell, exec into the busybox pod and test the egress traffic.
    
    `kubectl -n test run busybox --rm -ti --image=busybox /bin/sh`

2. **Verify Ingress and Egress:** Now you are in the busybox pod shell, send the ingress traffic to the nginx service and egress traffic from busybox using the below command.

    `wget -q --timeout=5 nginx -O -`

    It should return 
    ```
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>...
    ```

### step 7: Allow internet access

1. Allow Internet for required services that access outside services using the following YAML file.

```  yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: test
  name: allow-nginx
spec:
  podSelector:
    matchLabels:
      run: nginx
  egress:
  - {} 
  policyTypes: 
  - Egress
```

### step 8: Verify the internet access

1. Get the nginx pods.

    `kubectl -n test get pods`

2. Exec into the pod using one of the pod name.

    `kubectl -n test exec -it <podname> bash`

3. Verify the internet access to the nginx pod.

    `apt-get update`

    It should return 

    ```
    Get:1 http://security-cdn.debian.org/debian-security stretch/updates InRelease [94.3 kB]
    Get:4 http://security-cdn.debian.org/debian-security stretch/updates/main amd64 Packages [487 kB]
    Ign:2 http://cdn-fastly.deb.debian.org/debian stretch InRelease
    Get:3 http://cdn-fastly.deb.debian.org/debian stretch-updates InRelease [91.0 kB]
    ```

You can see all the network policies which you applied using the below command.

    `kubectl -n test get networkpolicies`


## Practice 2: ClusterRole,ClusterRoleBinding-Role,RoleBinding

There are different ways to authenticate with and secure Kubernetes clusters. Using role-based access controls (RBAC), you can grant users or groups access to only the resources they need.

By default AKS(Azure Kubernetes Service) cluster comes with RBAC enabled and the cluster can be configured to use Azure Active Directory (AD) for user authentication.

RBAC (Role Based Access Cluster) is used to restrict the access to the user on Cluster. It only allows admins to configure the policies using Kubernetes API *rbac.authorization.k8s.io*.

With RBAC, you create roles to define permissions, and then assign those roles to users with role bindings.

And to apply these ClusterRole, ClusterRoleBinding, Role, RoleBinding, the clsuetr must be integrated with Azure Active Directory (AAD) while creating.

In this Lab, the cluster is already integrated with Azure AD and deployed, now create roles and rolebindings to test the RBAC.

### Role and ClusterRole

Role is used to grant access to resources within a single namespace.

ClusterRole is used to grant the same permissions as a role, but it can be applied in the cluster level not just in namespace level.

### RoleBinding and ClusterRoleBinding

Once a role is defined to grant permissions to resources, you can assign those permissions using a rolebinding for a given namespace.

A ClusterRoleBinding works in the same way to bind roles to users, but it can be applied to resources across the entire cluster, not just a specific namespace.

## step 1: Create ClusterRoleBinding using default ClusterRole.

1. Create a ClusterRoleBinding for the user who you want to grant access to the AKS cluster.

2. In this lab, we created one test user to test the RBAC. Below are the test user login credentials

3. Now create a ClusterRoleBinding for the above user with predefined ClusterRole that is available on the cluster.

4. Create a group in AAD(Azure Active Directory) and add the test user to that group who can have cluster-admin access and take the object id from overview.

5. Now assign ClusterRoleBinding to the AAD group by entering the name of the ClusterRoleBinding, ClusterRole(Cluster-admin) and group Object Id in the following ClusterRoleBinding.

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
 name: <Name-of-the-ClusterRoleBinding>
roleRef:
 apiGroup: rbac.authorization.k8s.io
 kind: ClusterRole
 name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: <Object-ID-of-the-AAD-group>
```

6. Save the above YAML code in file and Create ClusterRoleBinding using the below command.

    `kubectl create -f <name of the file>`

### step 2: Access cluster with Azure AD

1. Now acces the cluster with non-admin user using az aks get-credentials command.

`az aks get-credentials --resource-group <myResourceGroup> --name <myAKSCluster>`

2. Run any cluster command with kubectl, for example get the nodes using `kubectl get nodes`

3. After you run a kubectl command, you are prompted to authenticate with Azure. Follow the on-screen instructions to complete the process. Here you need to aunthenticate the cluster using test user credntials which are provided in previous steps.

```
kubectl get nodes

To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code BUGHYDGNL to authenticate.

NAME                       STATUS    ROLES     AGE       VERSION
aks-nodepool1-79590246-0   Ready     agent     1h        v1.13.5
```

### step 3: Create Role and RoleBinding

1. Now test the RBAC in namespace level by assigning the role and rolebinding to the user.

2. Here we are restricting the user to a particular namespace in the AKS cluster using role and rolebinding.

3. Use the following test user to test the namespace level RBAC.

4. Create another group in AAD(Azure Active Directory) and add this user to that group who can have only namespace level access and take the object id.

5. Create a sample namespace using `kubectl create ns <namespace name>` and update the namespace name in the below yaml file.

5. Create a role using the below YAML which has only access to pods.

6. Enter the name of the role and namespace.

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: <name-of-the-role>
  namespace: <namespace>
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - list
  - watch
  - delete
  - create
- apiGroups:
  - "extensions"
  resources:
  - deployments
  verbs:
  - get
  - list
  - watch
  - delete
  - create

```
4. Once role is defined to grant permissions to resources, assign that role to the AAD group with RoleBinding by specifing the rolebinding name, namespace name and object ID of AAD group.

```  yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: <rolebinding-name>
  namespace: <namespace>
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kube-dev-role
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: <object ID of AAD group>

```

3. Save the above YAML code in file and Create RoleBinding using the below command.

    `kubectl create -f <name of the file>`

### step 2: Access cluster resources with Azure AD

1. Now acces the cluster with non-admin user using az aks get-credentials command.

`az aks get-credentials --resource-group <myResourceGroup> --name <myAKSCluster>`

2. Now run any command in the specified namespace which user has access.Example `kubectl -n <namespace name> get po`

3. After you run a kubectl command, you are prompted to authenticate with Azure. Follow the on-screen instructions to complete the process. Here you need to aunthenticate the cluster using test user credntials which are provided in previous steps.

```
kubectl -n test get pods

To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code BUCFYDGNL to authenticate.

No resources found.

```
4. Now the user got authenticated to the cluster and has only access to namespace which he assigned to.

5. check the other namespace using `kubectl -n sample get po` get this unauthorised error.

```
 kubectl -n sample get po
 
Error from server (Forbidden): pods is forbidden: User "nsuser@sysgain.com" cannot list resource "pods" in API group "" in the namespace "sample"

```

## practice 3: Cluster Autoscaler on Azure

The cluster autoscaler on Azure scales worker nodes within any specified autoscaling group. It will run as a Kubernetes deployment in your cluster.

## Kubernetes Version

Kubernetes v1.10.X and Cluster autoscaler v1.2+  are required to run on Azure.

Cluster autoscaler supports four VM types with Azure cloud provider:
- **aks**: Managed Kubernetes Service([AKS](https://docs.microsoft.com/en-us/azure/aks/))


### CA Version

You need to replace a placeholder, '{{ ca_version }}' in manifests with CA Version such as v1.2.2.

### Permissions

Get azure credentials by running the following command

```sh
# replace <subscription-id> with yours.
az ad sp create-for-rbac --role="Contributor" --scopes="/subscriptions/<subscription-id>" --output json
```

### step 1: Deployment manifests


### AKS deployment for Availability set

**Pre-requirements:**

- Get credentials from above `permissions` step.
- Get the cluster name using the following:
  - for AKS: `az aks list`

- Get a node pool name by extracting the value of the label **agentpool**
  ```sh
  kubectl get nodes --show-labels
  ```
- cas.yaml is at below url, change the secret settings to your azure AKS instance

  [view cas.yaml file](https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/azure/examples/cluster-autoscaler-containerservice.yaml)

- Encode each data with base64. using the website  https://www.base64encode.org/

Fill the values of cluster-autoscaler-azure secret in [cluster-autoscaler-containerservice](examples/cluster-autoscaler-containerservice.yaml), including

- ClientID: `<base64-encoded-client-id>`
- ClientSecret: `<base64-encoded-client-secret>`
- ResourceGroup: `<base64-encoded-resource-group>` (Note: ResourceGroup is case-sensitive)
- SubscriptionID: `<base64-encode-subscription-id>`
- TenantID: `<base64-encoded-tenant-id>`
- ClusterName: `<base64-encoded-clustername>`
- VMType: `<base64-encoded-vmtype>`
- NodeResourceGroup: `<base64-encoded-node-resource-group>` (only for AKS with VMAS)

> Note that all data above should be encoded with base64.

And fill the node groups in container command by `--nodes`, with the range of nodes (minimum to be set as 3 which is the default cluster size) and node pool name obtained from pre-requirements steps above, e.g.

```yaml
        - --nodes=1:5:nodepool1
```

The `vmType` param determines the kind of service we are interacting with:

### For AKS

$ echo -n AKS | base64
QUtT
```

The `NodeResourceGroup` param is only for AKS with VMAS, it should be in format `MC_<resource-group>_<cluster-name>_<location>`. Note the param is case sensitive and should be encoded with base64.

Then deploy cluster-autoscaler by running

```sh
kubectl create -f cas.yaml
```

### step 2: Scale up the node

  ### Get the nodes in a cluster.
  
  `kubectl get nodes`
  
```
NAME                        STATUS    ROLES     AGE       VERSION
aks- nodepooll- 33641294-0  Ready     agent     19h       v1.12.7
aks- nodepooll- 33641294-1  Ready     agent     19h       v1.12.7
aks- nodepooll- 33641294-2  Ready     agent     19h       v1.12.7

```

* Add more replicas to your deployment like (e.g. 200) in your deployment manifest file, Auto scaler will add a node in the cluster based on that request coming from the unschedulable pods.
 
` kubectl -n kube-system scale --replicas=200 deployment cluster-autoscaler`

* pending pods will schedule to new nodes, this will be managed by a kubernetes scheduler.
As can be seen from below output a new node is added

```
 kubectl get nodes
NAME                        STATUS    ROLES     AGE       VERSION
aks- nodepooll- 33641294-0  Ready     agent     19h       v1.12.7
aks- nodepooll- 33641294-1  Ready     agent     19h       v1.12.7
aks- nodepooll- 33641294-2  Ready     agent     19h       v1.12.7
aks- nodepooll- 33641294-3  Ready     agent     10m       v1.12.7

```
* follow the autoscaler pod streaming logs, by using the following command.if you want to save logs in your machine as a file add `> cas.log` end of the command.

  `kubectl -n kube-system logs -f <autoscaler-pod>` > cas.log

```
 1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 07:51:29.373609       1 scale_up.go:59] Pod tl-int/tlui-6b495c88bb-6n64d is unschedulable
I0508 07:51:29.373801       1 scale_up.go:59] Pod tl-int/tlui-6b495c88bb-hq5ts is unschedulable
I0508 07:51:29.376782       1 scale_up.go:59] Pod tl-int/tlui-6b495c88bb-stq9x is unschedulable
I0508 07:51:29.379030       1 scale_up.go:59] Pod tl-int/tlui-6b495c88bb-25f46 is unschedulable
I0508 07:51:29.379184       1 scale_up.go:59] Pod tl-int/tlui-6b495c88bb-td9k5 is unschedulable
I0508 07:51:29.379291       1 scale_up.go:59] Pod tl-int/tlui-6b495c88bb-r8kpw is unschedulable
I0508 07:51:29.379383       1 scale_up.go:59] Pod tl-int/tlui-6b495c88bb-nd5kj is unschedulable
I0508 07:51:29.379414       1 scale_up.go:59] Pod tl-int/tlui-6b495c88bb-jc6hr is unschedulable
I0508 07:51:29.379468       1 scale_up.go:59] Pod tl-int/tlui-6b495c88bb-86928 is unschedulable
I0508 07:51:29.379595       1 scale_up.go:59] Pod tl-int/tlui-6b495c88bb-cflg6 is unschedulable
I0508 07:51:29.379627       1 scale_up.go:59] Pod tl-int/tlui-6b495c88bb-csk2g is unschedulable
I0508 07:51:29.379643       1 scale_up.go:59] Pod tl-int/tlui-6b495c88bb-sh7jt is unschedulable
I0508 07:51:29.379682       1 scale_up.go:59] Pod tl-int/tlui-6b495c88bb-tv7jm is unschedulable
I0508 07:51:29.839722       1 scale_up.go:199] Best option to resize: nodepool1
I0508 07:51:29.842411       1 scale_up.go:203] Estimated 1 nodes needed in nodepool1
I0508 07:51:29.988130       1 scale_up.go:292] Final scale-up plan: [{nodepool1 3->4 (max: 5)}]
I0508 07:51:29.988176       1 scale_up.go:344] Scale-up: setting group nodepool1 size to 4
I0508 07:51:30.216118       1 azure_container_service_pool.go:206] Set size request: 4
I0508 07:51:30.301969       1 azure_container_service_pool.go:241] Current size: 3, Target size requested: 4
I0508 07:54:09.484267       1 azure_container_service_pool.go:276] Target size set done, AKS. Value: {Response:{Response:0xc421703050} ID:0xc422794db0 Name:0xc422794de0 Type:0xc422794e00 Location:0xc422794e20 Tags:0xc4218b0918 ManagedClusterProperties:0xc42296c300}
I0508 07:54:09.484320       1 azure_container_service_pool.go:299] Got Updated value. Time taken: 2m39.182059589s
I0508 07:54:19.718531       1 azure_manager.go:261] Refreshed ASG list, next refresh after 2019-05-08 07:55:19.718500245 +0000 UTC


```

### step 3: Scale down the node 

* If the pod is part of a daemonset, the pod is safe to turn down, since daemonsets are supposed to run statelessly on all nodes. Removing the node should not reschedule a pod in a daemonset.
* If the pod is a mirror pod, (only relevant if you’ve created static pods), it is considered safe to turn down.
Removing the pod does not bring the number of replicas below the specified minimum replica count, unless you have specified a pod disruption budget and have remaining disruptions to “spend” on moving the pod.

`kubectl -n kube-system scale --replicas=1 deployment cluster-autoscaler`

* The pod doesn’t use any local storage on the node; since the node is going away, that local storage will be lost.
Kube system pods won’t get moved unless they specify a pod disruption budget.
* If any of these node or pod-level checks do not pass, then the node will not be turned down.

[Referenced from](https://medium.com/kubecost/understanding-kubernetes-cluster-autoscaling-675099a1db92)



### Get the logs:

* Follow the autoscaler pod streaming logs, by using the following command.if you want to save logs in your machine as a file add > cas.log end of the command.

  `kubectl -n kube-system logs -f <autoscaler-pod> > cas.log`

```

1 static_autoscaler.go:280] No unschedulable pods
I0508 08:00:46.737863       1 scale_down.go:257] Finding additional 2 candidates for scale down.
I0508 08:00:46.738624       1 cluster.go:80] Fast evaluation: aks-nodepool1-33641294-2 for removal
I0508 08:00:46.738902       1 cluster.go:94] Fast evaluation: node aks-nodepool1-33641294-2 cannot be removed: non-daemonset, non-mirrored, non-pdb-assigned kube-system pod present: tiller-deploy-895d57dd9-5b9hg
I0508 08:00:46.739023       1 cluster.go:80] Fast evaluation: aks-nodepool1-33641294-0 for removal
I0508 08:00:46.739184       1 cluster.go:94] Fast evaluation: node aks-nodepool1-33641294-0 cannot be removed: non-daemonset, non-mirrored, non-pdb-assigned kube-system pod present: cluster-autoscaler-54cf56c958-w7bp5
I0508 08:00:46.739284       1 scale_down.go:294] 2 nodes found unremovable in simulation, will re-check them at 2019-05-08 08:05:46.189886747 +0000 UTC
I0508 08:00:57.017977       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:00:57.018403       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:00:57.267340       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:01:07.813807       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:01:07.817500       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:01:08.017264       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:01:18.567151       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:01:18.567364       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:01:18.706885       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:01:29.082866       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:01:29.085072       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:01:29.191832       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:01:39.303327       1 azure_manager.go:261] Refreshed ASG list, next refresh after 2019-05-08 08:02:39.303291668 +0000 UTC
I0508 08:01:39.449431       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:01:39.449662       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:01:39.547270       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:01:39.713840       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 2m17.383457925s
I0508 08:01:39.714472       1 scale_down.go:446] No candidates for scale down
I0508 08:01:49.867322       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:01:49.868864       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:01:49.989181       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:01:50.217652       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 2m27.80851311s
I0508 08:01:50.217679       1 scale_down.go:446] No candidates for scale down
I0508 08:02:00.417622       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:02:00.418494       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:02:00.548880       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:02:00.792969       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 2m38.346764692s
I0508 08:02:00.793248       1 scale_down.go:446] No candidates for scale down
I0508 08:02:10.967384       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:02:10.968402       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:02:11.062959       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:02:11.282642       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 2m48.896120673s
I0508 08:02:11.282680       1 scale_down.go:446] No candidates for scale down
I0508 08:02:21.457998       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:02:21.458216       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:02:21.589673       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:02:21.845442       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 2m59.405878616s
I0508 08:02:21.845903       1 scale_down.go:446] No candidates for scale down
I0508 08:02:32.019426       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:02:32.023301       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:02:32.201332       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:02:32.466896       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 3m9.947106638s
I0508 08:02:32.467027       1 scale_down.go:446] No candidates for scale down
I0508 08:02:42.482657       1 azure_manager.go:261] Refreshed ASG list, next refresh after 2019-05-08 08:03:42.482645501 +0000 UTC
I0508 08:02:42.610813       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:02:42.613832       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:02:42.706179       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:02:42.902731       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 3m20.562825758s
I0508 08:02:42.903292       1 scale_down.go:446] No candidates for scale down
I0508 08:02:53.063893       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:02:53.064317       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:02:53.195058       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:02:53.496959       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 3m31.009825778s
I0508 08:02:53.496989       1 scale_down.go:446] No candidates for scale down
I0508 08:03:03.727534       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:03:03.728423       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:03:03.837557       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:03:04.099184       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 3m41.619390795s
I0508 08:03:04.099221       1 scale_down.go:446] No candidates for scale down
I0508 08:03:14.367565       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:03:14.368906       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:03:14.488558       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:03:14.767016       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 3m52.19656128s
I0508 08:03:14.767046       1 scale_down.go:446] No candidates for scale down
I0508 08:03:25.022272       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:03:25.022850       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:03:25.116381       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:03:25.366881       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 4m2.869699674s
I0508 08:03:25.366991       1 scale_down.go:446] No candidates for scale down
I0508 08:03:35.625160       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:03:35.626191       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:03:35.767613       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:03:35.973862       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 4m13.461625951s
I0508 08:03:35.974702       1 scale_down.go:446] No candidates for scale down
I0508 08:03:45.992626       1 azure_manager.go:261] Refreshed ASG list, next refresh after 2019-05-08 08:04:45.992615203 +0000 UTC
I0508 08:03:46.149803       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:03:46.150148       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:03:46.252541       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:03:46.452790       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 4m24.072795861s
I0508 08:03:46.452815       1 scale_down.go:446] No candidates for scale down
I0508 08:03:56.718847       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:03:56.719757       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:03:56.823629       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:03:57.088630       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 4m34.618598718s
I0508 08:03:57.092569       1 scale_down.go:446] No candidates for scale down
I0508 08:04:07.404436       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:04:07.405237       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:04:07.539472       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:04:07.766891       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 4m45.282774499s
I0508 08:04:07.766921       1 scale_down.go:446] No candidates for scale down
I0508 08:04:17.935745       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:04:17.936806       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:04:18.041454       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:04:18.272691       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 4m55.875365731s
I0508 08:04:18.272955       1 scale_down.go:446] No candidates for scale down
I0508 08:04:28.569546       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:04:28.570458       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:04:28.704265       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:04:28.966940       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 5m6.471751906s
I0508 08:04:28.966972       1 scale_down.go:446] No candidates for scale down
I0508 08:04:39.212144       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:04:39.213203       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:04:39.344816       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:04:39.554529       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 5m17.142258617s
I0508 08:04:39.555009       1 scale_down.go:446] No candidates for scale down
I0508 08:04:49.568168       1 azure_manager.go:261] Refreshed ASG list, next refresh after 2019-05-08 08:05:49.56815509 +0000 UTC
I0508 08:04:49.726500       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:04:49.727083       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:04:49.849603       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:04:50.118505       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 5m27.648330147s
I0508 08:04:50.118728       1 scale_down.go:446] No candidates for scale down
I0508 08:05:00.348601       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:05:00.350050       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:05:00.460061       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:05:00.682831       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 5m38.214482489s
I0508 08:05:00.683461       1 scale_down.go:446] No candidates for scale down
I0508 08:05:11.013527       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:05:11.020732       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:05:11.107128       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:05:11.315369       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 5m48.783883044s
I0508 08:05:11.315412       1 scale_down.go:446] No candidates for scale down
I0508 08:05:21.471062       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:05:21.471442       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:05:21.565954       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:05:21.779971       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 5m59.412477999s
I0508 08:05:21.780000       1 scale_down.go:446] No candidates for scale down
I0508 08:05:31.967701       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:05:31.968195       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:05:32.133317       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:05:32.349249       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 6m9.8728855s
I0508 08:05:32.349300       1 scale_down.go:446] No candidates for scale down
I0508 08:05:42.567579       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:05:42.568608       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:05:42.669913       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:05:42.966860       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 6m20.462240359s
I0508 08:05:42.967695       1 scale_down.go:446] No candidates for scale down
I0508 08:05:52.989940       1 azure_manager.go:261] Refreshed ASG list, next refresh after 2019-05-08 08:06:52.989906356 +0000 UTC
I0508 08:05:53.171311       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:05:53.171779       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:05:53.384215       1 scale_down.go:257] Finding additional 2 candidates for scale down.
I0508 08:05:53.384369       1 cluster.go:80] Fast evaluation: aks-nodepool1-33641294-2 for removal
I0508 08:05:53.384611       1 cluster.go:94] Fast evaluation: node aks-nodepool1-33641294-2 cannot be removed: non-daemonset, non-mirrored, non-pdb-assigned kube-system pod present: tiller-deploy-895d57dd9-5b9hg
I0508 08:05:53.384671       1 cluster.go:80] Fast evaluation: aks-nodepool1-33641294-0 for removal
I0508 08:05:53.384850       1 cluster.go:94] Fast evaluation: node aks-nodepool1-33641294-0 cannot be removed: non-daemonset, non-mirrored, non-pdb-assigned kube-system pod present: cluster-autoscaler-54cf56c958-w7bp5
I0508 08:05:53.384926       1 scale_down.go:294] 2 nodes found unremovable in simulation, will re-check them at 2019-05-08 08:10:52.989878155 +0000 UTC
I0508 08:05:53.545125       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 6m31.070078713s
I0508 08:05:53.545724       1 scale_down.go:446] No candidates for scale down
I0508 08:06:03.720248       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:06:03.721172       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:06:03.814002       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:06:04.036610       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 6m41.649029309s
I0508 08:06:04.036748       1 scale_down.go:446] No candidates for scale down
I0508 08:06:14.267426       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:06:14.268490       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:06:14.384840       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:06:14.600783       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 6m52.142368012s
I0508 08:06:14.600808       1 scale_down.go:446] No candidates for scale down
I0508 08:06:24.758468       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:06:24.758749       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:06:24.861974       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:06:25.095281       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 7m2.703662395s
I0508 08:06:25.095309       1 scale_down.go:446] No candidates for scale down
I0508 08:06:35.267339       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:06:35.269121       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:06:35.389477       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:06:35.592488       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 7m13.194110719s
I0508 08:06:35.592598       1 scale_down.go:446] No candidates for scale down
I0508 08:06:45.847787       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:06:45.851304       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:06:45.952342       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:06:46.157005       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 7m23.696575923s
I0508 08:06:46.159151       1 scale_down.go:446] No candidates for scale down
I0508 08:06:56.196327       1 azure_manager.go:261] Refreshed ASG list, next refresh after 2019-05-08 08:07:56.196315798 +0000 UTC
I0508 08:06:56.348719       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:06:56.349078       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:06:56.433404       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:06:56.633345       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 7m34.276496455s
I0508 08:06:56.633374       1 scale_down.go:446] No candidates for scale down
I0508 08:07:06.867308       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:07:06.867583       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:07:06.974349       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:07:07.169585       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 7m44.775496062s
I0508 08:07:07.169845       1 scale_down.go:446] No candidates for scale down
I0508 08:07:17.411197       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:07:17.412257       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:07:17.493555       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:07:17.720242       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 7m55.287135914s
I0508 08:07:17.720269       1 scale_down.go:446] No candidates for scale down
I0508 08:07:27.857667       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:07:27.858758       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:07:27.956000       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:07:28.182973       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 8m5.814030608s
I0508 08:07:28.183004       1 scale_down.go:446] No candidates for scale down
I0508 08:07:38.400781       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:07:38.401234       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:07:38.526320       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:07:38.719159       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 8m16.303319648s
I0508 08:07:38.719192       1 scale_down.go:446] No candidates for scale down
I0508 08:07:48.967427       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:07:48.967882       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:07:49.074743       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:07:49.366984       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 8m26.823711472s
I0508 08:07:49.367015       1 scale_down.go:446] No candidates for scale down
I0508 08:07:59.397319       1 azure_manager.go:261] Refreshed ASG list, next refresh after 2019-05-08 08:08:59.39730812 +0000 UTC
I0508 08:07:59.798865       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:07:59.803411       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:07:59.964202       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:08:00.309126       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 8m37.477474677s
I0508 08:08:00.309168       1 scale_down.go:446] No candidates for scale down
I0508 08:08:10.552541       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:08:10.552906       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:08:10.743942       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:08:11.165952       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 8m48.417936073s
I0508 08:08:11.166143       1 scale_down.go:446] No candidates for scale down
I0508 08:08:21.453962       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:08:21.454257       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:08:21.751263       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:08:22.241577       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 8m59.269539089s
I0508 08:08:22.241618       1 scale_down.go:446] No candidates for scale down
I0508 08:08:32.614138       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:08:32.615193       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:08:32.914526       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:08:33.614834       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 9m10.352165882s
I0508 08:08:33.615111       1 scale_down.go:446] No candidates for scale down
I0508 08:08:44.022848       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:08:44.023171       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:08:44.285715       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:08:44.975568       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 9m21.712198902s
I0508 08:08:44.976314       1 scale_down.go:446] No candidates for scale down
I0508 08:08:55.223622       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:08:55.224447       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:08:55.438989       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:08:55.870289       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 9m33.091331289s
I0508 08:08:55.870312       1 scale_down.go:446] No candidates for scale down
I0508 08:09:05.914804       1 azure_manager.go:261] Refreshed ASG list, next refresh after 2019-05-08 08:10:05.914788083 +0000 UTC
I0508 08:09:06.226728       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:09:06.227670       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:09:06.315602       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:09:06.566767       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 9m43.99496054s
I0508 08:09:06.566811       1 scale_down.go:446] No candidates for scale down
I0508 08:09:16.906486       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:09:16.906894       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:09:17.079928       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:09:17.442731       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 9m54.746647319s
I0508 08:09:17.442764       1 scale_down.go:446] No candidates for scale down
I0508 08:09:27.722914       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:09:27.727164       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:09:27.903046       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:09:28.253140       1 scale_down.go:387] aks-nodepool1-33641294-3 was unneeded for 10m5.538079033s
I0508 08:09:28.415763       1 scale_down.go:594] Scale-down: removing empty node aks-nodepool1-33641294-3
I0508 08:09:28.492595       1 delete.go:53] Successfully added toBeDeletedTaint on node aks-nodepool1-33641294-3
I0508 08:09:28.496485       1 azure_container_service_pool.go:360] Node: azure:///subscriptions/7aa98dd2-d24a-476c-8cca-c2febfd47d51/resourceGroups/MC_AKS19ClusterRG_AKS19Cluster_eastus/providers/Microsoft.Compute/virtualMachines/aks-nodepool1-33641294-3
I0508 08:09:28.496738       1 azure_container_service_pool.go:364] ProviderID before calling acsmgr: azure:///subscriptions/7aa98dd2-d24a-476c-8cca-c2febfd47d51/resourceGroups/MC_AKS19ClusterRG_AKS19Cluster_eastus/providers/Microsoft.Compute/virtualMachines/aks-nodepool1-33641294-3
I0508 08:09:28.653818       1 azure_container_service_pool.go:333] ProviderID got to delete: azure:///subscriptions/7aa98dd2-d24a-476c-8cca-c2febfd47d51/resourceGroups/MC_AKS19ClusterRG_AKS19Cluster_eastus/providers/Microsoft.Compute/virtualMachines/aks-nodepool1-33641294-3
I0508 08:09:28.704819       1 azure_container_service_pool.go:338] VM name got to delete: aks-nodepool1-33641294-3
I0508 08:09:28.755209       1 azure_util.go:144] found nic name for VM (MC_AKS19ClusterRG_AKS19Cluster_eastus/aks-nodepool1-33641294-3): aks-nodepool1-33641294-nic-3
I0508 08:09:28.755232       1 azure_util.go:147] deleting VM: MC_AKS19ClusterRG_AKS19Cluster_eastus/aks-nodepool1-33641294-3
I0508 08:09:28.755239       1 azure_util.go:151] waiting for VirtualMachine deletion: MC_AKS19ClusterRG_AKS19Cluster_eastus/aks-nodepool1-33641294-3
I0508 08:11:14.839773       1 azure_util.go:157] VirtualMachine MC_AKS19ClusterRG_AKS19Cluster_eastus/aks-nodepool1-33641294-3 removed
I0508 08:11:14.839803       1 azure_util.go:160] deleting nic: MC_AKS19ClusterRG_AKS19Cluster_eastus/aks-nodepool1-33641294-nic-3
I0508 08:11:14.848795       1 azure_util.go:162] waiting for nic deletion: MC_AKS19ClusterRG_AKS19Cluster_eastus/aks-nodepool1-33641294-nic-3
I0508 08:11:35.411503       1 azure_util.go:192] deleting managed disk: MC_AKS19ClusterRG_AKS19Cluster_eastus/aks-nodepool1-33641294-3_OsDisk_1_0cfd53b492b14e669ab874c58361c42d

```
```
1 static_autoscaler.go:280] No unschedulable pods
I0508 08:17:01.113356       1 scale_down.go:257] Finding additional 2 candidates for scale down.
I0508 08:17:01.113779       1 cluster.go:80] Fast evaluation: aks-nodepool1-33641294-0 for removal
I0508 08:17:01.114346       1 cluster.go:94] Fast evaluation: node aks-nodepool1-33641294-0 cannot be removed: non-daemonset, non-mirrored, non-pdb-assigned kube-system pod present: cluster-autoscaler-54cf56c958-w7bp5
I0508 08:17:01.114538       1 cluster.go:80] Fast evaluation: aks-nodepool1-33641294-2 for removal
I0508 08:17:01.114717       1 cluster.go:94] Fast evaluation: node aks-nodepool1-33641294-2 cannot be removed: non-daemonset, non-mirrored, non-pdb-assigned kube-system pod present: tiller-deploy-895d57dd9-5b9hg
I0508 08:17:01.114877       1 scale_down.go:294] 2 nodes found unremovable in simulation, will re-check them at 2019-05-08 08:22:00.800121477 +0000 UTC
I0508 08:17:01.280958       1 scale_down.go:446] No candidates for scale down
I0508 08:17:11.308027       1 azure_manager.go:261] Refreshed ASG list, next refresh after 2019-05-08 08:18:11.308015822 +0000 UTC
I0508 08:17:11.467097       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0508 08:17:11.467287       1 static_autoscaler.go:280] No unschedulable pods
I0508 08:17:11.579929       1 scale_down.go:175] Scale-down calculation: ignoring 2 nodes, that were unremovable in the last 5m0s
I0508 08:17:11.674988       1 scale_down.go:446] No candidates for scale down
I0508 08:17:21.867079       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop

```

## Get the nodes 

* Verify it by using the following command, to check the status of the node, which is down. It will shows `NotReady', after sometime it will delete automatically.

 `kubectl get nodes`
  
```  
NAME                        STATUS    ROLES     AGE       VERSION
aks- nodepooll- 33641294-0  Ready     agent     19h       v1.12.7
aks- nodepooll- 33641294-1  Ready     agent     19h       v1.12.7
aks- nodepooll- 33641294-2  Ready     agent     19h       v1.12.7
aks- nodepooll- 33641294-3  NotReady  agent     30m       v1.12.7

```

taken from Kubernetes autoscaler github project


    

### practice 4:  Qudos


### What is Qudos ?

* Qloudable Developer operating name spaces `QuDoS` solves the problem of automating recurring tasks for developers, before code goes into integration environment.

### What Qudos can automate?

* Changing environment variables to point to required dependencies like db or other services.
* Pushing local changes to remote service.
* Forwarding cluster services to appear to be local services.
* Deploying changes after a save to the environment in cluster.
* Pulling other peoples code so conflicts are resolved early.
* Committing changes on each save to local git repo.
* But connecting to the debugged environment.
* Running some node processes.
* Restarting node process, restarting debugger, checking the old break points and variables.
    
### What Qudos cannot automate?

* Creating a local workspace, by cloning and checking out a branch from GitHub may be manual.
* Pushing changes to GitHub has to be manual.
* Debugging has to be manual.    
    
## step 1: Developer scripts to automate deploy and debug connect to pods in cluster.
 
 ### Pre-Requisites
 
   *  Gitbash for windows [Installation url](https://git-scm.com/download/win).
   *  Github Repo and Credentials.
   *  Qudos works on Kuberentes cluster for various clouds such as Azure, OKE, AWS(NodeJS Application).
   *  If you works on corresponding cloud(AKS,OKE,AWS) use following steps.
      *AKS* Requires `Azure CLI`[Installation url](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-windows?view=azure-cli-latest).
      *OKE* Requires `OCI CLI`[Installation url](https://docs.cloud.oracle.com/iaas/Content/API/SDKDocs/cliinstall.htm).
   *  Kubectl.exe download from these url [Installation url](https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/windows/amd64/kubectl.exe) needs to communicate with your Kubernetes API Server.
   * Create a `mkdir -p $HOME/.kube` .kube directory to configure to your cluster.
   * After everything is installed check with the following command. This will listout all namespaces in a cluster.
   
      `kubectl get ns`
   
### Installation Instructions

   * Clone the Repository(check out the right branch) which have the application code(NodeJs) by using a `Git Bash for windows` following command.
   
     `git clone ($url)`.
      
   * In this example it is '/c/Users/Documents/tl-pint/testdir/', in this folder the .git folder and .gitignore files should exist,          check using.
    `ls -ltra .git .gitignore`. 
   * NOTE: in windows the single quotes are important for paths with spaces in their names.
    `cd '/c/Users/Documents/tl-pint/testdir/'`      
   * After you copies of application code into your local workspace, clone the Qudos scripts from the Github, Then copy Quods scripts         into your local workspace where have a Nodejs Application code.
    `git clone $(url)`  
    `cp $(/quodsscripts to application code(nodejs) working directory)`.
      
   * Check for your files using ls -ltr *.sh, listout the scripts clonned from the qudos scripts.
   ![alt text](https://github.com/sysgain/qloudable-tl-labs/blob/manuals/images/qudos.JPG)
   * Create a .vscode directory for debugging with visual studio code.
   
     `mkdir -p .vscode`
     `ls -ltr .vscode`
    
### Usage Instructions

setup.sh--> Run this script in your workspace to ignore the qudos dev scripts into your github.
 
```
#!/bin/bash
# run this script once first to setup your work space
if  ls -ltra .gitignore 
then
echo autodeploy.sh >>.gitignore
echo autoconnect.sh >>.gitignore
echo autodebug.sh >>.gitignore
echo '*.log' >>.gitignore
echo gitstatus.txt >>.gitignore
echo setup.sh >>.gitignore
echo setupenv.sh >>.gitignore
echo installqudos.sh >>.gitignore
git commit ./.gitignore -m "excluding qudos dev scripts"
else
echo Are you sure you are in the root directory of your local git repo where your .git and .gitignore are?
echo
echo if not copy this file and other scripts to your local git repos root directory
fi

```


* Edit `setupenv.sh` to give your namespace, service directories, branch you will be changing and the service you wish to debug.
    
```
#!/bin/bash
#set up these variables before using the autocommit/connect/debug scripts

export namespace=labs-sidda

# all the services that will be changed, seperated by space, 
#   names have to be exactly same as directory names in git hub
#     choices for these services are given below

export svcs="billing-service transfer-service"

#the service that is going to be debugged
export debugservice=billing-service

# agreement-service  analytics-service  appstream-client  auth0-client  
# badge-service  billing-service  channel-service  cleanup-monitor-service  
# cleanup-resource-service  cleanup-service  deploy-container-service  eula-service  
# labs-service  metadata-service  notification-service  permission-service  plan-service  
# project-service  provider-router-service  provider-setup-service  provision-service  
# publisher-bio-service  reset-password-service  reviews-rating-service  signup-service   
# storage-service   stripe-service  
# template-service  terminal-creation-service  terminal-service  training-deployment-service  
# training-resource-service  training-scheduler-service  transfer-service  user-managment  
# user-training-service  view-activity-service

#set branch name you are working on here

export branch=tl-oci-pint

```
* Make sure that working directory needs to be choose right one which have the application code and Qudos dev scripts.
    * Run autodeploy.sh as ./autodeploy.sh.

#### autodeploy.sh

    * when user saves a file, and hits enter in the gitbash terminal, the file is copies to the cluster, the node process is restarted in the container, the file is committed to local repo ( on pressing enter ), files are pulled from the branch in github origin repo ( on pressing enter ) writes container logs to service-name.log and git conflicts and logs to git.log user needs to push to github origin to make changes permanent.( beyond container life time ).


```
#!/bin/bash

#set up your env variables in setupenv.sh
source ./setupenv.sh

           index=0
           for svc in $( echo $svcs )
           do
           echo $svc is being changed
           services["$index"]=$svc

           ((index++))
           done

echo
echo getting pod names for each service
echo
index=0
 service_count=${#services[@]}
while [ "$index" -lt "$service_count" ]
do  
  service=${services[$index]} 
 pods["$index"]="$(kubectl -n $namespace get po | grep ^"$service" | grep Running | awk '{print $1}')";
 podname=${pods[$index]} 
 touch ./$service.log
 kubectl -n $namespace logs -f  $podname >> ./$service.log &
  echo ${services[$index]} ${pods[$index]}
  ((index++))
done

####################main loop#############
touch git.log
while true
do
    git status -s >gitstatus.txt
    
if [ `cat gitstatus.txt | wc -l` -gt 0 ]
        then 
        echo
            echo there are changes to commit and deploy
         echo
    index=0
    service_count=${#services[@]}
    while [ "$index" -lt "$service_count" ]
    do    
  #  echo ${services[$index]}
    service=${services[$index]}
    podname=${pods[$index]}

    if [ `cat gitstatus.txt | grep "$service" | wc -l` -gt 0 ]
    then
    echo '############################################check for package.json changes################'
        if [ `cat gitstatus.txt | grep package.json$ | wc -l` -gt 0 ]
    then
    echo copying package.json
    kubectl -n $namespace cp "$service"/package.json "$podname":/usr/local/src/
    
    echo installing npm modules
    kubectl -n $namespace exec  $podname -- bash -c "cd /usr/local/src/ && npm install"

    fi
    
    echo copying.....
        for file in $(cat gitstatus.txt | awk '{print $2}' | grep "$service")
        do
         echo copying..... $file
        filepath=$(echo $file | sed "s#$service##")
        parentdir=$(dirname $filepath)
        kubectl -n $namespace cp "./$file" $podname:/usr/src/services/"$parentdir"
 #       kubectl -n $namespace exec  $podname -- ls -ltr /usr/src/services/"$parentdir"
        done    
   
    echo restarting node............
   # pid=$(kubectl -n $namespace exec  $podname -- ps -ef | grep node | awk '{print $2}')
    echo stopping node process $pid
    

      echo starting new node process
   #  kubectl -n $namespace exec $podname -- node app.js >> ./"$service".log &
 #         echo checking node process
#      kubectl -n $namespace exec  $podname -- ps -ef | grep node | grep -v grep
    #echo ${services[$index]} ${pods[$index]} ${ports[$index]} ${pfpids[$index]}
    
    fi
    ((index++))
    done  

#    sleep $sleeptime
echo   
    # read -p "Press enter to commit changes or CTL+C multiple times to exit"  
     #echo 

    echo committing changes to local repo
    git add * >>git.log 2>&1
 #   git commit -a -m "autocommit" >>git.log
 echo enter comments, or CTRL+C to exit
echo when done enter . as first character in an empty line and press enter
comment=''
while read line; do
   if [ "$line" == '.' ] 
    then
     break
      fi
comment="$comment $line"
done < /dev/stdin

echo  $comment
 git commit -a -m "$comment" >>git.log
    echo committed code 
    #  echo
#      read -p "Press enter to pull changes from origin repository or CTL+C multiple times to exit"   
     # echo
else
#echo
 #          echo there are no changes to deploy  
#            git status
#            git fetch origin $branch >>git.log 2>&1
 #           echo
           echo files to be pushed to $branch  >>git.log  2>&1
            git --no-pager diff  --name-only origin/$branch  >>git.log  2>&1
#            echo
#           echo rebasing from $branch from origin repo
  #          git pull --rebase origin $branch >>git.log  2>&1
            git status --untracked-files=no >>git.log
#            sleep $sleeptime
#echo
#             read -p "Press enter to check for local changes or CTL+C multiple times to exit"   
#echo
fi

done

```

* Make sure that whatever you specified in `setupenv.sh` to change the services, some addition to app.js,adding helloworld comment. 
![alt text](https://github.com/sysgain/qloudable-tl-labs/blob/manuals/images/qudos-appjs.JPG).


```
$ ./autodeploy.sh
provision-service is being changed

getting pod names for each service

provision-service provision-service-5c85464489-njjn2

there are changes to commit and deploy

############################################check for package.json changes################
copying.....
copying..... provision-service/app.js
restarting node............
stopping node process
starting new node process

committing changes to local repo
enter comments, or CTRL+C to exit
when done enter . as first character in an empty line and press enter
updated app.js with comment helloworld
.
updated app.js with comment helloworld
warning: LF will be replaced by CRLF in git.log.
The file will have its original line endings in your working directory.
warning: LF will be replaced by CRLF in provision-service.log.
The file will have its original line endings in your working directory.
warning: LF will be replaced by CRLF in git.log.
The file will have its original line endings in your working directory.
warning: LF will be replaced by CRLF in gitstatus.txt.
The file will have its original line endings in your working directory.
warning: LF will be replaced by CRLF in provision-service.log.
The file will have its original line endings in your working directory.
warning: LF will be replaced by CRLF in git.log.
The file will have its original line endings in your working directory.
warning: LF will be replaced by CRLF in gitstatus.txt.
The file will have its original line endings in your working directory.
warning: LF will be replaced by CRLF in provision-service.log.
The file will have its original line endings in your working directory.
committed code

there are changes to commit and deploy


committing changes to local repo
enter comments, or CTRL+C to exit
when done enter . as first character in an empty line and press enter

```
* After you run the `autodeploy.sh`, with servicename the log file will be generated.
* verify it now by checking in a kubernetes cluster, inside container with interactive.
```
PS C:\Users\sysgain> kubectl -n tl-navin exec -it provision-service-5c85464489-njjn2 bash
root@provision-service-5c85464489-njjn2:/usr/src/services# ls
DEE_entry_point.sh  README.md  app.js  docker-compose.yml     node_modules            package-lock.json  provision-service.log  test
Dockerfile          api        config  env_secrets_expand.sh  npm-debug.log.71321291  package.json       stack.yml
root@provision-service-5c85464489-njjn2:/usr/src/services# ls -ltr
total 196
drwxr-xr-x 8 root root     96 Apr 17 11:29 api
-rw-r--r-- 1 root root    204 Apr 17 11:29 stack.yml
-rw-r--r-- 1 root root      0 Apr 17 11:29 npm-debug.log.71321291
-rw-r--r-- 1 root root    462 Apr 17 11:29 docker-compose.yml
drwxr-xr-x 3 root root     17 Apr 17 11:29 test
-rw-r--r-- 1 root root   1971 Apr 17 11:29 README.md
-rw-r--r-- 1 root root    600 Apr 17 11:29 Dockerfile
-rw-r--r-- 1 root root     97 Apr 17 11:29 DEE_entry_point.sh
-rw-r--r-- 1 root root   1133 Apr 17 11:29 package.json
-rw-r--r-- 1 root root 157965 Apr 17 11:29 package-lock.json
-rw-r--r-- 1 root root   1223 Apr 17 11:29 env_secrets_expand.sh
drwxr-xr-x 2 root root     75 Apr 17 11:29 config
lrwxrwxrwx 1 root root     27 Apr 17 12:04 node_modules -> /usr/local/src/node_modules
-rw-rw-rw- 1 root root   2402 May  6 09:12 provision-service.log
-rw-rw-rw- 1 root root   4240 May  6 09:17 app.js
root@provision-service-5c85464489-njjn2:/usr/src/services# cat app.js
'use strict';
const dotenv = require('dotenv');
dotenv.load();
//LIBS helloworld

```

#### Autoconnect

* autoconnect.sh
    * When user starts this every cluster service (as mentioned in setupenv.sh) appears as it is a local service and can be accessed by curl command, browser and postman.(provision-service.log) 
```
#!/bin/bash
#set these parameters
#set up your env variables in setupenv.sh
source ./setupenv.sh

           index=0
           for svc in $( echo $svcs )
           do
           echo $svc xys
           services["$index"]=$svc

           ((index++))
           done

index=0
 service_count=${#services[@]}
while [ "$index" -lt "$service_count" ]
do  
  service=${services[$index]} 
 portforwards["$index"]="$( kubectl -n $namespace get cm $service -o yaml | grep -w PORT: | awk '{print $2'}  | sed s'/"//g' )";
 echo ${portforwards[$index]}
  ((index++))
done


#portforwards=( 7071 7072 )
pfpids=( 111111 222222 )


####################main loop#############

while true
do
    index=0
    service_count=${#services[@]}
while [ "$index" -lt "$service_count" ]
    do    
    echo ${services[$index]}
    service=${services[$index]}
    pfpid=${pfpids[$index]}
    portforward=${portforwards[$index]}

    if [ ` ps -ef | awk '{print $2}'| grep -w "$pfpid" ` ]
           then
           echo portforward is active for $service at port $portforward
     else
           podname="$(kubectl -n $namespace get po | grep ^"$service" | awk '{print $1}')";
           echo portforward is  not connected for $service at port $portforward
            kubectl -n $namespace port-forward $podname $portforward &
            pfpids["$index"]=$(echo $!)
    fi  

   ((index++))
    done   
#          sleep $sleeptime    
             read -p "Press enter to check if portforwarding is still active or CTL+C multiple times to exit"   
done



```

```
$ ./autoconnect.sh
provision-service xys
8089
provision-service
portforward is not connected for provision-service at port 8089
Press enter to check if portforwarding is still active or CTL+C multiple times to exit
provision-service
portforward is active for provision-service at port 8089
Press enter to check if portforwarding is still active or CTL+C multiple times to exitForwarding from 127.0.0.1:8089 -> 8089
Forwarding from [::1]:8089 -> 8089
Handling connection for 8089
Handling connection for 8089
```
* Run curl command generated in .log file with service name, will give swagger.

`curl http://127.0.0.1:8089/v2/components/provision-service/swagger`

```

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0{
  "swagger": "2.0",
  "info": {
    "description": "connection services API",
    "version": "1.0",
    "title": "TL provision services API"
  },
  "basePath": "/v2/components/provision-service",
  "schemes": [
    "http"
  ],
  "securityDefinitions": {
    "UserSecurity": {
      "type": "apiKey",
      "in": "header",
      "name": "Authorization"
    }
  },
  "paths": {

```


* In working directory log file will be generated with .log file extension with service name.

```
(provision-service.log)

[33m[nodemon] 1.18.10[39m
[33m[nodemon] to restart at any time, enter `rs`[39m
[33m[nodemon] watching: *.*[39m
[32m[nodemon] starting `node app.js`[39m
Creating a pool connected to rdb.tl-int:28015
Creating a pool connected to rdb.tl-int:28015
Creating a pool connected to rdb.tl-int:28015
Creating a pool connected to rdb.tl-int:28015
Creating a pool connected to rdb.tl-int:28015
try this:
curl http://127.0.0.1:8089/v2/components/provision-service/swagger
Database exists.
Table exists.
Table exists.
Table exists.
Table exists.
index for createdDTS already exists
index for channelId already exists
index for updatedDTS already exists
index for priority already exists
index for channelId already exists
index for channelcodeId already exists
verification started...
{"name":"qloudable-training-labs","hostname":"provision-service-5c85464489-njjn2","pid":20,"level":30,"msg":"verification started...","time":"2019-04-17T13:04:49.002Z","v":0}
verification successful
verification started...
{"name":"qloudable-training-labs","hostname":"provision-service-5c85464489-njjn2","pid":20,"level":30,"msg":"verification started...","time":"2019-04-17T13:04:50.002Z","v":0}
verification successful
[33m[nodemon] 1.18.10[39m
[33m[nodemon] to restart at any time, enter `rs`[39m
[33m[nodemon] watching: *.*[39m
[32m[nodemon] starting `node app.js`[39m
Creating a pool connected to rdb.tl-int:28015
Creating a pool connected to rdb.tl-int:28015
Creating a pool connected to rdb.tl-int:28015
Creating a pool connected to rdb.tl-int:28015
Creating a pool connected to rdb.tl-int:28015
try this:
curl http://127.0.0.1:8089/v2/components/provision-service/swagger
Database exists.
Table exists.
Table exists.
Table exists.
Table exists.
index for createdDTS already exists
index for channelId already exists
index for updatedDTS already exists
index for priority already exists
index for channelId already exists
index for channelcodeId already exists
verification started...
{"name":"qloudable-training-labs","hostname":"provision-service-5c85464489-njjn2","pid":20,"level":30,"msg":"verification started...","time":"2019-04-17T13:04:49.002Z","v":0}
verification successful
verification started...
{"name":"qloudable-training-labs","hostname":"provision-service-5c85464489-njjn2","pid":20,"level":30,"msg":"verification started...","time":"2019-04-17T13:04:50.002Z","v":0}
verification successful

```

#### Autodebug

    * To run ./autodebug.sh modify only the service name in `launch. json` found in this folder '/c/Users/Document/tl-pint/testdir/'/. Vscode.
    * Setup `setupenv.sh` Environment variables for which service you are going to be debug.
    
           `export debugservice=billing-service`
    
```
(launch.json)
{
    "version": "0.2.0",
    "configurations": [

      {
        "type": "node",
        "request": "attach",
        "name": "Attach",
        "port": "9229",
        "localRoot": "${workspaceRoot}/provision-service",
        "remoteRoot": "/usr/src/services/"
      }

        

    ]
  }

```

`autodebug.sh`
```
#!/bin/bash 


#set up your env variables in setupenv.sh
source ./setupenv.sh

           index=0
           for svc in $( echo "$debugservice" )
           do
           echo $svc to be debugged
           services["$index"]=$svc

           ((index++))
           done
portforwards=( 9229 )
pfpids=( 111111 )


####################main loop#############

while true
do
    index=0
    service_count=${#services[@]}
while [ "$index" -lt "$service_count" ]
    do    
    echo ${services[$index]}
    service=${services[$index]}
    pfpid=${pfpids[$index]}
    portforward=${portforwards[$index]}

    if [ ` ps -ef | awk '{print $2}'| grep -w "$pfpid" ` ]
           then
           echo portforward is active for $service at port $portforward
     else
           podname="$(kubectl -n $namespace get po | grep ^"$service" | grep Running | awk '{print $1}')";

              echo restarting node............on pod $podname
    pid=$(kubectl -n $namespace exec  $podname -- ps -ef | grep -vw sh | grep -vw nodemon | grep -w node | awk '{print $2}'  )
    echo stopping node process $pid
    kubectl -n $namespace exec  $podname -- kill -9 $pid 
    
    pid=$(kubectl -n $namespace exec  $podname -- ps -ef | grep -w nodemon | grep -v grep | awk '{print $2}'  )
   echo stopping node process $pid
    kubectl -n $namespace exec  $podname -- kill -9 $pid 

#      echo starting new node process with debug mode
    kubectl -n $namespace exec $podname -- nodemon --inspect-brk app.js &
         echo checking node process
      kubectl -n $namespace exec  $podname -- ps -ef | grep node | grep -v grep
    echo ${services[$index]} ${pods[$index]}
        
#           pods["$index"]="$(kubectl -n $namespace get po | grep ^"$service" | awk '{print $1}')";
            kubectl -n $namespace port-forward $podname $portforward &
               echo the portforward is  not connected for $service at port $portforward
#            pfpid=$(echo $!)
            pfpids["$index"]=$(echo $!)

    fi  

    ((index++))
    done   
#          sleep $sleeptime   
             read -p "Press enter to check if debug session port-fowarding is active or CTL+C multiple times to exit"   
done

```

./autodebug.sh

```

provision-service to be debugged
provision-service
restarting node............on pod provision-service-5c85464489-njjn2
stopping node process 310
stopping node process 287
checking node process
provision-service
the portforward is not connected for provision-service at port 9229
Press enter to check if debug session port-fowarding is active or CTL+C multiple times to exit?[33m[nodemon] 1.18.10?[39m
?[33m[nodemon] to restart at any time, enter `rs`?[39m
?[33m[nodemon] watching: *.*?[39m
?[32m[nodemon] starting `node --inspect-brk app.js`?[39m
Debugger listening on ws://127.0.0.1:9229/3cebc416-faf7-45df-8181-01097e7c0341
For help see https://nodejs.org/en/docs/inspector
Forwarding from 127.0.0.1:9229 -> 9229
Forwarding from [::1]:9229 -> 9229

provision-service
portforward is active for provision-service at port 9229
Press enter to check if debug session port-fowarding is active or CTL+C multiple times to exit
provision-service
portforward is active for provision-service at port 9229
Press enter to check if debug session port-fowarding is active or CTL+C multiple times to exit

```

### step 2: Admin Scripts for Qudos

### How does it work?

* Create one reference environment for all end to end services(including UI).

* Creates an one environment per feature/developer with only services (and callers) that are being changed, the rest pointing to the reference environment.
* Create a DEBUG environment, which can become prod/stg/int/or any commit within a seconds.
* Developers can connect to any services in these environments, debug them, update them, share their services with others, check to see what happens if their service goes into prod/stg/int/other developers namespace…

* These scripts can create, update and delete namespaces(as well as adds a service to the namespace and remove a service from the namespace). Edit `setupenv.sh` to set the environment variables for later usage.

```
(setupenv.sh)
#!/bin/bash
#set up these variables before using the autodeploy/connect/debug scripts

export devname=autoupdate
export gitusername=XXXXXX
export giturl='https://github.com/sysgain/qloudable-training-labs.git'
export gitpassword=XXXXXX
#set branch name you are working on here
export gitbranch=tl-oci-pint
export kubesource='https://github.com/sysgain/kubernetes-stack-files.git'
export kubebranch=tl-oke
export destination='./kubeyaml/'

# all the services that will be changed, seperated by space, 
#   names have to be exactly same as directory names in git hub
#     choices for these services are given below

export services="billing-service transfer-service"


# agreement-service  analytics-service  appstream-client  auth0-client  
# badge-service  billing-service  channel-service  cleanup-monitor-service  
# cleanup-resource-service  cleanup-service  deploy-container-service  eula-service  
# labs-service  metadata-service  notification-service  permission-service  plan-service  
# project-service  provider-router-service  provider-setup-service  provision-service  
# publisher-bio-service  reset-password-service  reviews-rating-service  signup-service   
# storage-service   stripe-service  
# template-service  terminal-creation-service  terminal-service  training-deployment-service  
# training-resource-service  training-scheduler-service  transfer-service  user-managment  
# user-training-service  view-activity-service



```

### Create the namespace

Download to an empty location above your repo and run using a gitbash.

`cd /download/dir ./create-ns.sh` This will creates a namespace.

```
#!/bin/bash -e
echo "$devname"
echo $services
kubectl create ns "$devname" 
kubectl get ns

# Create your own secrets according to requirements, here specified with the sample filenames. 

kubectl -n "$devname" create secret generic stupidsecret --from-file=/home/ubuntu/a.pem 

kubectl -n "$devname" create secret generic machine --from-file=/home/ubuntu/machine.pem 

kubectl -n "$devname" create secret docker-registry shaoketest --docker-server=iad.ocir.io --docker-username="jumpstart/XXXXXX" --docker-password="XXXXXXXXXXXX"

kubectl -n "$devname" get secret




dependency_file=$source/servicedeplendence.txt

rm -rf a.txt b.txt
touch a.txt b.txt

for x  in `echo $services`
do
echo   $x >>a.txt
done

sed -i '/^$/d' a.txt
svc=""
while read -r line; do
svc="$svc $line"
#echo $svc
done < a.txt

echo $svc
#cat b.txt | sort -u >a.txt


destination="$destination"/$devname
mkdir -p $destination
rm -rf "$destination"/*
#######################################copy all files from from source#######################
while read -r svcdir; do
ls -ltr "$source"/"$svcdir"
for file  in `ls "$source"/"$svcdir" | grep -v ext `
do
echo   $file
cp "$source"/"$svcdir"/"$file" "$destination"/
done

done < a.txt

svcdir=ui
for file  in `ls "$source"/"$svcdir" | grep -v ext`
do
echo   $file
cp "$source"/"$svcdir"/"$file" "$destination"/
done


#######################################################create all services.txt##########################################
cd $source
find . -name "*dep.yml" | sed 's/\// /g' | awk '{print $2}' >$WORKSPACE/allservices.txt
cd $WORKSPACE

#######################################################create all extservices.txt##########################################

grep -vwFf a.txt allservices.txt >extservices.txt

while read -r svcdir; do
ls -ltr "$source"/"$svcdir"
for file  in `ls "$source"/"$svcdir" | grep ext`
do
echo   $file
cp "$source"/"$svcdir"/"$file" "$destination"/
done

done < extservices.txt

echo
echo sedding deployments and ingresses

##################################copy all ingresses###############################

for ing in `find "$source" -name "*ing.yml"`
do
echo $ing
cp $ing "$destination"/
done

##################################sed all volume paths and sed ingress hosts#########################################
cd $destination
find . -name "*dep.yml" | xargs sed -i s"/sidda/$devname/g"

find . -name "*ing.yml" | xargs sed -i s"/host: tl-oke-newui.qloudable-npr.com/host: tl-oke-$devname.qloudable-npr.com/g"

##################################sed all volume paths and sed ingress hosts#########################################

kubectl -n "$devname" create configmap gitinit-config --from-literal=devname="$devname" --from-literal=gitusername="$gitusername" --from-literal=gitpassword="$gitpassword" --from-literal=giturl="$giturl" --from-literal=gitbranch="$gitbranch" --from-literal=services="$svc"

kubectl -n "$devname" get cm gitinit-config


echo services changed by developer
echo $services

echo services changed by developer and all services that call them
echo $svc


kubectl -n "$devname" apply -f /home/ubuntu/qudos/kubernetes-stack-files/gitmanage/init

#kubectl -n "$devname" get job
kubectl -n "$devname" get po

sleep 60

nodecount=$(  kubectl get nodes | grep Ready | wc -l )
count=$(  for pod in $( kubectl -n "$devname" get po | grep initjob | grep Running | awk '{ print $1 }' ); do kubectl -n "$devname" exec $pod -- ps -ef | grep sleep ; done | wc -l )
while [ $count -lt $nodecount ]
do 
echo git update is still happening...........................................
kubectl -n "$devname" get po
sleep 20s; 
count=$(  for pod in $( kubectl -n "$devname" get po | grep initjob | grep Running | awk '{ print $1 }' ); do kubectl -n "$devname" exec $pod -- ps -ef | grep sleep ; done | wc -l )
done

kubectl -n "$devname" get po
#kubectl -n "$devname" delete job --all

kubectl -n "$devname" apply -f $destination
 
```
### Update  the namespace

* Update script, creates an auto update for these environments without Jenkins/github hooks using nodemon image and kubernetes daemonset.
* While running the nodemon, node process is automatically restarts as soon as any code changes are detected.
   
 `./update-ns.sh`
```

#!/bin/bash
source ./setupenv.sh
       
echo updating the kube namespace
echo "$devname"
cd $devname
dirname=$(  echo $kubesource | sed 's/\// /g' | sed 's/.git//g' | awk ' {print $NF}' )
cd $dirname
kubectl -n $devname delete job --all
kubectl -n "$devname" apply -f ./gitmanage/update
sleep 30
kubectl -n "$devname" get job
kubectl -n "$devname" get po
# initcount=0
# while true
#do

# check=0
#  for pod in $(  kubectl -n "$devname" get po | grep ^updatejob | awk '{print $1}' )
 # do 
 # echo $pod;
#  if [ $( kubectl -n "$devname" logs $pod | grep "updated git repo and npm modules" | wc -l ) -gt $initcount ]
 # then
  #((check++))
 # fi
#  done
 # echo check="$check"
  
#  if [ $check -eq 3 ] 
#  then
 # ((initcount++))
 #  echo initcount="$initcount"
 #  echo all nodes updated
 # echo $pod
 # rm -rf serviceschanged.txt
 # kubectl -n $devname cp  $pod:/usr/src/services/"$devname"/qloudable-training-labs/serviceschanged.txt .
 # pwd
 # ls -ltr
  
  #for svc in `cat ./serviceschanged.txt | grep -v badge | grep -v deploy-image | sed 's/\// /g' | awk '{print $1}' | sort -u`
#do
 #  echo "here is where is you restart your container" $svc

#kubectl -n "$devname" scale --replicas=0 deployment/"$svc"
#kubectl -n "$devname" scale --replicas=1 deployment/"$svc"
#done
#fi
#done

```
### Add a Service to the namespace

* If a developer wants to add more services to the namespace, should use this script to add services to a namespace.
`add-service-namespace.sh`
```
#!/bin/bash -e
echo "$devname"
echo $services

 existing="$( kubectl -n srikala get cm  gitinit-config -o yaml | grep services: | sed s'/services://' | sed s"/'//g" )"
echo existing services $existing
services="$services $existing"
echo all services $services
dependency_file=$source/servicedeplendence.txt

rm -rf a.txt b.txt
touch a.txt b.txt

for x  in `echo $services`
do
echo   $x >>a.txt
cat $dependency_file | grep ^"$x" | awk '{print $2}' >> a.txt || echo $x has no callers
done

cat a.txt

for x  in `cat a.txt | sort -u`
do
echo   $x >>b.txt
cat $dependency_file | grep ^"$x" | awk '{print $2}' >> b.txt || echo $x has no callers
done



i=`cat b.txt | sort -u | wc -l`
j=`cat a.txt | sort -u | wc -l`


while [ $j -ne $i ]
do

cat b.txt | sort -u > a.txt
echo >b.txt

for x  in `cat a.txt | sort -u`
do
echo   $x >>b.txt
cat $dependency_file | grep ^"$x" | awk '{print $2}' >> b.txt || echo $x has no callers
done

i=`cat b.txt | sort -u | wc -l`
j=`cat a.txt | sort -u | wc -l`


done

cat b.txt | grep -v ssh-keygen-service | grep -v statemachine-services | grep -v stream-services  | sort -u >a.txt
echo $i
echo $j and below is the a.txt
cat a.txt
echo

sed -i '/^$/d' a.txt
svc=""
while read -r line; do
svc="$svc $line"
#echo $svc
done < a.txt

echo $svc


destination="$destination"/$devname
mkdir -p $destination
rm -rf "$destination"/*
#######################################copy all files from from source#######################
while read -r svcdir; do
ls -ltr "$source"/"$svcdir"
for file  in `ls "$source"/"$svcdir" | grep -v ext `
do
echo   $file
cp "$source"/"$svcdir"/"$file" "$destination"/
done

done < a.txt

svcdir=ui
for file  in `ls "$source"/"$svcdir" | grep -v ext`
do
echo   $file
cp "$source"/"$svcdir"/"$file" "$destination"/
done


#######################################################create all services.txt##########################################
cd $source
find . -name "*dep.yml" | sed 's/\// /g' | awk '{print $2}' >$WORKSPACE/allservices.txt
cd $WORKSPACE

#######################################################create all extservices.txt##########################################

grep -vwFf a.txt allservices.txt >extservices.txt

while read -r svcdir; do
ls -ltr "$source"/"$svcdir"
for file  in `ls "$source"/"$svcdir" | grep ext`
do
echo   $file
cp "$source"/"$svcdir"/"$file" "$destination"/
done

done < extservices.txt

echo
echo sedding deployments and ingresses

##################################copy all ingresses###############################

for ing in `find "$source" -name "*ing.yml"`
do
echo $ing
cp $ing "$destination"/
done

##################################sed all volume paths and sed ingress hosts#########################################
cd $destination
find . -name "*dep.yml" | xargs sed -i s"/sidda/$devname/g"

find . -name "*ing.yml" | xargs sed -i s"/host: tl-oke-newui.qloudable-npr.com/host: tl-oke-$devname.qloudable-npr.com/g"

##################################sed all volume paths and sed ingress hosts#########################################

kubectl -n "$devname" delete cm gitinit-config
kubectl -n "$devname" create configmap gitinit-config --from-literal=devname="$devname" --from-literal=gitusername="$gitusername" --from-literal=gitpassword="$gitpassword" --from-literal=giturl="$giturl" --from-literal=gitbranch="$gitbranch" --from-literal=services="$svc"

kubectl -n "$devname" get cm gitinit-config


echo services changed by developer
echo $services

echo services changed by developer and all services that call them
echo $svc

kubectl -n "$devname" delete ds --all
kubectl -n "$devname" apply -f /home/ubuntu/qudos/kubernetes-stack-files/gitmanage/init

kubectl -n "$devname" get job
kubectl -n "$devname" get po

sleep 60

kubectl -n "$devname" get po

nodecount=$(  kubectl get nodes | grep Ready | wc -l )
count=$(  for pod in $( kubectl -n "$devname" get po | grep initjob | grep Running | awk '{ print $1 }' ); do kubectl -n "$devname" exec $pod -- ps -ef | grep sleep ; done | wc -l )
while [ $count -lt $nodecount ]
do 
echo git update is still happening...........................................
kubectl -n "$devname" get po
sleep 20s; 
count=$(  for pod in $( kubectl -n "$devname" get po | grep initjob | grep Running | awk '{ print $1 }' ); do kubectl -n "$devname" exec $pod -- ps -ef | grep sleep ; done | wc -l )
done

kubectl -n "$devname" get po
#kubectl -n "$devname" delete job --all



kubectl -n "$devname" delete deploy --all
kubectl -n "$devname" delete svc --all
#kubectl -n "$devname" delete cm --all
#kubectl -n "$devname" delete ing --all
kubectl -n "$devname" apply -f $destination
 
```

### Remove a service from the namespace

* Remove the services which dont want to work with those services, can use remove service script to remove services from the namespace.

`(remove-service-namespace.sh)`
```
#!/bin/bash -e
echo "$devname"
echo $services

 existing="$( kubectl -n <namespace-name> get cm  gitinit-config -o yaml | grep services: | sed s'/services://' | sed s"/'//g" )"
echo existing services $existing


newservices=$existing
for s in $( echo $services )
do
newservices=$( echo $newservices | sed s"/$s//g" )
echo $newservices
done

services="$newservices"
echo all services $services
dependency_file=$source/servicedeplendence.txt

rm -rf a.txt b.txt
touch a.txt b.txt

for x  in `echo $services`
do
echo   $x >>a.txt
cat $dependency_file | grep ^"$x" | awk '{print $2}' >> a.txt || echo $x has no callers
done

cat a.txt

for x  in `cat a.txt | sort -u`
do
echo   $x >>b.txt
cat $dependency_file | grep ^"$x" | awk '{print $2}' >> b.txt || echo $x has no callers
done



i=`cat b.txt | sort -u | wc -l`
j=`cat a.txt | sort -u | wc -l`


while [ $j -ne $i ]
do

cat b.txt | sort -u > a.txt
echo >b.txt

for x  in `cat a.txt | sort -u`
do
echo   $x >>b.txt
cat $dependency_file | grep ^"$x" | awk '{print $2}' >> b.txt || echo $x has no callers
done

i=`cat b.txt | sort -u | wc -l`
j=`cat a.txt | sort -u | wc -l`


done

cat b.txt | grep -v ssh-keygen-service | grep -v statemachine-services | grep -v stream-services | sort -u >a.txt
echo $i
echo $j
echo

sed -i '/^$/d' a.txt
svc=""
while read -r line; do
svc="$svc $line"
#echo $svc
done < a.txt

echo $svc


destination="$destination"/$devname
mkdir -p $destination
rm -rf "$destination"/*
#######################################copy all files from from source#######################
while read -r svcdir; do
#ls -ltr "$source"/"$svcdir"
for file  in `ls "$source"/"$svcdir" | grep -v ext `
do
echo   $file
cp "$source"/"$svcdir"/"$file" "$destination"/
done

done < a.txt

svcdir=ui
for file  in `ls "$source"/"$svcdir" | grep -v ext`
do
echo   $file
cp "$source"/"$svcdir"/"$file" "$destination"/
done


#######################################################create all services.txt##########################################
cd $source
find . -name "*dep.yml" | sed 's/\// /g' | awk '{print $2}' >$WORKSPACE/allservices.txt
cd $WORKSPACE

#######################################################create all extservices.txt##########################################

grep -vwFf a.txt allservices.txt >extservices.txt

while read -r svcdir; do
#ls -ltr "$source"/"$svcdir"
for file  in `ls "$source"/"$svcdir" | grep ext`
do
echo   $file
cp "$source"/"$svcdir"/"$file" "$destination"/
done

done < extservices.txt

echo
echo sedding deployments and ingresses

##################################copy all ingresses###############################

for ing in `find "$source" -name "*ing.yml"`
do
echo $ing
cp $ing "$destination"/
done

##################################sed all volume paths and sed ingress hosts#########################################
cd $destination
find . -name "*dep.yml" | xargs sed -i s"/sidda/$devname/g"

find . -name "*ing.yml" | xargs sed -i s"/host: tl-oke-newui.qloudable-npr.com/host: tl-oke-$devname.qloudable-npr.com/g"

##################################sed all volume paths and sed ingress hosts#########################################

kubectl -n "$devname" delete cm gitinit-config
kubectl -n "$devname" create configmap gitinit-config --from-literal=devname="$devname" --from-literal=gitusername="$gitusername" --from-literal=gitpassword="$gitpassword" --from-literal=giturl="$giturl" --from-literal=gitbranch="$gitbranch" --from-literal=services="$svc"

kubectl -n "$devname" get cm gitinit-config


echo services changed by developer
echo $services

echo services changed by developer and all services that call them
echo $svc

kubectl -n "$devname" delete deploy --all
kubectl -n "$devname" delete svc --all
#kubectl -n "$devname" delete cm --all
#kubectl -n "$devname" delete ing --all

kubectl -n "$devname" delete ds --all
kubectl -n "$devname" apply -f /home/ubuntu/qudos/kubernetes-stack-files/gitmanage/init

kubectl -n "$devname" get ds
kubectl -n "$devname" get po

sleep 60

kubectl -n "$devname" get po

nodecount=$(  kubectl get nodes | grep Ready | wc -l )
count=$(  for pod in $( kubectl -n "$devname" get po | grep initjob | grep Running | awk '{ print $1 }' ); do kubectl -n "$devname" exec $pod -- ps -ef | grep sleep ; done | wc -l )
while [ $count -lt $nodecount ]
do 
echo git update is still happening...........................................
kubectl -n "$devname" get po
sleep 20s; 
count=$(  for pod in $( kubectl -n "$devname" get po | grep initjob | grep Running | awk '{ print $1 }' ); do kubectl -n "$devname" exec $pod -- ps -ef | grep sleep ; done | wc -l )
done

kubectl -n "$devname" get po
#kubectl -n "$devname" delete job --all



kubectl -n "$devname" apply -f $destination
 
```

### Delete the namespace

 * ./delete-ns.sh--> This script deletes the namespace with option of soft delete ( to be added ).

```

#!/bin/bash -e
#set up your env variables in setupenv.sh
source ./setupenv.sh
echo "$devname"
kubectl -n "$devname" apply -f ./gitmanage/delete
kubectl -n "$devname" get po
while [ `kubectl -n "$devname" get po | grep deletejob | grep -w Completed | wc -l` -lt 3 ]
do 
echo git update is still happening...........................................
sleep 5s; 
done
kubectl -n "$devname" get po
kubectl delete ns "$devname"
kubectl get ns

```

### Conclusion:
 

Congratulations! You have successfully completed the Qudos lab. In this lab, you create Developer scripts to automate deploy and debug connect to pods in cluster and Admin Scripts for Qudos .

Feel free to continue exploring or start a new lab.

Thank you for taking this training lab!
    


