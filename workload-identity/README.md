# Workload identity

⚠️ **Warning:** the post https://blog.baeke.info/2022/05/18/quick-guide-to-kubernetes-workload-identity-on-aks/ which describes the steps below is outdated. The steps below have been updated to reflect the latest changes in the Workload Identity for AKS.

Some things that have changed:
- the steps below use managed identities instead of app registrations
- you do not need to install the webhook
- the `azwi` CLI is not required

Requirements:
- bash
- Azure CLI and logged in to subscription
- kubectl

Here are all the commands:

```bash
# update Azure CLI aks-preview extension
az extension update --name aks-preview

# register preview feature
az feature register \
  --namespace "Microsoft.ContainerService" --name "EnableWorkloadIdentityPreview"

# check if registered
az feature list -o table \
  --query "[?contains(name, 'Microsoft.ContainerService/EnableWorkloadIdentityPreview')].{Name:name,State:properties.state}"

# if registered then continue
az provider register --namespace Microsoft.ContainerService

# deploy AKS with oidc enabled
CLUSTER=kub-oidc
RG=rg-aks-oidc
LOCATION=westeurope
IDENTITY=id-aks-oidc
 
az group create -n $RG -l $LOCATION

az aks create -g $RG -n $CLUSTER --node-count 1 --enable-oidc-issuer \
  --enable-workload-identity --generate-ssh-keys

# get cluster credentials; requires kubectl
az aks get-credentials -g $RG -n $CLUSTER --admin

# get OIDC issuer URL
export AKS_OIDC_ISSUER="$(az aks show -n $CLUSTER -g $RG --query "oidcIssuerProfile.issuerUrl" -otsv)"

# get current subscription id
export SUBSCRIPTION_ID="$(az account show --query "id" -otsv)"

# create user assigned managed identity
az identity create --name $IDENTITY --resource-group $RG \
  --location $LOCATION --subscription $SUBSCRIPTION_ID

# get identity client id
export USER_ASSIGNED_CLIENT_ID="$(az identity show -n $IDENTITY -g $RG --query "clientId" -otsv)"

# grant identity reader role on subscription; we will later use 
# this identity and use az group list to verify if it works
az role assignment create --role "Reader" \
  --assignee $USER_ASSIGNED_CLIENT_ID --subscription $SUBSCRIPTION_ID

# create Kubernetes service account
SANAME=sademo

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: ${USER_ASSIGNED_CLIENT_ID}
  labels:
    azure.workload.identity/use: "true"
  name: $SANAME
  namespace: default
EOF

# create federated credentials; check the managed identity in the portal 
# to see the federated credentials settings
az identity federated-credential create --name fic-sademo --identity-name $IDENTITY \
  --resource-group $RG --issuer ${AKS_OIDC_ISSUER} \
  --subject system:serviceaccount:default:$SANAME

# create pod that can access the federation token
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azcli-deployment
  namespace: default
  labels:
    app: azcli
spec:
  replicas: 1
  selector:
    matchLabels:
      app: azcli
  template:
    metadata:
      labels:
        app: azcli
    spec:
      # needs to refer to service account used with federation
      serviceAccount: $SANAME
      containers:
        - name: azcli
          image: mcr.microsoft.com/azure-cli:latest
          command:
            - "/bin/bash"
            - "-c"
            - "sleep infinity"
EOF

# Get pod name and get a shell to container in pod
POD_NAME=$(kubectl get pods -l "app=azcli" -o jsonpath="{.items[0].metadata.name}")
 
# get a shell to the container
kubectl exec -it $POD_NAME -- bash

# run this inside the container; the webhook created the 
# environment variables below
echo $AZURE_CLIENT_ID
echo $AZURE_TENANT_ID
echo $AZURE_FEDERATED_TOKEN_FILE
cat $AZURE_FEDERATED_TOKEN_FILE
echo $AZURE_AUTHORITY_HOST
 
# list the standard Kubernetes service account secrets
cd /var/run/secrets/kubernetes.io/serviceaccount
ls 
 
# check the folder containing the AZURE_FEDERATED_TOKEN_FILE
cd /var/run/secrets/azure/tokens
ls
 
# you can use the AZURE_FEDERATED_TOKEN_FILE with the Azure CLI
# together with $AZURE_CLIENT_ID and $AZURE_TENANT_ID
# a password is not required since we are doing federated token exchange
 
az login --federated-token "$(cat $AZURE_FEDERATED_TOKEN_FILE)" \
--service-principal -u $AZURE_CLIENT_ID -t $AZURE_TENANT_ID
 
# list resource groups
az group list

# this should fail
az group create -n test -l westeurope

# exit the container
exit
```