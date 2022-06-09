# Nginx Ingress Controller

```bash
RG=myRG
LOCATION=westeurope
ACRNAME=myacr$RANDOM

# create resource group
az group create -n $RG -l $LOCATION

# create Azure container registry (ACR) to pull images from
# although we can pull the images directly from k8s.gcr.io we
# want more control over where we pull from; our AKS cluster
# might have policies that retrict pulling from any registry
az acr create -n $ACRNAME -g $RG --sku Basic

# import ingress-nginx images
SOURCE_REGISTRY=k8s.gcr.io
CONTROLLER_IMAGE=ingress-nginx/controller
CONTROLLER_TAG=v1.1.2
PATCH_IMAGE=ingress-nginx/kube-webhook-certgen
PATCH_TAG=v1.1.1
DEFAULTBACKEND_IMAGE=defaultbackend-amd64
DEFAULTBACKEND_TAG=1.5

# import controller image
az acr import --name $ACRNAME --source $SOURCE_REGISTRY/$CONTROLLER_IMAGE:$CONTROLLER_TAG --image $CONTROLLER_IMAGE:$CONTROLLER_TAG

# import patch image
az acr import --name $ACRNAME --source $SOURCE_REGISTRY/$PATCH_IMAGE:$PATCH_TAG --image $PATCH_IMAGE:$PATCH_TAG

# import default backend image
az acr import --name $ACRNAME --source $SOURCE_REGISTRY/$DEFAULTBACKEND_IMAGE:$DEFAULTBACKEND_TAG --image $DEFAULTBACKEND_IMAGE:$DEFAULTBACKEND_TAG

# list the images in ACR; three repositories should be listed
az acr repository list -n $ACRNAME

# create a Key Vault to hold the Nginx certificate
KVNAME=gebakv$RANDOM
az keyvault create --name $KVNAME --resource-group $RG --enable-rbac-authorization

# grab your user id and subscription id
USER_OBJECT_ID=$(az ad signed-in-user show --query objectId -o tsv)
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# grant yourself access to Key Vault
az role assignment create --assignee-object-id $USER_OBJECT_ID --role "Key Vault Administrator" --scope /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG/providers/Microsoft.KeyVault/vaults/$KVNAME

# create a certificate; it will not have a valid CN but that is ok for testing
az keyvault certificate create --vault-name $KVNAME -n mycert -p "$(az keyvault certificate get-default-policy)"

# if you have a certificate in a .pfx you can use the following command:
# az keyvault certificate import --vault-name $KVNAME -n mycert \
#   -f /path/to/pfx --password password_to_pfx

# get AKS credentials; we use --admin to bypass AAD
AKSNAME=CLUSTERNAME
AKS_RG=rg-aks
az aks get-credentials -n $AKSNAME -g $AKS_RG --admin

# set AKS resource group
MC_RG=MC_....... GET FROM AKS DETAILS ABOVE

# create an IP address in the AKS MC_ resource group
IP=$(az network public-ip create \
                --resource-group $MC_RG \
                --name ip-nginx \
                --location $LOCATION \
                --sku Standard \
                --allocation-method static \
                --query publicIp.ipAddress \
                --output tsv)



# set full name of ACR
ACR_URL=$ACRNAME.azurecr.io

# add nginx repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# create namespace and secretProviderClass
# the Azure Tenant ID is retrieved from the current Azure CLI session
# the user-assigned identity in CSI_ID is the identity from Secret Store
# CSI provider; that identity is in the MC_ group
AZURE_TENANT_ID=$(az account show --query tenantId -o tsv)
CSI_ID=$(az identity show -g $MC_RG --name azurekeyvaultsecretsprovider-$AKSNAME --query principalId -o tsv)

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name:  nginx-ingress
---
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: tls-cert
  namespace: nginx-ingress
spec:
  provider: azure
  secretObjects:
  - secretName: tls-cert
    type: kubernetes.io/tls
    data:
    - objectName: "tls-cert"
      key: tls.key
    - objectName: "tls-cert"
      key: tls.crt
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    userAssignedIdentityID: "$CSI_ID"
    keyvaultName: "$KVNAME"
    objects: |
      array:
        - |
          objectName: "tls-cert"
          objectType: secret
    tenantId: "$AZURE_TENANT_ID"
EOF

# install nginx with helm
# override the default images with the ones from out own ACR

CHART_VERSION=4.0.18
NS=nginx-ingress
REPLICAS=2

helm upgrade nginx-ingress ingress-nginx/ingress-nginx \
    --install \
    --wait \
    --timeout 10m \
    --version $CHART_VERSION \
    --namespace $NS --create-namespace \
    --set controller.replicaCount=$REPLICAS \
    --set controller.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.image.registry=$ACR_URL \
    --set controller.image.image=$CONTROLLER_IMAGE \
    --set controller.image.tag=$CONTROLLER_TAG \
    --set controller.image.digest="" \
    --set controller.admissionWebhooks.patch.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.admissionWebhooks.patch.image.registry=$ACR_URL \
    --set controller.admissionWebhooks.patch.image.image=$PATCH_IMAGE \
    --set controller.admissionWebhooks.patch.image.tag=$PATCH_TAG \
    --set controller.admissionWebhooks.patch.image.digest="" \
    --set controller.service.externalTrafficPolicy=Local \
    --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux \
    --set defaultBackend.image.registry=$ACR_URL \
    --set defaultBackend.image.image=$DEFAULTBACKEND_IMAGE \
    --set defaultBackend.image.tag=$DEFAULTBACKEND_TAG \
    --set defaultBackend.image.digest="" \
    --set controller.service.loadBalancerIP=$IP \
    -f - <<EOF
    controller:
    extraVolumes:
        - name: tls-cert
            csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
                secretProviderClass: "tls-cert"
    extraVolumeMounts:
        - name: tls-cert
            mountPath: "/mnt/tls-cert"
            readOnly: true
    extraArgs:
        default-ssl-certificate: "nginx-ingress/tls-cert"
    EOF

```