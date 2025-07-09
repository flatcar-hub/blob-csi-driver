## Driver Parameters
 > parameter names are case-insensitive

<details><summary>required permissions for CSI driver controller</summary>
<pre>
 # To grant permissions for following actions, you need to assign both "Storage Account Contributor" 
 # and "Storage Blob Data Contributor" roles to the CSI driver controller.
Microsoft.Storage/storageAccounts/write
Microsoft.Storage/storageAccounts/read
Microsoft.Storage/storageAccounts/listKeys/action
Microsoft.Storage/storageAccounts/*/delete
Microsoft.Storage/storageAccounts/blobServices/containers/write
Microsoft.Storage/storageAccounts/blobServices/containers/read
Microsoft.Storage/storageAccounts/blobServices/containers/delete
Microsoft.Storage/operations/read
# this is only necessary if the driver creates the storage account with a private endpoint:
Microsoft.Network/virtualNetworks/subnets/write
Microsoft.Network/virtualNetworks/subnets/read
Microsoft.Network/privateEndpoints/write
Microsoft.Network/privateEndpoints/read
Microsoft.Network/privateEndpoints/privateDnsZoneGroups/write
Microsoft.Network/privateDnsZones/write
Microsoft.Network/privateDnsZones/virtualNetworkLinks/write
Microsoft.Network/privateDnsZones/virtualNetworkLinks/read
Microsoft.Network/privateDnsZones/read
Microsoft.Network/privateDnsOperationStatuses/read
Microsoft.Network/locations/operations/read
# this is only necessary if the subnet carrying the write permission has these additional resources configured:
Microsoft.Network/serviceEndpointPolicies/join/action
Microsoft.Network/natGateways/join/action
Microsoft.Network/networkIntentPolicies/join/action
Microsoft.Network/networkSecurityGroups/join/action
Microsoft.Network/routeTables/join/action
Microsoft.Network/networkManagers/ipamPools/associateResourcesToPool/action
</pre>
</details>

### Dynamic Provisioning
  > [blobfuse example](../deploy/example/storageclass-blobfuse.yaml)

  > [nfs example](../deploy/example/storageclass-blob-nfs.yaml)

Name | Meaning | Example | Mandatory | Default value
--- | --- | --- | --- | ---
skuName | Azure storage account type (alias: `storageAccountType`) | `Standard_LRS`, `Premium_LRS`, `Standard_GRS`, `Standard_RAGRS`, `Standard_ZRS`, `Premium_ZRS`  | No | `Standard_LRS`
location | Azure location | `eastus`, `westus`, etc. | No | if empty, driver will use the same location name as current k8s cluster
resourceGroup | Azure resource group name | existing resource group name | No | if empty, driver will use the same resource group name as current k8s cluster
subscriptionID | specify Azure subscription ID in which blob storage directory will be created | Azure subscription ID | No | if not empty, `resourceGroup` must be provided
storageAccount | specify Azure storage account name| STORAGE_ACCOUNT_NAME | No | When a specific storage account name is not provided, the driver will look for a suitable storage account that matches the account settings within the same resource group. If it fails to find a matching storage account, it will create a new one. However, if a storage account name is specified, the storage account must already exist.
protocol | specify blobfuse, blobfuse2 or NFSv3 mount | `fuse`, `fuse2`, `nfs` | No | `fuse`
networkEndpointType | specify network endpoint type for the storage account created by driver. If `privateEndpoint` is specified, a private endpoint will be created for the storage account. For other cases, a service endpoint will be created for `nfs` protocol by default. | "",`privateEndpoint` | No | "",<br>for AKS cluster, make sure cluster Control plane identity (that is, your AKS cluster name) is added to the Contributor role in the resource group hosting the VNet
storageEndpointSuffix | specify Azure storage endpoint suffix | `core.windows.net`, `core.chinacloudapi.cn`, etc | No | if empty, driver will use default storage endpoint suffix according to cloud environment, e.g. `core.windows.net`
containerName | specify the existing container(directory) name | existing container name | No | if empty, driver will create a new container name, starting with `pvc-fuse` for blobfuse or `pvc-nfs` for NFSv3
containerNamePrefix | specify Azure storage directory prefix created by driver | can only contain lowercase letters, numbers, hyphens, and length should be less than 21 | No |
server | specify Azure storage account server address | existing server address, e.g. `accountname.blob.core.chinacloudapi.cn` | No | if empty, driver will use the default Azure storage account server address based on cloud provider config
accessTier | [Access tier for storage account](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview) | Standard account can choose `Hot` or `Cool`, and Premium account can only choose `Premium` | No | empty(use default setting for different storage account types)
allowBlobPublicAccess | Allow or disallow public access to all blobs or containers for storage account created by driver | `true`,`false` | No | `false`
allowSharedKeyAccess | Allow or disallow shared key access for storage account created by driver (only applicable for NFS mount or blobfuse mount with managed identity) | `true`,`false` | No | `true`
requireInfraEncryption | specify whether or not the service applies a secondary layer of encryption with platform managed keys for data at rest for storage account created by driver | `true`,`false` | No | `false`
storageEndpointSuffix | specify Azure storage endpoint suffix | `core.windows.net`, `core.chinacloudapi.cn`, etc | No | if empty, driver will use default storage endpoint suffix according to cloud environment
tags | [tags](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/tag-resources) would be created in newly created storage account | tag format: 'foo=aaa,bar=bbb' | No | ""
matchTags | whether matching tags when driver tries to find a suitable storage account | `true`,`false` | No | `false`
useDataPlaneAPI | specify whether use data plane API for blob container create/delete, this could solve the SRP API throttling issue since data plane API has almost no limit, while it would fail when there is firewall or vnet setting on storage account | `true`,`false` | No | `false`
softDeleteBlobs | Enable [soft delete for blobs](https://learn.microsoft.com/en-us/azure/storage/blobs/soft-delete-blob-overview), specify the days to retain deleted blobs | "7" | No | Soft Delete Blobs is disabled if empty
softDeleteContainers | Enable [soft delete for containers](https://learn.microsoft.com/en-us/azure/storage/blobs/soft-delete-container-overview), specify the days to retain deleted containers | "7" | No | Soft Delete Containers is disabled if empty
enableBlobVersioning | Enable [blob versioning](https://learn.microsoft.com/en-us/azure/storage/blobs/versioning-overview), can't enabled when `protocol` is `nfs` or `isHnsEnabled` is `true` | `true`,`false` | No | versioning for blobs is disabled if empty
--- | **Following parameters are only for blobfuse** | --- | --- |
storeAccountKey | Should the storage account key be stored in a Kubernetes secret <br> (Note:  if set to `false`, the driver will use the kubelet identity to obtain the account key) | `true`,`false` | No | `true`
getLatestAccountKey | whether getting the latest account key based on the creation time, this driver would get the first key by default | `true`,`false` | No | `false`
secretName | specify secret name to store account key | | No |
secretNamespace | specify the namespace of secret to store account key | `default`,`kube-system`, etc | No | pvc namespace
isHnsEnabled | enable `Hierarchical namespace` for Azure DataLake storage account | `true`,`false` | No | `false`
--- | **Following parameters are only for NFS protocol** | --- | --- |
mountPermissions | mounted folder permissions. The default is `0777`, if set as `0`, driver will not perform `chmod` after mount | `0777` | No |
fsGroupChangePolicy | indicates how volume's ownership will be changed by the driver, pod `securityContext.fsGroupChangePolicy` is ignored  | `OnRootMismatch`(by default), `Always`, `None` | No | `OnRootMismatch`
--- | **Following parameters are only for vnet setting** | --- | --- |
vnetResourceGroup | specify vnet resource group where virtual network is | existing resource group name | No | if empty, driver will use the `vnetResourceGroup` value in azure cloud config file
vnetName | virtual network name | existing virtual network name | No | if empty, driver will use the `vnetName` value in azure cloud config file
subnetName | subnet name | existing subnet name(s) of the agent node, if you want to update service endpoints on multiple subnets, separate them using a comma (`,`) | No | if empty, driver will update all the subnets under the cluster virtual network
vnetLinkName | virtual network link name associated with private dns zone |  | No | if empty, driver will use the `vnetName + "-vnetlink"` by default
publicNetworkAccess | `PublicNetworkAccess` property of created storage account by the driver | `Enabled`, `Disabled`, `SecuredByPerimeter` | No |

 - account tags format created by dynamic provisioning
```
k8s-azure-created-by: azure
```

 - file share name format created by dynamic provisioning(example)
```
pvc-92a4d7f2-f23b-4904-bad4-2cbfcff6e388
```

 - VolumeID(`volumeHandle`) is the identifier for the volume handled by the driver, format of VolumeID: `rg#accountName#containerName#uuid#secretNamespace#subscriptionID`
 > `uuid`, `secretNamespace`, `subscriptionID` are optional

### Static Provisioning(bring your own storage container)
  > [blobfuse example](../deploy/example/pv-blobfuse-csi.yaml)

  > [nfs example](../deploy/example/pv-blobfuse-nfs.yaml)

  > [blobfuse read account key or SAS token from key vault example](../deploy/example/pv-blobfuse-csi-keyvault.yaml)

  > [blobfuse Managed Identity and Service Principal Name auth example](../deploy/example/pv-blobfuse-auth.yaml)

Name | Meaning | Available Value | Mandatory | Default value
--- | --- | --- | --- | ---
volumeHandle | Specify a value the driver can use to uniquely identify the storage blob container in the cluster. | A recommended way to produce a unique value is to combine the globally unique storage account name and container name: {account-name}_{container-name}.| Yes |
volumeAttributes.subscriptionID | specify Azure subscription ID where blob storage directory is located | Azure subscription ID | No | if not empty, `resourceGroup` must be provided
volumeAttributes.resourceGroup | Azure resource group name | existing resource group name | No | if empty, driver will use the same resource group name as current k8s cluster
volumeAttributes.storageAccount | existing storage account name | existing storage account name | Yes |
volumeAttributes.containerName | existing container name | existing container name | Yes |
volumeAttributes.protocol | specify blobfuse, blobfuse2 or NFSv3 mount (blobfuse2 is still in Preview) | `fuse`, `fuse2`, `nfs` | No | `fuse`
volumeAttributes.server | specify Azure storage account server address | existing server address, e.g. `accountname.blob.core.windows.net` | No | if empty, driver will use default `accountname.blob.core.windows.net` or other sovereign cloud account address
volumeAttributes.storageEndpointSuffix | specify Azure storage endpoint suffix | `core.windows.net`, `core.chinacloudapi.cn`, etc | No | if empty, driver will use default storage endpoint suffix according to cloud environment
--- | **Following parameters are only for blobfuse** | --- | --- |
volumeAttributes.secretName | secret name that stores storage account name and key(only applies for SMB) | | No |
volumeAttributes.secretNamespace | secret namespace | `default`,`kube-system`, etc | No | pvc namespace
volumeAttributes.getLatestAccountKey | whether getting the latest account key based on the creation time, this driver would get the first key by default | `true`,`false` | No | `false`
nodeStageSecretRef.name | secret name that stores(check below examples):<br>`azurestorageaccountkey`<br>`azurestorageaccountsastoken`<br>`msisecret`<br>`azurestoragespnclientsecret` | existing Kubernetes secret name |  No  |
nodeStageSecretRef.namespace | secret namespace | k8s namespace  |  Yes  |
--- | **Following parameters are only for NFS protocol** | --- | --- |
volumeAttributes.mountPermissions | mounted folder permissions | `0777` | No |
volumeAttributes.fsGroupChangePolicy | indicates how volume's ownership will be changed by the driver, pod `securityContext.fsGroupChangePolicy` is ignored  | `OnRootMismatch`(by default), `Always`, `None` | No | `OnRootMismatch`
--- | **Following parameters are only for feature: blobfuse [Managed Identity and Service Principal Name auth](https://github.com/Azure/azure-storage-fuse?tab=readme-ov-file#environment-variables)** | --- | --- |
volumeAttributes.AzureStorageAuthType | Authentication Type | `Key`, `SAS`, `MSI`, `SPN` | No | `Key`
volumeAttributes.AzureStorageIdentityClientID | Identity Client ID |  | No |
volumeAttributes.AzureStorageIdentityObjectID | Identity Object ID (deprecated) |  | No |
volumeAttributes.AzureStorageIdentityResourceID | Identity Resource ID |  | No |
volumeAttributes.MSIEndpoint | MSI Endpoint |  | No |
volumeAttributes.AzureStorageSPNClientID | SPN Client ID |  | No |
volumeAttributes.AzureStorageSPNTenantID | SPN Tenant ID |  | No |
volumeAttributes.AzureStorageAADEndpoint | AADEndpoint |  | No |
--- | **Following parameters are only for feature: blobfuse [Workload Identity auth](../docs/workload-identity-static-pv-mount.md)** | --- | --- |
volumeAttributes.ClientID | clientid of the managed identity assigned to the storage account |  | No |
volumeAttributes.mountWithWorkloadIdentityToken (Preview) | indicates whether performing blobfuse mount with workload identity token | `true`,`false` | No | `false` (limitation: the workload identity token would expire after 24 hours, make sure the blobfuse volume would be remounted by your application before it expires)
--- | **Following parameters are only for feature: blobfuse read account key or SAS token from key vault** | --- | --- |
volumeAttributes.keyVaultURL | Azure Key Vault DNS name | existing Azure Key Vault DNS name | No |
volumeAttributes.keyVaultSecretName | Azure Key Vault secret name | existing Azure Key Vault secret name | No |
volumeAttributes.keyVaultSecretVersion | Azure Key Vault secret version | existing version | No |if empty, driver will use `current version`

 - create a Kubernetes secret for `nodeStageSecretRef.name`
 ```console
kubectl create secret generic azure-secret --from-literal=azurestorageaccountname="xxx" --from-literal azurestorageaccountkey="xxx" --type=Opaque
kubectl create secret generic azure-secret --from-literal=azurestorageaccountname="xxx" --from-literal azurestorageaccountsastoken="xxx" --type=Opaque
kubectl create secret generic azure-secret --from-literal msisecret="xxx" --type=Opaque
# azurestoragespnclientid, azurestoragespntenantid field setting in secret is only supported from v1.21.3
kubectl create secret generic azure-secret --from-literal azurestoragespnclientsecret="xxx" azurestoragespnclientid="xxx" azurestoragespntenantid="xxx" --type=Opaque
 ```

### Tips
 - mounting blobfuse requires account key, if `nodeStageSecretRef` field is not provided in PV config, azure file driver would try to get `azure-storage-account-{accountname}-secret` in the pod namespace first, if that secret does not exist, it would get account key by Azure storage account API directly using kubelet identity (make sure kubelet identity has reader access to the storage account).
 > If you have recently rotated the account key, it is important to update the account key stored in the Kubernetes secret. Additionally, the application pods that reference the Azure blob volume should be restarted after the secret has been updated. In cases where two pods share the same PVC on the same node, it is necessary to reschedule the pods to a different node without that PVC mounted to ensure that remounting occurs successfully. To safely rotate the account key without experiencing downtime, you can follow the steps outlined [here](https://github.com/kubernetes-sigs/azurefile-csi-driver/issues/1218#issuecomment-1851996062).
 - mounting blob storage NFSv3 does not need account key, NFS mount access is configured by following setting:
    - `Firewalls and virtual networks`: select `Enabled from selected virtual networks and IP addresses` with same vnet as agent node
 - `securityContext.fsGroup` setting
    - blobfuse volume mount does not respect `securityContext.fsGroup`. Instead, you could utilize `-o gid=1000` in `mountOptions` to establish ownership, check [here](https://github.com/Azure/azure-storage-fuse/tree/blobfuse-1.4.5#mount-options) for more mountoptions.
    - when there is a large number of files inside an NFS volume, the process of setting volume ownership can slow down the NFS volume mount when `securityContext.fsGroup` is different from group ownership of volume. By configuring `fsGroupChangePolicy: None` in the `parameters` of storage class or persistent volume, you can bypass the volume ownership setting step, resulting in faster NFS volume mounts.
 - To support an [Azure DataLake storage account](https://docs.microsoft.com/en-us/azure/storage/blobs/upgrade-to-data-lake-storage-gen2-how-to) when using blobfuse mount, you'll need to do the following:
   - To create an ADLS account using the driver in dynamic provisioning, specify `isHnsEnabled: "true"` in the storage class parameters.
   - To enable blobfuse access to an ADLS account in static provisioning, specify the mount option `--use-adls=true` in the persistent volume.
 - blobfuse cache(`--tmp-path` [mount option](https://github.com/Azure/azure-storage-fuse/tree/blobfuse-1.4.5#mount-options))
   - By default, the blobfuse cache is located in the `/mnt` directory. If the VM SKU provides a temporary disk, the `/mnt` directory is mounted on the temporary disk. However, if the VM SKU does not provide a temporary disk, the `/mnt` directory is mounted on the OS disk. 
   - with blobfuse-proxy deployment (default on AKS), user could set `--tmp-path=` mount option to specify a different cache directory
 - [Mount Azure blob storage with managed identity](../deploy/example/blobfuse-mi)
 - [Blobfuse Performance and caching](https://github.com/Azure/azure-storage-fuse?tab=readme-ov-file#frequently-asked-questions)
 - [Blobfuse CLI Flag Options v1 & v2](https://github.com/Azure/azure-storage-fuse/blob/main/MIGRATION.md#blobfuse-cli-flag-options)

#### `containerName` parameter supports following pv/pvc metadata conversion
> if `containerName` value contains following strings, it would be converted into corresponding pv/pvc name or namespace
 - `${pvc.metadata.name}`
 - `${pvc.metadata.namespace}`
 - `${pv.metadata.name}`

#### [Storage considerations for Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/scenarios/app-platform/aks/storage)
#### [Compare access to Azure Files, Blob Storage, and Azure NetApp Files with NFS](https://learn.microsoft.com/en-us/azure/storage/common/nfs-comparison#comparison)
