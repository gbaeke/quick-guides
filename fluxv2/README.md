# Flux v2 on AKS

## Requirements

You need the following to run the commands:

- An Azure subscription with a deployed AKS cluster; a single node will do
- Azure CLI and logged in to the subscription with owner access
- All commands run in bash, in my case in WSL 2.0 on Windows 11
- kubectl and a working kube config (use az aks get-credentials)

## Register AKS Extension Manager

```bash
# register the feature
az feature register --namespace Microsoft.ContainerService --name AKS-ExtensionManager
 
# after a while, check if the feature is registered
# the command below should return "state": "Registered"
az feature show --namespace Microsoft.ContainerService --name AKS-ExtensionManager | grep Registered
 
# ensure you run Azure CLI 2.15 or later
# the command will show the version; mine showed 2.36.0
az version | grep '"azure-cli"'
 
# register the following providers; if these providers are already
# registered, it is safe to run the commands again
 
az provider register --namespace Microsoft.Kubernetes
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.KubernetesConfiguration
 
# enable CLI extensions or upgrade if there is a newer version
az extension add -n k8s-configuration --upgrade
az extension add -n k8s-extension --upgrade
 
# check your Azure CLI extensions
az extension list -o table
```

## Install Flux v2

```bash
RG=rg-aks
CLUSTER=clu-pub
 
# list installed extensions
az k8s-extension list -g $RG -c $CLUSTER -t managedClusters
 
# install flux; note that the name (-n) is a name you choose for
# the extension instance; the command will take some time
# this extension will be installed with cluster-wide scope
 
az k8s-extension create -g $RG -c $CLUSTER -n flux --extension-type microsoft.flux -t managedClusters --auto-upgrade-minor-version true
 
# list Kubernetes namespaces; there should be a flux-system namespace
kubectl get ns
 
# get pods in the flux-system namespace
kubectl get pods -n flux-system
```

## Create a Flux configuration

⚠️ If you want to create a configuration **FROM THE GROUND UP** skip this section. The next section creates a new public git repo allowing us to add our own Flux configuration. The one in this section, uses a Microsoft sample repo.

```bash
# create the configuration; this will take some time
az k8s-configuration flux create -g $RG -c $CLUSTER \
  -n cluster-config --namespace cluster-config -t managedClusters \
  --scope cluster -u https://github.com/gbaeke/gitops-flux2-quick-guide \
  --branch main  \
  --kustomization name=infra path=./infrastructure prune=true \
  --kustomization name=apps path=./apps/staging prune=true dependsOn=["infra"]
 
# check namespaces; there should be a cluster-config namespace
kubectl get ns
 
# check the configuration that was created in the cluster-config namespace
# this is a resource of type FluxConfig
# in the spec, you will find a gitRepository and two kustomizations
 
kubectl get fluxconfigs cluster-config -o yaml -n cluster-config
 
# the Microsoft flux controllers create the git repository source
# and the two kustomizations based on the flux config created above
# they also report status back to Azure
 
# check the git repository; this is a resource of kind GitRepository
# the Flux source controller uses the information in this
# resource to download the git repo locally
 
kubectl get gitrepo cluster-config -o yaml -n cluster-config
 
# check the kustomizations
# the infra kustomization uses folder ./infrastructure in the
# git repository to install redis and nginx with Helm charts
# this kustomization creates other Flux resources such as
# Helm repos and Helm Releases; the Helm Releases are used
# to install nginx and redis with their respective Helm
# charts
 
kubectl get kustomizations cluster-config-infra -o yaml -n cluster-config
 
# the app kustomization depends on infra and uses the ./apps
# folder in the repo to install the podinfo application via
# a kustomize overlay (staging)
 
kubectl get kustomizations cluster-config-apps -o yaml -n cluster-config
```

## Create a Flux configuration from scratch

Extra requirements:
- a GitHub account
- `gh` cli and logged in to your account
- `flux` cli
- Install the CLIs with `brew`
    - `brew install gh`
    - `brew install fluxcd/tap/flux`

```bash
# run flux version; it show both the client and server versions
# for example: source-controller v0.24.2
flux version

# create a random scratch folder and cd into it
SUFFIX=$RANDOM
mkdir scratch$SUFFIX; cd scratch$SUFFIX

# use github CLI to create a public repo and clone it
# change REPO if needed
REPO=flux-quick
gh repo create $REPO --public --clone; cd $REPO

# add the infra folder
mkdir infra

# create a Helm repository source in the folder
flux create source helm redis \
   --url=https://charts.bitnami.com/bitnami \
   --interval=10m --export > infra/redis-chart.yaml

# look at the created file; not that it uses namespace flux-system
cat infra/redis-chart.yaml

# later, we will create the Flux configuration in a namespace called cluster-config
# let's create the redis chart again, this time using the cluster-config namespace
# instead of flux-system
flux create source helm redis \
   --url=https://charts.bitnami.com/bitnami --namespace cluster-config \
   --interval=10m --export > infra/redis-chart.yaml

# now we create a HelmRelease, which will tell FLux how to install the chart
# from the source we just defined
flux create helmrelease redis \
  --source=HelmRepository/redis \
  --chart=redis \
  --release-name=redis \
  --namespace=cluster-config \
  --target-namespace=redis \
  --create-target-namespace=true \
  --export > infra/redis-release.yaml

# check the release yaml
cat infra/redis-release.yaml

# let's create a Flux configuration now; first grab the remote origin
REMOTE=$(git remote get-url origin)

# create the configuration; gh command created a master branch
az k8s-configuration flux create -g $RG -c $CLUSTER \
  -n cluster-config --namespace cluster-config -t managedClusters \
  --scope cluster -u $REMOTE \
  --branch master  \
  --kustomization name=infra path=./infra prune=true

# this should result in errors because we did not push changes to git
flux get kustomizations -n cluster-config

# in the flux configuration, you should see errors as well like ArtifactFailed
az k8s-configuration flux show -g $RG -c $CLUSTER -n cluster-config -t managedClusters

# let's push changes to GitHub
git add .
git commit -m "flux config"
git push --set-upstream origin master
git status # should show you are up-to date

# let's speed things up; the flux configuration created a git source that points to our repo
# the source is in the cluster-config folder
flux reconcile source git cluster-config -n cluster-config
flux reconcile kustomization cluster-config-infra -n cluster-config

# it can take a while before the helm release is created because the default Redis chart
# creates a Redis cluster with 3 replicas
# run the command below and rerun until it says Release reconcilation succeeded
flux get hr -n cluster-config

# you have not created the following in the cluster-config namespace
# - a GitRepository resource: points to the repo we created with gh CLI
# - a Kustomization: installs all YAML found in the ./infra folder of our repo
# - a HelmRepository resource: tells Flux where to find Bitnami charts
# - a HelmRelease resource: tells Flux how to install the Redis chart
# the first two resources where created by az k8s-configuration create
# the last two were created from our git repo
# the commands below show these resources at the Kubernetes level
kubectl get gitrepository -n cluster-config
kubectl get kustomization -n cluster-config
kubectl get helmrepository -n cluster-config
kubectl get helmrelease -n cluster-config

# we now want to add an application that is dependent on Redis
# the app we will install does not actually need Redis; let's just pretend
# make sure you are still in the root of the local flux-quick repo and create an apps folder
mkdir apps

# add another HelmRepositoryk; this one is from https://github.com/gbaeke/helm-chart
flux create source helm super-api \
   --url=https://gbaeke.github.io/helm-chart/ \
   --namespace cluster-config \
   --interval=10m --export > apps/super-api-chart.yaml

# check the YAML
cat apps/super-api-chart.yaml

# create a values.yaml
cat << EOF > apps/values.yaml
replicaCount: 3
image:
  tag: "1.0.7"
EOF

# add another Helm release
flux create helmrelease super-api \
  --source=HelmRepository/super-api \
  --chart=super-api \
  --release-name=super-api \
  --namespace=cluster-config \
  --target-namespace=super-api \
  --create-target-namespace=true \
  --values=apps/values.yaml \
  --export > apps/super-api-release.yaml

# check the YAML; the contents of values.yaml has been included
cat apps/super-api-release.yaml

# push the files to git
git add .
git commit -m "flux config"
git push --set-upstream origin master
git status # should show you are up-to date

# the helm chart will not be installed; there is no kustomization that
# uses the apps path; let's fix that and make the apps kustomization
# dependent on infra
az k8s-configuration flux update -g $RG -c $CLUSTER \
  -n cluster-config -t managedClusters \
  --kustomization name=apps path=./apps prune=true dependsOn=["infra"]

# notice that we also commited values.yaml to the apps folder
# the kustomization will try to apply that yaml to your cluster and that
# will result in an error: failed to decode Kubernetes YAML
# let's remove that file from the repo
rm apps/values.yaml
git add .
git commit -m "remove values"
git push --set-upstream origin master
git status # should show you are up-to date

# tell Flux to pull changes from git NOW! :-)
flux reconcile source git cluster-config -n cluster-config

# reconsile the kustomization as well
flux reconcile kustomization cluster-config-apps -n cluster-config
flux reconcile kustomization cluster-config-infra -n cluster-config


# check if the apps kustomization is ready; this can take some time
# it will take a while for the Azure Portal to show all config objects as well
flux get kustomization cluster-config-apps -n cluster-config
```
