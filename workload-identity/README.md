# Workload identity

Please see https://blog.baeke.info/2022/05/18/quick-guide-to-kubernetes-workload-identity-on-aks/ for more details.

Requirements:
- bash
- Azure CLI and logged in to subscription
- working Kube config and kubectl

Here are all the commands:

```bash
# update Azure CLI aks-preview extension
az extension update --name aks-preview

# register preview feature
az feature register --namespace "Microsoft.ContainerService" --name "EnableWorkloadIdentityPreview"

# check if registered
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/EnableWorkloadIdentityPreview')].{Name:name,State:properties.state}"

# if registered then continue
az provider register --namespace Microsoft.ContainerService

# install oidc issuer
CLUSTER=kub-oidc
RG=rg-aks-oidc
 
az aks create -g $RG -n $CLUSTER --node-count 1 --enable-oidc-issuer --enable-workload-identity --generate-ssh-keys

# get OIDC issuer URL
export AKS_OIDC_ISSUER="$(az aks show -n $CLUSTER -g $RG --query "oidcIssuerProfile.issuerUrl" -otsv)"

# get current subscription id

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