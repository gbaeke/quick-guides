# Workload identity

Please see https://blog.baeke.info/2022/05/18/quick-guide-to-kubernetes-workload-identity-on-aks/ for more details.

Requirements:
- bash
- Azure CLI and logged in to subscription
- working Kube config and kubectl
- helm
- azwi: `brew install Azure/azure-workload-identity/azwi`

Here are all the commands:

```bash
# install oidc issuer
CLUSTER=<AKS_CLUSTER_NAME>
RG=<AKS_CLUSTER_RESOURCE_GROUP>
 
az aks update -n $CLUSTER -g $RG --enable-oidc-issuer

# add workload identity webhook
AZURE_TENANT_ID=$(az account show --query tenantId -o tsv)
  
helm repo add azure-workload-identity https://azure.github.io/azure-workload-identity/charts
  
helm repo update
  
helm install workload-identity-webhook azure-workload-identity/workload-identity-webhook \
   --namespace azure-workload-identity-system \
   --create-namespace \
   --set azureTenantID="${AZURE_TENANT_ID}"

# create Azure AD application
APPLICATION_NAME=WorkloadDemo
azwi serviceaccount create phase app --aad-application-name $APPLICATION_NAME

# get application app id and grant Reader on Azure CLI subscription
APPLICATION_CLIENT_ID="$(az ad sp list --display-name $APPLICATION_NAME --query '[0].appId' -otsv)"

SUBSCRIPTION_ID=$(az account show --query id -o tsv)
 
az role assignment create --assignee-object-id $APPLICATION_CLIENT_ID --role "Reader" --scope /subscriptions/$SUBSCRIPTION_ID

# Create Kubernetes service account to use with workload identity
SERVICE_ACCOUNT_NAME=sademo
SERVICE_ACCOUNT_NAMESPACE=default
 
azwi serviceaccount create phase sa \
  --aad-application-name "$APPLICATION_NAME" \
  --service-account-namespace "$SERVICE_ACCOUNT_NAMESPACE" \
  --service-account-name "$SERVICE_ACCOUNT_NAME"

# Configure federation on the AAD app 
azwi serviceaccount create phase federated-identity \
  --aad-application-name "$APPLICATION_NAME" \
  --service-account-namespace "$SERVICE_ACCOUNT_NAMESPACE" \
  --service-account-name "$SERVICE_ACCOUNT_NAME" \
  --service-account-issuer-url "$ISSUER_URL"


# create pods that can access the federation token
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
      serviceAccount: sademo
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
 
kubectl exec -it $POD_NAME -- bash

# run inside the container
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
```