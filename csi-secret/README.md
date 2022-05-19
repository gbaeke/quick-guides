# Secret store CSI Provider for Azure Key Vault

Please see https://blog.baeke.info/2022/05/19/quick-guide-to-the-secret-store-csi-driver-for-azure-key-vault-on-aks/ for more details.

Requirements:
- bash
- Azure CLI and logged in to subscription
- working Kube config and kubectl


Here are all the commands:

```bash
# enable the driver
RG=YOUR_RESOURCE_GROUP
CLUSTER=YOUR_CLUSTER_NAME
 
az aks enable-addons --addons=azure-keyvault-secrets-provider --name=$CLUSTER --resource-group=$RG

# create Key Vault, grant yourself acess and create demo secret
# replace <SOMETHING> with a value like your initials for example
KV=<SOMETHING>$RANDOM
 
# name of the key vault secret
SECRET=demosecret
 
# value of the secret
VALUE=demovalue
 
# create the key vault and turn on Azure RBAC; we will grant a managed identity access to this key vault below
az keyvault create --name $KV --resource-group $RG --location westeurope --enable-rbac-authorization true
 
# get the subscription id
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
 
# get your user object id
USER_OBJECT_ID=$(az ad signed-in-user show --query objectId -o tsv)
 
# grant yourself access to key vault
az role assignment create --assignee-object-id $USER_OBJECT_ID --role "Key Vault Administrator" --scope /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG/providers/Microsoft.KeyVault/vaults/$KV
 
# add a secret to the key vault
az keyvault secret set --vault-name $KV --name $SECRET --value $VALUE


# Grant Azure Key Vault Secrets Provider managed identity access to Key Vault
# grab the managed identity principalId assuming it is in the default
# MC_ group for your cluster and resource group
IDENTITY_ID=$(az identity show -g MC\_$RG\_$CLUSTER\_westeurope --name azurekeyvaultsecretsprovider-$CLUSTER --query principalId -o tsv)
 
# grant access rights on Key Vault; note that the principalId is used and not the clientId
az role assignment create --assignee-object-id $IDENTITY_ID --role "Key Vault Administrator" --scope /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG/providers/Microsoft.KeyVault/vaults/$KV


# create SecretProviderClass in default namespace
# ensure a working kubectl cli
AZURE_TENANT_ID=$(az account show --query tenantId -o tsv)
CLIENT_ID=$(az aks show -g $RG -n $CLUSTER --query addonProfiles.azureKeyvaultSecretsProvider.identity.clientId -o tsv)
 
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: demo-secret
  namespace: default
spec:
  provider: azure
  secretObjects:
  - secretName: demosecret
    type: Opaque
    data:
    - objectName: "demosecret"
      key: demosecret
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    userAssignedIdentityID: "$CLIENT_ID"
    keyvaultName: "$KV"
    objects: |
      array:
        - |
          objectName: "demosecret"
          objectType: secret
    tenantId: "$AZURE_TENANT_ID"
EOF

# create pod that mounts secrets in volume and sets env var in container
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: secretpods
  name: secretpods
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secretpods
  template:
    metadata:
      labels:
        app: secretpods
    spec:
      containers:
      - image: nginx
        name: nginx
        env:
          - name:  demosecret
            valueFrom:
              secretKeyRef:
                name:  demosecret
                key:  demosecret
        volumeMounts:
          - name:  secret-store
            mountPath:  "mnt/secret-store"
            readOnly: true
      volumes:
        - name:  secret-store
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "demo-secret"
EOF

# get a shell to the pod
export POD_NAME=$(kubectl get pods -l "app=secretpods" -o jsonpath="{.items[0].metadata.name}")
 
# if this does not work, check the status of the pod
# if still in ContainerCreating there might be an issue
kubectl exec -it $POD_NAME -- sh
 
# run these commands in the container in the pod
cd /mnt/secret-store
ls # the file containing the secret is listed
cat demosecret; echo # demovalue is revealed
 
# echo the value of the environment variable
echo $demosecret # demovalue is revealed
```