# Velero-k8s
Take a Backup &amp; Restoration in K8s cluster using velero.

Velero is an open source tool that helps backup and restore Kubernetes resources. It also helps with migrating Kubernetes resources from one cluster to another. Also, it can help backup/restore data in persistent volumes

Below things required to test velero in Azure AKS
(i) One Azure account with proper access 
(ii) AKS Cluster
(iii) Storage account 

Prerequisites:

Need to install azure cli. We can do all the things either Azure cli or UI. 
Here I chosen Azure cli.

Install Azure cli in MAC
--------------------------
```
$ brew update && brew install azure-cli 
```
Login Azure account
```
$ az login 
```
Setup AKS Cluster
----------------------
Step 1: Create Resource Group 
```
Velero_Resource_Group="aks-test"
region=eastus
$ az group create --location $region --name $Velero_Resource_Group
```
Step 2: Create AKS Cluster

First check the all available k8s versions in the region where you going to setup AKS
```
$ az aks get-versions -l $region
AKS_Cluster_Name="aks-cluster-1"
$ az aks create --resource-group $Velero_Resource_Group --name $AKS_Cluster_Name --node-count 1 --kubernetes-version 1.18.0
```
Created AKS cluster with single node using above command. You can modify the node count using --node-count argument

Step 3: Verify the AKS Cluster which we created
```
$ az aks show --name $AKS_Cluster_Name --resource-group $Velero_Resource_Group
```

Step 4: Configure kubeconfig to local 

A file that is used to configure access to clusters is called a kubeconfig file. 
This is a generic way of referring to configuration files. It does not mean that there is a file named kubeconfig.

By default, kubectl looks for a file named config in the $HOME/.kube directory. You can specify other kubeconfig files by setting the KUBECONFIG environment variable or by setting the --kubeconfig flag
```
$ az aks get-credentials --resource-group $Velero_Resource_Group --name $AKS_Cluster_Name

$ kubectl get nodes 
```

Create Storage account
---------------------------
Need one storage account for velero backup files.
```
Velero_Storage_Account="testnewakstorage"
Velero_SA_blob_Container="testakscontainer"

$ az storage account create --name $Velero_Storage_Account --resource-group $Velero_Resource_Group --location $region --kind StorageV2 --sku Standard_LRS --encryption-services blob --https-only true --access-tier Hot

$ az storage container create --name $Velero_SA_blob_Container --public-access off --account-name $Velero_Storage_Account
```

Velero installation
-----------------------
(i) Velero Client (locally)

(ii) Velero Server in AKS Cluster
```
$ brew install velero  ----> Install velero package locally 

$ which velero ---> you can verify whether velero executable exist or not 
```
Install Velero Server in AKS Cluster
---------------------------------------
Need to pass some variables to install velero in AKS cluster

```
AZURE_SUBSCRIPTION_ID=`az account list --query '[?isDefault].id' -o tsv`
AZURE_TENANT_ID=`az account list --query '[?isDefault].tenantId' -o tsv`
AZURE_CLIENT_SECRET=$(az ad sp create-for-rbac -n $Velero_Storage_Account --role contributor --query password --output tsv)
AZURE_AKS_RESOURCE_GROUP=$(az aks show --query nodeResourceGroup --name $AKS_Cluster_Name --resource-group $Velero_Resource_Group --output tsv)
AZURE_CLIENT_ID=$(az ad sp show --id http://$Velero_Storage_Account --query appId --output tsv)

cat << EOF > ./credentials-velero
AZURE_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID
AZURE_TENANT_ID=$AZURE_TENANT_ID
AZURE_CLIENT_ID=$AZURE_CLIENT_ID
AZURE_CLIENT_SECRET=$AZURE_CLIENT_SECRET
AZURE_AKS_RESOURCE_GROUP=$AZURE_AKS_RESOURCE_GROUP"
AZURE_CLOUD_NAME=AzurePublicCloud
EOF

velero install \
--provider azure \
--plugins velero/velero-plugin-for-microsoft-azure:v1.0.1 \
--bucket $Velero_Storage_Account_Container \
--secret-file ./credentials-velero \
--backup-location-config resourceGroup=$Velero_Resource_Group,storageAccount=$Velero_Storage_Account,subscriptionId=$AZURE_SUBSCRIPTION_ID \
--snapshot-location-config apiTimeout=5m,resourceGroup=$Velero_Resource_Group,subscriptionId=$AZURE_SUBSCRIPTION_ID
```
Get all resources in velero namespace 

$ kubectl get all -n velero

Backup & Restore in AKS cluster using Velero without persistent volume
-------------------------------------------------------------------------
Deploy nginx pod and expose it as Load Balancer service
```
kubectl apply -f app/nginx-without-pv.yaml
kubectl get po,svc -n nginx-test

velero backup create nginx-test 
velero get backup
velero describe backup nginx-test 
velero restore create --from-backup nginx-test
```
If want to schedule a backup every 24 hrs then use below command.
```
velero create schedule nginx-test --schedule ="@every 24h"
```

Backup & Restore in AKS cluster using Velero with persistent volume
-------------------------------------------------------------------------
Deploy nginx pod with persistent volume and expose it as Load Balancer service
```
kubectl apply -f app/nginx-with-pv.yaml
kubectl get po,svc,pvc -n nginx-test-pv

velero backup create nginx-pv
velero get backup
velero describe backup nginx-pv 
velero restore create --from-backup nginx-pv
```

Backup hook feature
--------------------
you can specify one or more commands to execute in a container in a pod, when a pod is being backed up.
Can be used in scenarios like freeze the file system to ensure that all pending disk I/O operations have completed prior to taking a snapshot

