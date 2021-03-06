---
title: Secure pods with network policies in Azure Kubernetes Service (AKS)
description: Learn how to secure traffic that flows in and out of pods by using Kubernetes network policies in Azure Kubernetes Service (AKS)
services: container-service
author: iainfoulds

ms.service: container-service
ms.topic: article
ms.date: 02/12/2019
ms.author: iainfou
---

# Preview - Secure traffic between pods using network policies in Azure Kubernetes Service (AKS)

When you run modern, microservices-based applications in Kubernetes, you often want to control which components can communicate with each other. The principle of least privilege should be applied to how traffic can flow between pods in an Azure Kubernetes Service (AKS) cluster. Let's say you likely want to block traffic directly to back-end applications. The *Network Policy* feature in Kubernetes lets you define rules for ingress and egress traffic between pods in a cluster.

Calico, an open source networking and network security solution founded by Tigera, offers a network policy engine which can implement Kubernetes network policy rules. This article shows you how to install the Calico network policy engine and create Kubernetes network policies to control the flow of traffic between pods in AKS.

> [!IMPORTANT]
> AKS preview features are self-service and opt-in. Previews are provided to gather feedback and bugs from our community. However, they are not supported by Azure technical support. If you create a cluster, or add these features to existing clusters, that cluster is unsupported until the feature is no longer in preview and graduates to general availability (GA).
>
> If you encounter issues with preview features, [open an issue on the AKS GitHub repo][aks-github] with the name of the preview feature in the bug title.

## Before you begin

You need the Azure CLI version 2.0.56 or later installed and configured. Run `az --version` to find the version. If you need to install or upgrade, see [Install Azure CLI][install-azure-cli].

To create an AKS cluster that can use network policy, first enable a feature flag on your subscription. To register the *EnableNetworkPolicy* feature flag, use the [az feature register][az-feature-register] command as shown in the following example:

```azurecli-interactive
az feature register --name EnableNetworkPolicy --namespace Microsoft.ContainerService
```

It takes a few minutes for the status to show *Registered*. You can check on the registration status by using the [az feature list][az-feature-list] command:

```azurecli-interactive
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/EnableNetworkPolicy')].{Name:name,State:properties.state}"
```

When ready, refresh the registration of the *Microsoft.ContainerService* resource provider by using the [az provider register][az-provider-register] command:

```azurecli-interactive
az provider register --namespace Microsoft.ContainerService
```

## Overview of network policy

All pods in an AKS cluster can send and receive traffic without limitations, by default. To improve security, you can define rules that control the flow of traffic. Back-end applications are often only exposed to required front-end services, for example. Or, database components are only accessible to the application tiers that connect to them.

Network policies are Kubernetes resources that let you control the traffic flow between pods. You can choose to allow or deny traffic based on settings like assigned labels, namespace, or traffic port. Network policies are defined as YAML manifests. These policies can be included as part of a wider manifest that also creates a deployment or service.

To see network policies in action, let's create and then expand on a policy that defines traffic flow:

* Deny all traffic to pod.
* Allow traffic based on pod labels.
* Allow traffic based on namespace.

## Create an AKS cluster and enable network policy

Network policy can only be enabled when the cluster is created. You can't enable network policy on an existing AKS cluster. 

To use network policy with an AKS cluster, you must use the [Azure CNI plug-in][azure-cni] and define your own virtual network and subnets. For more detailed information on how to plan out the required subnet ranges, see [configure advanced networking][use-advanced-networking].

The following example script:

* Creates a virtual network and subnet.
* Creates an Azure Active Directory (Azure AD) service principal for use with the AKS cluster.
* Assigns *Contributor* permissions for the AKS cluster service principal on the virtual network.
* Creates an AKS cluster in the defined virtual network and enables network policy.

Provide your own secure *SP_PASSWORD*. You can replace the *RESOURCE_GROUP_NAME* and *CLUSTER_NAME* variables:

```azurecli-interactive
SP_PASSWORD=mySecurePassword
RESOURCE_GROUP_NAME=myResourceGroup-NP
CLUSTER_NAME=myAKSCluster
LOCATION=canadaeast

# Create a resource group
az group create --name $RESOURCE_GROUP_NAME --location $LOCATION

# Create a virtual network and subnet
az network vnet create \
    --resource-group $RESOURCE_GROUP_NAME \
    --name myVnet \
    --address-prefixes 10.0.0.0/8 \
    --subnet-name myAKSSubnet \
    --subnet-prefix 10.240.0.0/16

# Create a service principal and read in the application ID
SP_ID=$(az ad sp create-for-rbac --password $SP_PASSWORD --skip-assignment --query [appId] -o tsv)

# Wait 15 seconds to make sure that service principal has propagated
echo "Waiting for service principal to propagate..."
sleep 15

# Get the virtual network resource ID
VNET_ID=$(az network vnet show --resource-group $RESOURCE_GROUP_NAME --name myVnet --query id -o tsv)

# Assign the service principal Contributor permissions to the virtual network resource
az role assignment create --assignee $SP_ID --scope $VNET_ID --role Contributor

# Get the virtual network subnet resource ID
SUBNET_ID=$(az network vnet subnet show --resource-group $RESOURCE_GROUP_NAME --vnet-name myVnet --name myAKSSubnet --query id -o tsv)

# Create the AKS cluster and specify the virtual network and service principal information
# Enable network policy by using the `--network-policy` parameter
az aks create \
    --resource-group $RESOURCE_GROUP_NAME \
    --name $CLUSTER_NAME \
    --node-count 1 \
    --kubernetes-version 1.12.6 \
    --generate-ssh-keys \
    --network-plugin azure \
    --service-cidr 10.0.0.0/16 \
    --dns-service-ip 10.0.0.10 \
    --docker-bridge-address 172.17.0.1/16 \
    --vnet-subnet-id $SUBNET_ID \
    --service-principal $SP_ID \
    --client-secret $SP_PASSWORD \
    --network-policy calico
```

It takes a few minutes to create the cluster. When the cluster is ready, configure `kubectl` to connect to your Kubernetes cluster by using the [az aks get-credentials][az-aks-get-credentials] command. This command downloads credentials and configures the Kubernetes CLI to use them:

```azurecli-interactive
az aks get-credentials --resource-group $RESOURCE_GROUP_NAME --name $CLUSTER_NAME
```

## Deny all inbound traffic to a pod

Before you define rules to allow specific network traffic, first create a network policy to deny all traffic. This policy gives you a starting point to begin to whitelist only the  desired traffic. You can also clearly see that traffic is dropped when the network policy is applied.

For the sample application environment and traffic rules, let's first create a namespace called *development* to run the example pods:

```console
kubectl create namespace development
kubectl label namespace/development purpose=development
```

Create an example back-end pod that runs NGINX. This back-end pod can be used to simulate a sample back-end web-based application. Create this pod in the *development* namespace, and open port *80* to serve web traffic. Label the pod with *app=webapp,role=backend* so that we can target it with a network policy in the next section:

```console
kubectl run backend --image=nginx --labels app=webapp,role=backend --namespace development --expose --port 80 --generator=run-pod/v1
```

Create another pod and attach a terminal session to test that you can successfully reach the default NGINX webpage:

```console
kubectl run --rm -it --image=alpine network-policy --namespace development --generator=run-pod/v1
```

At the shell prompt, use `wget` to confirm that you can access the default NGINX webpage:

```console
wget -qO- http://backend
```

The following sample output shows that the default NGINX webpage returned:

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
[...]
```

Exit out of the attached terminal session. The test pod is automatically deleted.

```console
exit
```

### Create and apply a network policy

Now that you've confirmed you can use the basic NGINX webpage on the sample back-end pod, create a network policy to deny all traffic. Create a file named `backend-policy.yaml` and paste the following YAML manifest. This manifest uses a *podSelector* to attach the policy to pods that have the *app:webapp,role:backend* label, like your sample NGINX pod. No rules are defined under *ingress*, so all inbound traffic to the pod is denied:

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: backend-policy
  namespace: development
spec:
  podSelector:
    matchLabels:
      app: webapp
      role: backend
  ingress: []
```

Apply the network policy by using the [kubectl apply][kubectl-apply] command and specify the name of your YAML manifest:

```azurecli-interactive
kubectl apply -f backend-policy.yaml
```

### Test the network policy


Let's see if you can use the NGINX webpage on the back-end pod again. Create another test pod and attach a terminal session:

```console
kubectl run --rm -it --image=alpine network-policy --namespace development --generator=run-pod/v1
```

At the shell prompt, use `wget` to see if you can access the default NGINX webpage. This time, set a timeout value to *2* seconds. The network policy now blocks all inbound traffic, so the page can't be loaded, as shown in the following example:

```console
$ wget -qO- --timeout=2 http://backend

wget: download timed out
```

Exit out of the attached terminal session. The test pod is automatically deleted.

```console
exit
```

## Allow inbound traffic based on a pod label

In the previous section, a back-end NGINX pod was scheduled, and a network policy was created to deny all traffic. Let's create a front-end pod and update the network policy to allow traffic from front-end pods.

Update the network policy to allow traffic from pods with the labels *app:webapp,role:frontend* and in any namespace. Edit the previous *backend-policy.yaml* file, and add *matchLabels* ingress rules so that your manifest looks like the following example:

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: backend-policy
  namespace: development
spec:
  podSelector:
    matchLabels:
      app: webapp
      role: backend
  ingress:
  - from:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          app: webapp
          role: frontend
```

> [!NOTE]
> This network policy uses a *namespaceSelector* and a *podSelector* element for the ingress rule. The YAML syntax is important for the ingress rules to be additive. In this example, both elements must match for the ingress rule to be applied. Kubernetes versions prior to *1.12* might not interpret these elements correctly and restrict the network traffic as you expect. For more about this behavior, see [Behavior of to and from selectors][policy-rules].

Apply the updated network policy by using the [kubectl apply][kubectl-apply] command and specify the name of your YAML manifest:

```azurecli-interactive
kubectl apply -f backend-policy.yaml
```

Schedule a pod that is labeled as *app=webapp,role=frontend* and attach a terminal session:

```console
kubectl run --rm -it frontend --image=alpine --labels app=webapp,role=frontend --namespace development --generator=run-pod/v1
```

At the shell prompt, use `wget` to see if you can access the default NGINX webpage:

```console
wget -qO- http://backend
```

Because the ingress rule allows traffic with pods that have the labels *app: webapp,role: frontend*, the traffic from the front-end pod is allowed. The following example output shows the default NGINX webpage returned:

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
[...]
```

Exit out of the attached terminal session. The pod is automatically deleted.

```console
exit
```

### Test a pod without a matching label

The network policy allows traffic from pods labeled *app: webapp,role: frontend*, but should deny all other traffic. Let's test to see whether another pod without those labels can access the back-end NGINX pod. Create another test pod and attach a terminal session:

```console
kubectl run --rm -it --image=alpine network-policy --namespace development --generator=run-pod/v1
```

At the shell prompt, use `wget` to see if you can access the default NGINX webpage. The network policy blocks the inbound traffic, so the page can't be loaded, as shown in the following example:

```console
$ wget -qO- --timeout=2 http://backend

wget: download timed out
```

Exit out of the attached terminal session. The test pod is automatically deleted.

```console
exit
```

## Allow traffic only from within a defined namespace

In the previous examples, you created a network policy that denied all traffic, and then updated the policy to allow traffic from pods with a specific label. Another common need is to limit traffic to only within a given namespace. If the previous examples were for traffic in a *development* namespace, create a network policy that prevents traffic from another namespace, such as *production*, from reaching the pods.

First, create a new namespace to simulate a production namespace:

```console
kubectl create namespace production
kubectl label namespace/production purpose=production
```

Schedule a test pod in the *production* namespace that is labeled as *app=webapp,role=frontend*. Attach a terminal session:

```console
kubectl run --rm -it frontend --image=alpine --labels app=webapp,role=frontend --namespace production --generator=run-pod/v1
```

At the shell prompt, use `wget` to confirm that you can access the default NGINX webpage:

```console
wget -qO- http://backend.development
```

Because the labels for the pod match what is currently permitted in the network policy, the traffic is allowed. The network policy doesn't look at the namespaces, only the pod labels. The following example output shows the default NGINX webpage returned:

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
[...]
```

Exit out of the attached terminal session. The test pod is automatically deleted.

```console
exit
```

### Update the network policy

Let's update the ingress rule *namespaceSelector* section to only allow traffic from within the *development* namespace. Edit the *backend-policy.yaml* manifest file as shown in the following example:

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: backend-policy
  namespace: development
spec:
  podSelector:
    matchLabels:
      app: webapp
      role: backend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: development
      podSelector:
        matchLabels:
          app: webapp
          role: frontend
```

In more complex examples, you could define multiple ingress rules, like a *namespaceSelector* and then a *podSelector*.

Apply the updated network policy by using the [kubectl apply][kubectl-apply] command and specify the name of your YAML manifest:

```azurecli-interactive
kubectl apply -f backend-policy.yaml
```

### Test the updated network policy

Schedule another pod in the *production* namespace and attach a terminal session:

```console
kubectl run --rm -it frontend --image=alpine --labels app=webapp,role=frontend --namespace production --generator=run-pod/v1
```

At the shell prompt, use `wget` to see that the network policy now denies traffic:

```console
$ wget -qO- --timeout=2 http://backend.development

wget: download timed out
```

Exit out of the test pod:

```console
exit
```

With traffic denied from the *production* namespace, schedule a test pod back in the *development* namespace and attach a terminal session:

```console
kubectl run --rm -it frontend --image=alpine --labels app=webapp,role=frontend --namespace development --generator=run-pod/v1
```

At the shell prompt, use `wget` to see that the network policy allows the traffic:

```console
wget -qO- http://backend
```

Traffic is allowed because the pod is scheduled in the namespace that matches what's permitted in the network policy. The following sample output shows the default NGINX webpage returned:

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
[...]
```

Exit out of the attached terminal session. The test pod is automatically deleted.

```console
exit
```

## Clean up resources

In this article, we created two namespaces and applied a network policy. To clean up these resources, use the [kubectl delete][kubectl-delete] command and specify the resource names:

```console
kubectl delete namespace production
kubectl delete namespace development
```

## Next steps

For more about network resources, see [Network concepts for applications in Azure Kubernetes Service (AKS)][concepts-network].

To learn more about policies, see [Kubernetes network policies][kubernetes-network-policies].

<!-- LINKS - external -->
[kubectl-apply]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply
[kubectl-delete]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#delete
[kubernetes-network-policies]: https://kubernetes.io/docs/concepts/services-networking/network-policies/
[azure-cni]: https://github.com/Azure/azure-container-networking/blob/master/docs/cni.md
[terms-of-use]: https://azure.microsoft.com/support/legal/preview-supplemental-terms/
[policy-rules]: https://kubernetes.io/docs/concepts/services-networking/network-policies/#behavior-of-to-and-from-selectors
[aks-github]: https://github.com/azure/aks/issues]

<!-- LINKS - internal -->
[install-azure-cli]: /cli/azure/install-azure-cli
[use-advanced-networking]: configure-advanced-networking.md
[az-aks-get-credentials]: /cli/azure/aks?view=azure-cli-latest#az-aks-get-credentials
[concepts-network]: concepts-network.md
[az-feature-register]: /cli/azure/feature#az-feature-register
[az-feature-list]: /cli/azure/feature#az-feature-list
[az-provider-register]: /cli/azure/provider#az-provider-register
