# Save Helm chart to OCI Repository

We will create a default Helm chart from a starter and save it to Azure Container Registry.

To know more about working with Helm and creating Helm charts from scratch, see [helm\README.md](../helm/README.md)

```bash
# create a scratch folder
scratch=scratch$RANDOM
mkdir $scratch; cd $scratch

# create chart based on nginx starter
helm create super-api

# check the template of the chart; use the release name 
# mynginx to avoid the generic release-name in resource names
helm template mynginx super-api

# create a resource group and Azure Container Registry
# enable the admin user on ACR to login with username
# and password
RG=rg-ocitest
LOCATION=westeurope
ACR=acrgeba$RANDOM
az group create --name $RG --location $LOCATION
az acr create --name $ACR -g $RG --sku Basic --admin-enabled

# retrieve ACR password
PASSWORD=$(az acr credential show -n $ACR | jq .passwords[0].value -r)

# login to ACR with Helm
helm registry login $ACR.azurecr.io -u $ACR -p $PASSWORD

# package Helm chart to .tgz
# check that the .tgz is super-api-0.1.0.tgz; if not, adjust the helm push command
helm package super-api

# push the chart
helm push super-api-0.1.0.tgz oci://$ACR.azurecr.io

# check the repos in ACR; does not look different for container repo
az acr repository list -n $ACR

# check the manifest; the output should show the manifest is of 
# type application/vnd.cncf.helm.config.v1+json
# there is only one layer, the .tgz file
az acr manifest show -r $ACR -n super-api:0.1.0


# template the chart directly from ACR; do not specify the version as a tag
# the prefix oci:// is required
helm template mynginx oci://$ACR.azurecr.io/super-api --version=0.1.0

# to install the chart
helm install mynginx oci://$ACR.azurecr.io/super-api --version=0.1.0

# cleanup
cd ..; rm scratch -rf
az group delete --name $RG

```