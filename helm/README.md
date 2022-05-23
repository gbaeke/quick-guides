# Helm ⛵

This quick guide has two parts:

- Using Helm
- Creating a Helm chart (👉 TODO)

## Using Helm

Requirements:
- bash
- Helm v3 (`brew install helm`)
- kubectl (and working kube config)

```bash
# Check helm version; mine was 3.9.x
helm version

# Helm charts live in repositories; add the repository for super-api
# Find useful repositories over at https://artifacthub.io/
helm repo add super-api https://gbaeke.github.io/helm-chart/

# List local repositories; super-api should be listed
helm repo list

# Check .config folder in $HOME
cd $HOME/.config/helm
cat repositories.yaml # to show file where repositories are saved

# Check .cache folder
cd $HOME/.cache/helm/repository
ls # shows cached repository indices 

# Go back to home folder
cd $HOME

# Search the repository
helm search repo super-api

# The repository has one chart; it has a chart version, an appVersion and a description
# example: super-api/super-api 1.0.0   1.0.3  A Helm chart for super-api
# by convention, app version is the version of the app the chart installs by default
# Let's install the chart and call the release myapp; install in a namespace and create it
# if it does not exist

helm install myapp super-api/super-api --namespace super-api --create-namespace

# list Helm installations in namespace
helm list --namespace super-api

# check if pods were deployed; should result in one pod
# the name of the pod includes myapp (release name) and super-api (chart name)
# the names of deployments & pods depend on how the chart was created
kubectl get pods --namespace super-api

# run helm test (-n is the same as --namespace)
# you can run this command because the super-api chart has a test defined
# the test should return success
helm test myapp -n super-api

# check pods again; there should be a completed pod that helm used to perform the test
# the test simply ran a curl/wget command to verify that super-api returns a response
kubectl get pods --namespace super-api

# Let's install version 1.0.7 of the application  by setting a user-defined value
# We use helm upgrade instead of helm install
# --install to install the chart if the chart was not installed yet
# --set is used to set the value of image.tag to 1.0.7; the chart was created to use that value as the image tag

helm upgrade --install myapp super-api/super-api --namespace super-api --set image.tag=1.0.7

# run helm list again; we are now at revision 2 of myapp
helm list -n super-api

# get more information about the revisions with helm history
# you will see that revision two is the deployed one
# do not be misled by the APP VERSION; that is the default set in the chart and has no relation
# with the actual installed version (1.0.7)
helm history myapp -n super-api

# get information about the latest release; first get ALL information about the release
# ALL INFORMATION = metadata, user-supplied values, computed values, the manifest, hooks, notes.txt 
helm get all myapp -n super-api

# to only get user supplied values
helm get values myapp -n super-api

# to only get the manifest that is generated from the templates in the chart we can use helm get manifest
# the chart developer decides which templates to include and how to fill them with
# computed values; computed values are the combination of default values & user supplied values
helm get manifest  myapp -n super-api

# to only get notes; notes are defined by the chart developer by including a notes.txt in the templates
# and optionally show computed values in the notes (.e.g. deployment name, pod names, ...)
helm get notes myapp -n super-api

# get information about a specific revision
# e.g., get values for the first revision which results in null (no values supplied)
helm get values myapp -n super-api --revision 1

# rollback to revision 1
helm rollback myapp 1 -n super-api

# helm history will now show 3 revisions; the 3rd revision is the rollback to 1
helm history myapp -n super-api

# helm keeps history information as secrets in the namespace
# run the command below to see three secrets of type helm.sh/release.v1
kubectl get secrets -n super-api

# let's uninstall the chart but keep the history (by default, history is removed)
helm uninstall myapp -n super-api --keep-history

# history will still show 3 releases (the secrets are still in the namespace)
helm history myapp -n super-api

# nothing will be shown if you run helm list
helm list -n super-api

# but you can still rollback
helm rollback myapp 2 -n super-api

# helm history will show revision 3 uninstalled and revision 4 as rollback to 3
# note: if you do not use --keep-history during uninstall you cannot do a rollback afterwards
helm history myapp -n super-api

# what if you want to check the chart before installing?
helm pull super-api/super-api

# the above command only downloads the .tgz archive; Helm charts are packaged that way
# let's also untar the file
helm pull super-api/super-api --untar

# a folder with name super-api is now in your current folder
# checkout the files in the folder to see the contents of the chart
# you can actually install the chart from the folder
helm upgrade --install myapp super-api  # super-api is the folder containing the chart

# or use
cd super-api
helm upgrade --install myapp . # use the chart in the current folder

# to check the chart for errors or warnings
helm lint

# to check the manifest that the chart generates
helm template .

# to check the template with a user-defined value
helm template . --set image.tag=HELLO | grep HELLO  # will show the line that sets image
helm template . --set replicaCount=10 autoscaling.enabled=false # replicas in deployment only set when autoscaling disabled

# setting a lot of user-defined values with --set is annoying; we can create a values file in the current folder
cat << EOF > myvalues.yaml
replicaCount: 10
autoscaling:
  enabled: false
EOF

# now use the values file; we will test with helm template but this will work with install and upgrade as well
helm template . --values myvalues.yaml | grep replicas # should show 10

# by default, Helm does not wait until your deployments and pods are ready
# to let Helm wait, use the following command:
helm upgrade --install myapp super-api/super-api --namespace super-api --wait

```

## Creating a Helm chart 🔨

Although you can create a Helm chart with a starter, we will create one from scratch. To use the default starter, use `helm create chartname`. That creates a directory `chartname` in your current directory.

A chart is just a bunch of folders and files that we will create manually. In this example, we build a simple chart for super-api. The application's container image is at ghcr.io/gbaeke/super. There are multiple tags available.

Requirements:
- bash 
- helm (>=3.8)
- sed

```bash
# go to your $HOME folder
cd $HOME

# create a mycharts folder and cd into it
mkdir mycharts; cd mycharts

# create a directory to hold your chart and cd into it
mkdir super-api; cd super-api

# create a templates folder; this folder will hold templates for Kubernetes objects
# such as deployments, services, etc...
mkdir templates

# create a Chart.yaml value for the chart metadata such as chart version, app version, description, and more...
cat << EOF > Chart.yaml
apiVersion: v2
name: superapi
description: A Helm chart to install superapi

# optional
# kubeVersion: ">=1.20"

type: application
version: "1.0.0"
appVersion: "1.0.7"
EOF

# when Chart.yaml is created you can already run helm lint
# it will provide some information but no should not list errors
helm lint

# create a file values.yaml to hold default values
# these values, combined with user-defined values that may overwrite the defaults
# will result in the final set of computed values; the computer values are sent to
# your templates in the templates folder to be rendered and can be found in .Values
cat << EOF > values.yaml
replicaCount: 1

image:
  repository: ghcr.io/gbaeke/super
  pullPolicy: IfNotPresent
  tag: ""
EOF

# the above properties can have any name; you refer to these values later in your templates
# now cd into templates
cd templates

# create deployment.yaml
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name:  superapi-deployment
  labels:
    app: superapi
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  selector:
    matchLabels:
      app: superapi
  template:
    metadata:
      labels:
        app:  superapi
    spec:
      restartPolicy: Always
      containers:
      - image: ghcr.io/gbaeke/super:1.0.7
        imagePullPolicy: Always
        name:  superapi
        resources:
          limits:
            cpu: "20m"
            memory: "55M"          
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 90
          timeoutSeconds: 10
        readinessProbe:
          httpGet:
            path: /readyz
            port: 8080
          initialDelaySeconds: 30
          timeoutSeconds: 10
        env:
        - name:  WELCOME
          value: "Hello from superapi deployed by Helm"
        - name: PORT
          value: "8080"      
        ports:
        - containerPort:  8080
          name:  http
EOF

# Create service.yaml
cat <<EOF > service.yaml
kind: Service
apiVersion: v1
metadata:
  name:  superapi-svc
spec:
  selector:
    app:  superapi
  type:  ClusterIP
  ports:
  - name:  http
    port:  80
    targetPort:  8080
EOF

# run helm template to see the manifest that Helm renders from the templates
# in the templates folder; of course, the templates we just created do not
# contain any code to pickup any of our user supplied values

helm template ../   # need to go one level higher than templates

# in deployment.yaml, on line 23, let's use the values from the values.yaml and Chart.yaml file
sed -i 's/ghcr.io\/gbaeke\/super:1.0.7/"{{.Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"/' deployment.yaml

# check the line that contains image:
# the output should be:
# - image: "{{.Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
# this uses the image repo and image tag from values.yaml; in image.tag is empty than use appVersion defined in Chart.yaml
cat deployment.yaml | grep image:

# run helm template again and grep for image:
# image should be: "ghcr.io/gbaeke/super:1.0.7"
helm template ../ | grep image:

# check with user supplied value
helm template ../ --set image.tag=HELLO | grep image:

# add some resources yaml to values.yaml

cat <<EOF >> ../values.yaml

resources:
  limits:
    cpu: "20m"
    memory: "50M"
EOF

# modify deployment.yaml to include the resources YAML block from values.yaml
# remove line 27 to 29 (only resources: is kept)
sed -i '27,29d' deployment.yaml

# replace resources: block
# toYaml is a function that converts the internal implementation of .Values.resources to YAML
# the - in the beginning removes whitespace and the previous newline so we add a newline and indent
# by 12 spaces (n in nindent does the newline)

sed -i 's/resources:/resources:\n            {{- toYaml .Values.resources | nindent 12 }}/' deployment.yaml

# in deployment.yaml you should now find
# resources:
#   {{- toYaml .Values.resources | nindent 12 }}

# run helm template again with another cpu limit and check if the limit is set to 50m
# if it is, the chart picks use the YAML under resources: in values.yaml
# however, you replace the cpu limit with --set resulting in the final computed values 

helm template ../ --set resources.limits.cpu="50m"

# let's add some default values for use with a ConfigMap
cat <<EOF >> ../values.yaml
config:
  enabled: true
  welcome: "Welcome to Super API v1"
EOF

# now add a ConfigMap
cat <<EOF > configmap.yaml
{{- if .Values.config.enabled -}}
kind: ConfigMap
apiVersion: v1
metadata:
  name: superapi-config
  namespace: default
data:
  welcome: {{ .Values.config.welcome | upper | quote }} # Welcome message
{{- end -}}
EOF

# above, the ConfigMap is only rendered when config.enabled=true (default)
# the welcome field is set to .Values.config.welcome converted to uppercase and quoted with ""
# run helm template with default values: the ConfigMap should be in the output
helm template ../ | grep ConfigMap

# run helm template with config.enabled=false; the output does not show kind:ConfigMap because it is not in the output
helm template ../ --set config.enabled=false | grep ConfigMap

# install the chart from the file system; a chart does not have to come from a registry
cd ..
helm install chart-demo . 

# the chart should have been installed to your current namespace (usually default)
# use helm list to check
helm list

# so see all installed Helm charts in all namespaces
helm list --all-namespaces

# add a NOTES.txt
cat << EOF > templates/NOTES.txt

Congratulations, you have installed app version {{ .Values.image.tag | default .Chart.AppVersion }}!

EOF

# upgrade the chart to see notes.txt
helm upgrade chart-demo .
