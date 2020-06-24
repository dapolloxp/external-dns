# Set up ExternalDNS for Azure Private DNS

This tutorial describes how to set up ExternalDNS for managing records in Azure Private DNS on an AKS Cluster.  

It comprises of the following steps:

1) Installing an AKS Cluster with the App Gateway Add-on.
2) Grant permissions to the AKS Cluster over the VNET
3) Provision Azure Private DNS
4) Configure AAD Pod Identity
5) Configure a managed identity for managing the zone
6) Deploy a private front-end IP to the App Gateway Ingress Controller
7) Deploy ExternalDNS
8) Deploy a sample app to confirm that ExternalDNS is working with the Ingress Controller
9) Deploy a sample app to confirm that ExternalDNS is working with an internal kubernetes service

Everything will be deployed on Kubernetes.
Therefore, please see the subsequent prerequisites.

## Prerequisites

- A VNET is already created and private connectivity is established via ExpressRoute or a Site-to-Site VPN
- [Azure CLI 2.0](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) and `kubectl` installed on the box to execute the subsequent steps
- The AKS-IngressApplicationGatewayAddon has been installed and registered: https://docs.microsoft.com/en-us/azure/application-gateway/tutorial-ingress-controller-add-on-new
- kubectl is installed

## Deploy a new AKS Cluster with the App Gateway Add-on (AGIC) enabled

The steps below are modified from the original document:

https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/application-gateway/tutorial-ingress-controller-add-on-new.md

## Create a resource group

In Azure, you allocate related resources to a resource group. Create a resource group by using [az group create](/cli/azure/group#az-group-create). The following example creates a resource group named *aks-rg* in the *eastus2* location (region). 

```azurecli-interactive
az group create --name aks-rg --location eastus2
```


# Deploy AKS with the AGIC add-on (New Cluster with managed identity enabled and Azure CNI)

AKS with the App Gateway add-on can be enabled at deployment by running the following command:


```
$ az aks create \
    --resource-group <your resource group name> \
    --name <your AKS cluster name> \
    --node-count 1 \
    --network-plugin azure \
    --enable-managed-identity \
    --generate-ssh-keys \
    --enable-addons ingress-appgw \
    --appgw-name <your app gateway name> \
    --kubernetes-version 1.16.9 \
    --docker-bridge-address 172.17.0.1/16 \
    --service-cidr 10.211.0.0/16 \
    --dns-service-ip 10.211.0.10 \
    --vnet-subnet-id <your vnet subnet id> \
    --appgw-subnet-id <your app gateway subnet id> \
    --appgw-watch-namespace default
    --node-resource-group <AKS node pool RG>
```
  
Once the cluster is deployed, an app gateway instance will be created and tied to AKS.

The credentials will need to be merged into your existing kubeconfig file. In my example below, my cluster name is aks-s1 and it resides in the aks-rg resource group.
```
$ az aks get-credentials -g aks-rg -n aks-s1

```

Note: By default AKS will automatically create a cluster resources group that contains the app gateway instance and the cluster node pools. It will generally follow the format of MC_<resource_group_name>_<cluster_name>_<region>. In my example above, the cluster resources group is *aks-c1-nodes-rg* since I provided the node resource group name at provisioning time. 

We can query for the addonProfiles to ensure that the ingress controller as been added.

```
$ az aks show --resource-group aks-rg --name aks-c1 --query addonProfiles

```

The output will be similar to the following:
```
"IngressApplicationGateway": {
    "config": {
      "applicationGatewayName": "appgw-aks",
      "effectiveApplicationGatewayId": "/subscriptions/<sub id>/resourceGroups/MC_aks-rg_aks-c1_eastus2/providers/Microsoft.Network/applicationGateways/appgw-aks",
      "subnetId": "/subscriptions/<sub id>/resourceGroups/AzureHubVNET/providers/Microsoft.Network/virtualNetworks/<VNET name>/subnets/<subnet name>",
      "watchNamespace": "default"
    }
```

## Grant permissions to the AKS Cluster over the VNET

https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni#prerequisites

As per the prerequisites documented in the link above, the AKS cluster must have at least network contributor to the VNET. However, in highly restricted environments, this may not be an option. The two permissions that need to be granted are as follows:
    Microsoft.Network/virtualNetworks/subnets/join/action
    Microsoft.Network/virtualNetworks/subnets/read

In this example, I will be granting the AKS Cluster Network Contributor rights

```
# Query for the AKS cluster Client ID

$ az aks show -g aks-rg -n aks-c1 --query identity

{
  "principalId": "<app id>",
  "tenantId": "<tenant id>",
  "type": "SystemAssigned"
}

# Grant Network Contributor to the AKS Cluster

$ az role assignment create --role "Network Contributor" --assignee <app Id> --scope /subscriptions/<SubscriptionID>/resourcegroups/<Network RG>

```

In my example, my cluster is as follows:

```
az role assignment create --role "Network Contributor" --assignee <appId GUID> --scope /subscriptions/<SubscriptionID>/resourcegroups/AzureHubVNET
```

## Provision Azure Private DNS

The provider will find suitable zones for domains it manages. It will not automatically create zones.

For this tutorial, we will create an Azure Private DNS zone within the existing VNET resource group.

As mentioned above, there is an assumption that a VNET has already been created. In my example, my existing VNET resides in a resource group called AzureHubVNET

create a Azure Private DNS zone for "contoso.com":

```
$ az network private-dns zone create -g AzureHubVNET -n contoso.com
```

Substitute a domain you own for "contoso.com".

Finally, create the mentioned link with the VNET.

In my example, my existing VNET is named AzureEUS2VNET1 and contained in a resource group called AzureHubVNET

```
$ az network private-dns link vnet create -g AzureHubVNET -n mylink -z contoso.com -v AzureEUS2VNET1 --registration-enabled false
```

## Configure user managed identity for managing the zone

ExternalDNS needs permissions to make changes in Azure Private DNS.  
These permissions are roles assigned to the user managed identity used by ExternalDNS.

A user identity with a minimum access level of `Private DNS Zone Contributor` to the Private DNS zone(s) and `Reader` to the resource group containing the Azure Private DNS zone(s) is necessary.

More powerful role-assignments like `Contributor` or assignments on subscription-level work too. 

Start off by **creating the user managed identity** without role-assignments.

I will be creating the user managed identity in the AKS nodes RG.

az identity create -g <RESOURCE GROUP> -n <USER ASSIGNED IDENTITY NAME>
```
$ az identity create -g aks-c1-nodes-rg -n externaldnsmi

{
  "clientId": "2bed99cb-b6bb-4d77-b823-f904545c21b1",
  ......
  "id": "/subscriptions/<Sub ID>/resourcegroups/aks-c1-nodes-rg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/externaldnsmi",
  "location": "eastus2",
  "name": "externaldnsmi",
  "principalId": "d060d57c-7084-40ea-a7b4-8a9dc610b86d",
  "resourceGroup": "aks-c1-nodes-rg",  
  "tenantId": "<tenant ID>",
  "type": "Microsoft.ManagedIdentity/userAssignedIdentities"
}
```
Get the resource ID for the user identity:

```
az identity show --name externaldnsmi --resource-group aks-c1-nodes-rg --subscription <sub id>
```
{
  ...
  "id": "/subscriptions/<Sub id>/resourcegroups/aks-c1-nodes-rg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/externaldnsmi",
  "location": "eastus2",
  "name": "externaldnsmi",
  "principalId": "<app id>",
  ...
}


Next, assign the roles to the user identity.  
But first **retrieve the ID's** of the objects to assign roles on.

```
# find out the resource ids of the resource group where the dns zone is deployed, and the dns zone itself
$ az group show --name AzureHubVNET --query id -o tsv
/subscriptions/<Sub ID>/resourceGroups/AzureHubVNET

$ az network private-dns zone show --name contoso.com -g AzureHubVNET --query id -o tsv
/subscriptions/<Sub ID>/resourceGroups/azurehubvnet/providers/Microsoft.Network/privateDnsZones/contoso.com
```
**create role assignments**
```
# 1. as a reader to the resource group
$ az role assignment create --role "Reader" --assignee <appId GUID> --scope <resource group resource id>  

# 2. as a contributor to DNS Zone itself
$ az role assignment create --role "Private DNS Zone Contributor" --assignee <appId GUID> --scope <dns zone resource id>  
```

## Deploy Azure AD Pod Identity (RBAC-enabled cluster)

Configure `kubectl` to be able to communicate and authenticate with your cluster.   
This is per default done through the file `~/.kube/config`.

For general background information on this see [kubernetes-docs](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/).  
Azure-CLI features functionality for automatically maintaining this file for AKS-Clusters. See [Azure-Docs](https://docs.microsoft.com/de-de/cli/azure/aks?view=azure-cli-latest#az-aks-get-credentials).

In my example, I am deploying Pod Identity to an RBAC-enabled AKS cluster. Refer to the following link for the most update to date instructions:

https://github.com/Azure/aad-pod-identity#1-deploy-aad-pod-identity

Deploy aad-pod-identity components to an RBAC-enabled cluster:
```
$ kubectl apply -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment.yaml

# For AKS clusters, deploy the MIC and AKS add-on exception by running -
$ kubectl apply -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/mic-exception.yaml 

```

AKS Clusters with Pod Identity require specific permissions to functional properly. This is documented in detail here:

https://github.com/Azure/aad-pod-identity/blob/master/docs/readmes/README.role-assignment.md

In this scenario, the AKS cluster has been created with the --managed-identity flag and will have two managed identities for the cluster:

## Obtaining the ID of the managed identity for the AKS Cluster

```
az aks show -g <AKSResourceGroup> -n <AKSClusterName> --query identityProfile.kubeletidentity.clientId -otsv
```

In my example below, I am using my resource group and cluster name.

```
az aks show -g aks-rg -n aks-c1 --query identityProfile.kubeletidentity.clientId -otsv
```

The roles Managed Identity Operator and Virtual Machine Contributor must be assigned to the cluster managed identity or service principal, identified by the ID obtained above, before deploying AAD Pod Identity so that it can assign and un-assign identities from the underlying VM/VMSS.

For AKS cluster, the cluster resource group refers to the resource group with a MC_ prefix, which contains all of the infrastructure resources associated with the cluster like VM/VMSS. In my example below, I am using the custom resource group name that I defined in the creation of the AKS cluster

```
az role assignment create --role "Managed Identity Operator" --assignee <ID> --scope /subscriptions/<SubscriptionID>/resourcegroups/<ClusterResourceGroup>
az role assignment create --role "Virtual Machine Contributor" --assignee <ID> --scope /subscriptions/<SubscriptionID>/resourcegroups/<ClusterResourceGroup>

# Pod Identity requires permissions on the VNET. The minimum rights are VNET join. A custom role can be granted with this permission. Always refer to the latest Pod Identity documentation as the permissions can change.
az role assignment create --role "Virtual Machine Contributor" --assignee <ID> --scope /subscriptions/<SubscriptionID>/resourcegroups/<VNETResourceGroup>
```

In my example, I will run the following commands:

```
az role assignment create --role "Managed Identity Operator" --assignee <AKS ClientID> --scope /subscriptions/<Sub ID>/resourceGroups/aks-c1-nodes-rg
az role assignment create --role "Virtual Machine Contributor" --assignee <AKS ClientID> --scope /subscriptions/<Sub ID>/resourceGroups/aks-c1-nodes-rg
az role assignment create --role "Virtual Machine Contributor" --assignee <AKS ClientID> --scope /subscriptions/<Sub ID>/resourceGroups/AzureHubVNet
az role assignment create --role "Virtual Machine Contributor" --assignee <AKS Node Node Pool ClientID> --scope 
```

## Deploy a private front-end IP to the App Gateway Ingress Controller

In my example below, I am setting a private front-end IP on my subnet (named public), for the AKS Ingress Controller. Since my VNET and App Gateway reside in different resource groups, I have to specific the resource ID for the subnet parameter. In my example below, the public subnet CIDR rangeis

```
az network application-gateway frontend-ip create --gateway-name appgw-aks --name aks-private-ingress --private-ip-address 10.20.2.200 --resource-group aks-c1-nodes-rg --subnet /subscriptions/<Sub Id>/resourceGroups/AzureHubVNET/providers/Microsoft.Network/virtualNetworks/AzureEUS2VNET1/subnets/public
```



## Deploy ExternalDNS

Apply one of the following manifests depending on whether you use RBAC or not (Note: RBAC is the only option in this doc at the moment).

Create the following file 'external.yaml' with the contents below: 

Note: Ensure that the user idenitity is populated with the correct values.

### Manifest (for clusters with RBAC enabled, cluster access)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: externaldns
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: externaldns
rules:
- apiGroups: [""]
  resources: ["services","endpoints","pods"]
  verbs: ["get","watch","list"]
- apiGroups: ["extensions"] 
  resources: ["ingresses"] 
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: externaldns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: externaldns
subjects:
- kind: ServiceAccount
  name: externaldns
  namespace: default
---
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentity
metadata:
  # Important: Name must be the same as the Resource Name!
  # Warning: The attribute names are case-sensitive: https://github.com/Azure/aad-pod-identity#v160-breaking-change
  name: externaldns-mui
spec:
  type: 0
  resourceID: /subscriptions/<Sub ID>/resourcegroups/aks-c1-nodes-rg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/externaldnsmi
  clientID: <externaldnsmi clientid>
---
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentityBinding
metadata:
  name: externaldns-mui-binding
spec:
  azureIdentity: externaldns-mui
  selector: externaldns-mui-selector
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: externaldns
  labels:
    app: externaldns
    aadpodidbinding: externaldns-mui-selector
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: externaldns
  template:
    metadata:
      labels:
        app: externaldns
        aadpodidbinding: externaldns-mui-selector
    spec:
      serviceAccountName: externaldns
      containers:
      - name: externaldns
        image: registry.opensource.zalan.do/teapot/external-dns:latest
        args:
        - --source=service
        - --source=ingress
        - --domain-filter=contoso.com
        - --provider=azure-private-dns
        - --azure-resource-group=azurehubvnet
        - --azure-subscription-id=<sub id>
```

Create the deployment for ExternalDNS:

```
$ kubectl create -f externaldns.yaml
```

## Deploying sample service

Create a service file called 'aspnet.yaml' with the following contents:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aspnetapp
  labels:
    app: aspnetapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: aspnetapp
  template:
    metadata:
      labels:
        app: aspnetapp
    spec:
      containers:
      - name: aspnetapp-image
        image: "mcr.microsoft.com/dotnet/core/samples:aspnetapp"
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: aspnetapp
spec:
  selector:
    app: aspnetapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: aspnetapp
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
    appgw.ingress.kubernetes.io/use-private-ip: "true"
spec:
  rules:
  - host: aspnetapp.contoso.com
  - http:
      paths:
      - path: /
        backend:
          serviceName: aspnetapp
          servicePort: 80
```

When using ExternalDNS with ingress objects it will automatically create DNS records based on host names specified in ingress objects that match the domain-filter argument in the externaldns deployment manifest. When those host names are removed or renamed the corresponding DNS records are also altered.

Create the deployment, service and ingress object:

```
$ kubectl create -f aspnet.yaml
```

Your external IP will have the private IP associated to the app gateway ingress service.

```
$ kubectl get ingress

NAME        HOSTS                   ADDRESS       PORTS   AGE
aspnetapp   aspnetapp.contoso.com   10.20.2.200   80      16m
```
## Deploy a sample app to confirm that ExternalDNS is working with an internal kubernetes service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azure-vote-back
spec:
  replicas: 1
  selector:
    matchLabels:
      app: azure-vote-back
  template:
    metadata:
      labels:
        app: azure-vote-back
    spec:
      nodeSelector:
        "beta.kubernetes.io/os": linux
      containers:
      - name: azure-vote-back
        image: redis
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
        ports:
        - containerPort: 6379
          name: redis
---
apiVersion: v1
kind: Service
metadata:
  name: azure-vote-back
spec:
  ports:
  - port: 6379
  selector:
    app: azure-vote-back
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azure-vote-front
spec:
  replicas: 1
  selector:
    matchLabels:
      app: azure-vote-front
  template:
    metadata:
      labels:
        app: azure-vote-front
    spec:
      nodeSelector:
        "beta.kubernetes.io/os": linux
      containers:
      - name: azure-vote-front
        image: microsoft/azure-vote-front:v1
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
        ports:
        - containerPort: 80
        env:
        - name: REDIS
          value: "azure-vote-back"
---
apiVersion: v1
kind: Service
metadata:
  name: azure-vote-front
  annotations:
     service.beta.kubernetes.io/azure-load-balancer-internal: "true"
     external-dns.alpha.kubernetes.io/hostname: azure-vote-front.contoso.com
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: azure-vote-front
```

## Verify created records

Run the following command to view the A records for your Azure Private DNS zone:

```
$ az network private-dns record-set a list -g AzureHubVNET -z contoso.com
```

Substitute the zone for the one created above if a different domain was used.

This should show the external IP address of the service as the A record for your domain ('@' indicates the record is for the zone itself).

```
[
  {
    "aRecords": [
      {
        "ipv4Address": "10.20.2.200"
      }
    ],
    "etag": "22726f62-077e-4a57-a6e1-aed0d1698fbc",
    "fqdn": "aspnetapp.contoso.com.",
    "id": "/subscriptions/<Sub ID>/resourceGroups/azurehubvnet/providers/Microsoft.Network/privateDnsZones/contoso.com/A/aspnetapp",
    "isAutoRegistered": false,
    "metadata": null,
    "name": "aspnetapp",
    "resourceGroup": "azurehubvnet",
    "ttl": 300,
    "type": "Microsoft.Network/privateDnsZones/A"
  },
  {
    "aRecords": [
      {
        "ipv4Address": "10.20.10.4"
      }
    ],
    "etag": "85174ad9-bf44-45a3-a092-882ce0f67045",
    "fqdn": "azure-vote-front.contoso.com.",
    "id": "/subscriptions/<Sub ID>/resourceGroups/azurehubvnet/providers/Microsoft.Network/privateDnsZones/contoso.com/A/azure-vote-front",
    "isAutoRegistered": false,
    "metadata": null,
    "name": "azure-vote-front",
    "resourceGroup": "azurehubvnet",
    "ttl": 300,
    "type": "Microsoft.Network/privateDnsZones/A"
  }
]
```
