# Workload identity

This quick guide starts from an AKS cluster deployed with workload identity. To quickly create such a cluster, run the commands below:

```
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
```

Installation steps for Azure Service Operator v2:

```
# install certmgr
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.8.2/cert-manager.yaml

# get azure tenant id
AZURE_TENANT_ID=$(az account show --query tenantId --output tsv)
AZURE_SUBSCRIPTION_ID=$(az account show --query id --output tsv)

# create service principal with contributor on your subscription
az ad sp create-for-rbac -n azure-service-operator --role owner \
    --scopes /subscriptions/$AZURE_SUBSCRIPTION_ID
```

The last command outputs an appId and a password. Save the appId to a variable called AZURE_CLIENT_ID and the password to AZURE_CLIENT_SECRET:

```
AZURE_CLIENT_ID=<your-client-id>
AZURE_CLIENT_SECRET=<your-client-secret>
```

Now continue with the commands below. Cut and paste them in the same terminal window:

```
# add the helm repository for azure service operator v2
helm repo add aso2 https://raw.githubusercontent.com/Azure/azure-service-operator/main/v2/charts

helm upgrade --install --devel aso2 aso2/azure-service-operator \
     --create-namespace \
     --namespace=azureserviceoperator-system \
     --set azureSubscriptionID=$AZURE_SUBSCRIPTION_ID \
     --set azureTenantID=$AZURE_TENANT_ID \
     --set azureClientID=$AZURE_CLIENT_ID \
     --set azureClientSecret=$AZURE_CLIENT_SECRET
```

Azure Service Operator v2 is now installed. To check if it works, let's create a resource group:

```
cat <<EOF | kubectl apply -f -
apiVersion: resources.azure.com/v1alpha1api20200601
kind: ResourceGroup
metadata:
  name: aso-rg
  namespace: default
spec:
  location: westeurope
EOF

# check if the group was created (you might need to re-run this until it shows up)
az group show --name aso-rg
```


Let's create a managed identity and run a pod with that identity:

```
cat <<EOF | kubectl apply -f -
apiVersion: managedidentity.azure.com/v1beta20181130
kind: UserAssignedIdentity
metadata:
  name: id-azcli
  namespace: default
spec:
  location: westeurope
  owner:
    name: aso-rg
  operatorSpec:
    configMaps:
      principalId:
        name: identity-settings
        key: principalId
      clientId:
        name: identity-settings
        key: clientId
EOF

CLIENT_ID=$(kubectl get configmap identity-settings -o jsonpath='{.data.clientId}')


cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: $CLIENT_ID
  labels:
    azure.workload.identity/use: "true"
  name: id-azcli
  namespace: default
EOF

# retrieve OIDC ISSUER
AKS_OIDC_ISSUER="$(az aks show -n $CLUSTER -g $RG --query "oidcIssuerProfile.issuerUrl" -otsv)"

RG=aso-rg

# we cannot set federated credential via service operator
az identity federated-credential create --name fic-sademo --identity-name id-azcli \
  --resource-group $RG --issuer ${AKS_OIDC_ISSUER} \
  --subject system:serviceaccount:default:id-azcli

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
      serviceAccount: id-azcli
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

# run this in the container
echo $AZURE_CLIENT_ID
echo $AZURE_TENANT_ID
echo $AZURE_FEDERATED_TOKEN_FILE
cat $AZURE_FEDERATED_TOKEN_FILE
echo $AZURE_AUTHORITY_HOST

# try to login
az login --federated-token "$(cat $AZURE_FEDERATED_TOKEN_FILE)" \
--service-principal -u $AZURE_CLIENT_ID -t $AZURE_TENANT_ID

# you should see an error, the account has no rights on the resource group we created
# exit the pod

exit

# assign role
cat <<EOF | kubectl apply -f -
apiVersion: authorization.azure.com/v1beta20200801preview
kind: RoleAssignment
metadata:
  name: fee8b6b1-fe6e-481d-8330-0e950e4e6b86
  namespace: default
spec:
  owner:
    name: aso-rg
    group: resources.azure.com
    kind: ResourceGroup
  principalIdFromConfig:
    name: identity-settings
    key: principalId
  roleDefinitionReference:
    armId: /subscriptions/00000000-0000-0000-0000-000000000000/providers/Microsoft.Authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c
EOF

# get a shell to the container again
kubectl exec -it $POD_NAME -- bash

# try to login, this should work
az login --federated-token "$(cat $AZURE_FEDERATED_TOKEN_FILE)" \
    --service-principal -u $AZURE_CLIENT_ID -t $AZURE_TENANT_ID

# get resource group info, if this works the pod uses federated identity to auth to Azure
az group show -n aso-rg

# exit the pod

exit
```