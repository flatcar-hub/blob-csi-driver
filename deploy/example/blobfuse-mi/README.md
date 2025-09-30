# Mount Azure blob storage with managed identity

This article demonstrates the process of utilizing blobfuse mount with user-assigned managed identity.
> you could leverage the built-in user assigned managed identity(kubelet identity) bound to the AKS agent node pool(with naming rule [`AKS Cluster Name-agentpool`](https://docs.microsoft.com/en-us/azure/aks/use-managed-identity#summary-of-managed-identities)), if you have created your own managed identity, make sure the managed identity is bound to the agent node pool.

## Before you begin
 - Make sure the managed identity assigned the `Storage Blob Data Contributor` role for the storage account
 > here is an example that uses Azure CLI commands to assign the `Storage Blob Data Contributor` role to the managed identity for the storage account. If the storage account is created by the driver(dynamic provisioning), then you need to grant `Storage Blob Data Contributor` role on the resource group where the storage account is located

```bash
mid="$(az identity list -g "$resourcegroup" --query "[?name == 'managedIdentityName'].principalId" -o tsv)"
said="$(az storage account list -g "$resourcegroup" --query "[?name == '$storageaccountname'].id" -o tsv)"
az role assignment create --assignee-object-id "$mid" --role "Storage Blob Data Contributor" --scope "$said"
```

 - Retrieve the clientID of managed identity.
 > If you are using kubelet identity, the identity will be named `{aks-cluster-name}-agentpool` and located in the node resource group.
```bash
AzureStorageIdentityClientID=`az identity list -g "$resourcegroup" --query "[?name == '$identityname'].clientId" -o tsv`
```
    
## Dynamic Provisioning
- Ensure that the system-assigned identity of your cluster control plane has been assigned the `Storage Blob Data Contributor` role for the storage account.
 > if the storage account is created by the driver, then you need to grant `Storage Blob Data Contributor` role on the resource group where the storage account is located

1. Create a storage class
    ```yml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: blob-fuse
    provisioner: blob.csi.azure.com
    parameters:
      skuName: Premium_LRS 
      protocol: fuse
      resourceGroup: EXISTING_RESOURCE_GROUP_NAME   # optional, node resource group by default if it's not provided
      storageAccount: EXISTING_STORAGE_ACCOUNT_NAME # optional, a new account will be created if it's not provided
      containerName: EXISTING_CONTAINER_NAME  # optional, a new container will be created if it's not provided
      AzureStorageAuthType: MSI
      AzureStorageIdentityClientID: "xxxxx-xxxx-xxx-xxx-xxxxxxx"
    reclaimPolicy: Delete
    volumeBindingMode: Immediate
    allowVolumeExpansion: true
    mountOptions:
      - -o allow_other
      - --file-cache-timeout-in-seconds=120
      - --use-attr-cache=true
      - --cancel-list-on-mount-seconds=10  # prevent billing charges on mounting
      - -o attr_timeout=120
      - -o entry_timeout=120
      - -o negative_timeout=120
      - --log-level=LOG_WARNING  # LOG_WARNING, LOG_INFO, LOG_DEBUG
      - --cache-size-mb=1000  # Default will be 80% of available memory, eviction will happen beyond that.
    ```

1. create a statefulset with blobfuse volume mount
```bash
kubectl create -f https://raw.githubusercontent.com/kubernetes-sigs/blob-csi-driver/master/deploy/example/statefulset.yaml
```

## Static Provisioning

> bring your own storage account and blob container

1. create PV with specified account name, blob container and AzureStorageIdentityClientID
    ```yml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: pv-blob
    spec:
      capacity:
        storage: 100Gi
      accessModes:
        - ReadWriteMany
      persistentVolumeReclaimPolicy: Retain  # If set as "Delete" container would be removed after pvc deletion
      storageClassName: blob-fuse
      mountOptions:
        - -o allow_other
        - --file-cache-timeout-in-seconds=120
      csi:
        driver: blob.csi.azure.com
        # make sure this volumeid is unique in the cluster
        volumeHandle: "{resource-group-name}#{account-name}#{container-name}"
        volumeAttributes:
          protocol: fuse
          resourceGroup: EXISTING_RESOURCE_GROUP_NAME   # optional, node resource group if it's not provided
          storageAccount: EXISTING_STORAGE_ACCOUNT_NAME
          containerName: EXISTING_CONTAINER_NAME
          AzureStorageAuthType: MSI
          AzureStorageIdentityClientID: "xxxxx-xxxx-xxx-xxx-xxxxxxx"
    ```

1. create a pvc and a deployment with blobfuse volume mount
    ```console
    kubectl create -f https://raw.githubusercontent.com/kubernetes-sigs/blob-csi-driver/master/deploy/example/deployment.yaml
    ```
