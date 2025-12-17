# azure-storage-local-arc
Azure IoT Edge storage module combined with ARC Storage replication
![diagram](/image.png)

Documenting lab work based on idea proposed by Chris Coveyduck.

# Context

## Blob Storage at the Edge

- Azure Local does not support Blob endpoint on-prem.
- IoT Edge module allowed for Blob API to run locally https://hub.docker.com/r/microsoft/azure-blob-storage
- Doc: https://learn.microsoft.com/en-us/previous-versions/azure/iot-edge/how-to-store-data-blob#feedback

## ARC replicate storage

- To replicate the Blob volume to Cloud with native tooling there is Azure Container Storage enabled by Azure Arc
- https://learn.microsoft.com/en-us/azure/azure-arc/container-storage/howto-install-edge-volumes?tabs=single
- CloudIngest volumes allow onprem>Azure replication. (It is also possible to do the opposite in preview if you want onprem Cache for cloud data)

## Setup K3S on Ubuntu VM

Follow this guide - https://learn.microsoft.com/en-us/azure/iot-operations/deploy-iot-ops/howto-prepare-cluster?tabs=ubuntu

## Setup IoT Edge module 

- https://learn.microsoft.com/en-us/previous-versions/azure/iot-edge/how-to-store-data-blob#feedback
- Note old API, use Storage Explorer in Stack mode per guide
- validate container push locally with file

## Setup ARC replicated storage

- https://learn.microsoft.com/en-us/azure/azure-arc/container-storage/howto-install-edge-volumes?tabs=single

## Bring it all together

- Setup local sync between IoT Edge module mount and CloudIngest volume, or mount AACS PVC inside IoT Edge Storage container

## Setup 

## Scratchpad

'''
 az connectedk8s connect --name "k3s" --resource-group "VS-GBB-ER-LAB-NE" --location "northeurope" --correlation-id "c18ab9d0-685e-48e7-ab55-12588447b0ed" --tags "Datacent
er City StateOrDistrict CountryOrRegion"



az k8s-extension create --cluster-name "k3s" --name "k3s-certmgr" --resource-group "VS-GBB-ER-LAB-NE" --cluster-type connectedClusters --extension-type microsoft.iotoperations.platform --scope cluster --release-namespace cert-manager --release-train preview



az k8s-extension create --resource-group "VS-GBB-ER-LAB-NE" --cluster-name "k3s" --cluster-type connectedClusters --name azure-arc-containerstorage --extension-type microsoft.arc.containerstorage





export CLUSTER_NAME=k3s
export RESOURCE_GROUP=VS-GBB-ER-LAB-NE
export EXTENSION_TYPE=${1:-"microsoft.arc.containerstorage"}
az k8s-extension list --cluster-name ${CLUSTER_NAME} --resource-group ${RESOURCE_GROUP} --cluster-type connectedClusters | jq --arg extType ${EXTENSION_TYPE} 'map(select(.extensionType == $extType)) | .[] | .identity.principalId' -r

7f19265c-ffe6-4f52-a3b3-709459765624

STORAGE_ACCOUNT_NAME=cloudbackupk3s
RESOURCE_GROUP=VS-GBB-ER-LAB-NE


PRINCIPAL_ID=7f19265c-ffe6-4f52-a3b3-709459765624
SUBSCRIPTION_ID=d7530696-27ed-4771-9043-9b3a68941716


az role assignment create --assignee $PRINCIPAL_ID --role "Storage Blob Data Owner" --scope /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT_NAME

cloudIngestPVC.yaml


kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  ### Create a name for your PVC ###
  name: cloudingest
  ### Use a namespace that matched your intended consuming pod, or "default" ###
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  storageClassName: cloud-backed-sc




no external storage




apiVersion: "arccontainerstorage.azure.net/v1"
kind: IngestSubvolume
metadata:
  name: subvol
spec:
  edgevolume: cloudingest
  path: ingestSubDir # Don't use a preceding slash
  authentication:
    authType: MANAGED_IDENTITY
  storageAccountEndpoint: "https://cloudbackupk3s.blob.core.windows.net/"
  containerName: backup
  ingest:
    order: newest-first
    minDelaySec: 60
  eviction:
    order: unordered
    minDelaySec: 120
  onDelete: trigger-immediate-ingest












apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudingestsubvol-deployment ### This must be unique for each deployment you choose to create.
spec:
  replicas: 2
  selector:
    matchLabels:
      name: acsa-testclientdeployment
  template:
    metadata:
      name: acsa-testclientdeployment
      labels:
        name: acsa-testclientdeployment
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - acsa-testclientdeployment
            topologyKey: kubernetes.io/hostname
      containers:
        ### Specify the container in which to launch the busy box. ###
        - name: ingest-deployment-container
          image: mcr.microsoft.com/azure-cli:2.57.0@sha256:c7c8a97f2dec87539983f9ded34cd40397986dcbed23ddbb5964a18edae9cd09
          command:
            - "/bin/sh"
            - "-c"
            - "dd if=/dev/urandom of=/data/ingestSubDir/acsaingesttestfile count=16 bs=1M && while true; do ls /data &>/dev/null || break; sleep 1; done"
          volumeMounts:
            ### This name must match the volumes.name attribute below ###
            - name: acsa-volume
              ### This mountPath is where the PVC is attached to the pod's filesystem ###
              mountPath: "/data"
      volumes:
          ### User-defined 'name' that's used to link the volumeMounts. This name must match volumeMounts.name as previously specified. ###
        - name: acsa-volume
          persistentVolumeClaim:
            ### This claimName must refer to your PVC metadata.name (Line 5)
            claimName: cloudingest

'''



