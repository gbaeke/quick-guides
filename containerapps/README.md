# Quick guide to Azure Container Apps

This guide looks at Azure Container apps (ACA). You will do the following:
- Create an ACA environment linked to a Log Analytics workspace
- Create two container apps: they run in the environment
- The container apps use Dapr:
    - front-end container app uses service invocation to call the back-end
    - back-end uses a Cosmos DB Dapr component to write to Cosmos DB (SQL API)
- Use `az containerapp up` to demonstrate the dev inner loop experience

Requirements:
- All commands run from bash in WSL 2
- Azure CLI and logged in to subscription
- ACA extension for Azure CLI: run  `az extension add --name containerapp --upgrade`
- Microsoft.App namespace registered: run `az provider register --namespace Microsoft.App`
- If you have never used Log Analytics, also register Microsoft.OperationalInsights: run `az provider register --namespace Microsoft.OperationalInsights`
- jq, curl, sed, git

Here are the commands:

```bash
RG=rg-aca
LOCATION=westeurope
ENVNAME=env-aca
LA=la-aca # log analytics workspace name
 
# create the resource group
az group create --name $RG --location $LOCATION
 
# create the log analytics workspace
az monitor log-analytics workspace create \
  --resource-group $RG \
  --workspace-name $LA
 
# retrieve workspace ID and secret
LA_ID=`az monitor log-analytics workspace show --query customerId -g $RG -n $LA -o tsv | tr -d '[:space:]'`
 
LA_SECRET=`az monitor log-analytics workspace get-shared-keys --query primarySharedKey -g $RG -n $LA -o tsv | tr -d '[:space:]'`
 
# check workspace ID and secret; if empty, something went wrong
# in previous two steps
echo $LA_ID
echo $LA_SECRET
 
# create the ACA environment; no integration with a virtual network
az containerapp env create \
  --name $ENVNAME \
  --resource-group $RG\
  --logs-workspace-id $LA_ID \
  --logs-workspace-key $LA_SECRET \
  --location $LOCATION \
  --tags env=test owner=geert
 
# check the ACA environment
az containerapp env list -o table

# create the front-end app
APPNAME=frontend
DAPRID=frontend # could be different
IMAGE="ghcr.io/gbaeke/super:1.0.5" # image to deploy
PORT=8080 # port that the container accepts requests on
 
# create the container app and make it available on the internet
# with --ingress external; the envoy proxy used by container apps
# will proxy incoming requests to port 8080
 
az containerapp create --name $APPNAME --resource-group $RG \
--environment $ENVNAME --image $IMAGE \
--min-replicas 0 --max-replicas 5 --enable-dapr \
--dapr-app-id $DAPRID --target-port $PORT --ingress external
 
# check the app
az containerapp list -g $RG -o table
 
# grab the resource id of the container app
APPID=$(az containerapp list -g $RG | jq .[].id -r)
 
# show the app via its id
az containerapp show --ids $APPID
 
# because the app has an ingress type of external, it has an FQDN
# let's grab the FQDN (fully qualified domain name)
FQDN=$(az containerapp show --ids $APPID | jq .properties.configuration.ingress.fqdn -r)
 
# curl the URL; it should return "Hello from Super API"
curl https://$FQDN
 
# container apps work with revisions; you are now at revision 1
az containerapp revision list -g $RG -n $APPNAME -o table
 
# let's deploy a newer version
IMAGE="ghcr.io/gbaeke/super:1.0.7"
 
# use update to change the image
# you could also run the create command again (same as above but image will be newer)
az containerapp update -g $RG --ids $APPID --image $IMAGE
 
# look at the revisions again; the new revision uses the new
# image and 100% of traffic
# NOTE: in the portal you would only see the last revision because
# by default, single revision mode is used; switch to multiple 
# revision mode and check "Show inactive revisions"
 
az containerapp revision list -g $RG -n $APPNAME -o table

# deploy Cosmos DB
uniqueId=$RANDOM
LOCATION=useast # changed because of capacity issues in westeurope at the time of writing
 
# create the account; will take some time
az cosmosdb create \
  --name aca-$uniqueId \
  --resource-group $RG \
  --locations regionName=$LOCATION \
  --default-consistency-level Strong
 
# create the database
az cosmosdb sql database create \
  -a aca-$uniqueId \
  -g $RG \
  -n aca-db
 
# create the collection; the partition key is set to a 
# field in the document called partitionKey; Dapr uses the
# document id as the partition key
az cosmosdb sql container create \
  -a aca-$uniqueId \
  -g $RG \
  -d aca-db \
  -n statestore \
  -p '/partitionKey' \
  --throughput 400

# deploy the back-end
# grab the Cosmos DB documentEndpoint
ENDPOINT=$(az cosmosdb list -g $RG | jq .[0].documentEndpoint -r)
 
# grab the Cosmos DB primary key
KEY=$(az cosmosdb keys list -g $RG -n aca-$uniqueId | jq .primaryMasterKey -r)
 
# update variables, IMAGE and PORT are the same
APPNAME=backend
DAPRID=backend # could be different
 
# create the Cosmos DB component file
# it uses the ENDPOINT above + database name + collection name
# IMPORTANT: scopes is required so that you can scope components
# to the container apps that use them
 
cat << EOF > cosmosdb.yaml
componentType: state.azure.cosmosdb
version: v1
metadata:
- name: url
  value: "$ENDPOINT"
- name: masterkey
  secretRef: cosmoskey
- name: database
  value: aca-db
- name: collection
  value: statestore
secrets:
- name: cosmoskey
  value: "$KEY"
scopes:
- $DAPRID
EOF
 
# create Dapr component at the environment level
# this used to be at the container app level
az containerapp env dapr-component set \
    --name $ENVNAME --resource-group $RG \
    --dapr-component-name cosmosdb \
    --yaml cosmosdb.yaml
 
# create the container app; the app needs an environment 
# variable STATESTORE with a value that is equal to the 
# dapr-component-name used above
# ingress is internal; there is no need to connect to the backend from the internet
 
az containerapp create --name $APPNAME --resource-group $RG \
--environment $ENVNAME --image $IMAGE \
--min-replicas 1 --max-replicas 1 --enable-dapr \
--dapr-app-port $PORT --dapr-app-id $DAPRID \
--target-port $PORT --ingress internal \
--env-vars STATESTORE=cosmosdb

# verify connectivity
# create some string data to send
STRINGDATA="'$(jq --null-input --arg appId "backend" --arg method "savestate" --arg httpMethod "POST" --arg payload '{"key": "mykey", "data": "123"}' '{"appId": $appId, "method": $method, "httpMethod": $httpMethod, "payload": $payload}' -c -r)'"
 
# check the string data (double quotes should be escaped in payload)
# payload should be a string and not JSON, hence the quoting
echo $STRINGDATA
 
# call the front end to save some data
# in Cosmos DB data explorer, look for a document with id 
# backend||mykey; content is base64 encoded because 
# the data is not json
 
echo curl -X POST -d $STRINGDATA https://$FQDN/call | bash
 
# create some real JSON data to save; now we need to escape the
# double quotes and jq will add extra escapes
JSONDATA="'$(jq --null-input --arg appId "backend" --arg method "savestate" --arg httpMethod "POST" --arg payload '{"key": "myjson", "data": "{\"name\": \"geert\"}"}' '{"appId": $appId, "method": $method, "httpMethod": $httpMethod, "payload": $payload}' -c -r)'"
 
# call the front end to save the data
# look for a document id backend||myjson; data is json
 
echo curl -v -X POST -d $JSONDATA https://$FQDN/call | bash

# check the logs
# check frontend logs
az containerapp logs show -n frontend -g $RG
 
# I want to see the dapr logs of the container app
az containerapp logs show -n frontend -g $RG --container daprd
 
# if you do not see log entries about our earlier calls, save data again
# the log stream does not show all logs; log analytics contains more log data
echo curl -v -X POST -d $JSONDATA https://$FQDN/call | bash
 
# now let's check the logs again but show more earlier logs and follow
# there should be an entry method with custom content; that's the
# result of saving the JSON data
 
az containerapp logs show -n frontend -g $RG --tail 300 --follow


# use az containerapp up
# clone the super-api repo and cd into it
git clone https://github.com/gbaeke/super-api.git && cd super-api
 
# checkout the quickguide branch
git checkout quickguide
 
# bring up the app; container build will take some time
az containerapp up -n super-api --source . --ingress external --target-port 8080 --environment env-aca
 
# list apps; super-api has been added with a new external Fqdn
az containerapp list -g $RG -o table
 
# check ACR in the resource group
az acr list -g $RG -o table
 
# grab the ACR name
ACR=$(az acr list -g $RG | jq .[0].name -r)
 
# list repositories
az acr repository list --name $ACR
 
# more details about the repository
az acr repository show --name $ACR --repository super-api
 
# show tags; az containerapp up uses numbers based on date and time
az acr repository show-tags --name $ACR --repository super-api
 
# make a small change to the code; ensure you are still in the
# root of the cloned repo; instead of Hello from Super API we
# will say Hi from Super API when curl hits the /
sed -i s/Hello/Hi/g cmd/app/main.go
 
# run az containerapp up again; a new container image will be
# built and pushed to ACR and deployed to the container app
az containerapp up -n super-api --source . --ingress external --target-port 8080 --environment env-aca
 
# check the image tags; there are two
az acr repository show-tags --name $ACR --repository super-api
 
# curl the endpoint; should say "Hi from Super API"
curl https://$(az containerapp show -g $RG -n super-api | jq .properties.configuration.ingress.fqdn -r)
```