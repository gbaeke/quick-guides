# Quick guide to Kubernetes Service Accounts

Please see https://youtu.be/keoYFZhtg0U for a video that explains Kubernetes service accounts.

Requirements:
- working Kube config and kubectl
- bash

Here are the commands:

```bash
# create service account in the default namespace
kubectl create serviceaccount sa-demo --namespace=default

# check the YAML of the service account to see what was created
kubectl get sa sa-demo -o yaml

# the YAML references a secret, check the secret (replace hash with actual value)
kubectl get secret sa-demo-token-<hash> -o yaml

# retrieve the JWT token from the secret
# TIP: go to https://jwt.io and paste the token to see the decoded result
kubectl get secret sa-demo-token-<hash> -o jsonpath='{.data.token}' | base64 -d


# use the token in a pod; first we create an nginx deployment
kubectl create deploy nginx --image=nginx

# patch the deployment to use the service account
kubectl patch deploy nginx -p '{"spec":{"template":{"spec":{"serviceAccount":"sa-demo"}}}}'

# check the YAML of the deployment and check for serviceAccount
kubectl get deploy nginx -o yaml

# exec into the container of an nginx pod
export POD_NAME=$(kubectl get pods -l "app=nginx" -o jsonpath="{.items[0].metadata.name}")
kubectl exec -it $POD_NAME -- sh

# inside the pod, issue the following commands
cd /var/run/secrets/kubernetes.io/serviceaccount
ls # the list you see are the secrets mounted from the token
cat token # JWT token we retrieved earlier; can be submitted to auth to K8S API
cat ca.crt # certificate to verify requests to https://kubernetes (see later)
cat namespace # just prints the namespace

# use curl to talk to the Kubernetes API using the kubernetes service in each namespace
# we try to list pods
curl -H "Authorization: Bearer $(cat token)" --cacert ca.crt https://kubernetes/api/v1/namespaces/default/pods

# the above command will fail because the service account does not have access
# open another terminal on your computer and leave the shell to the pod open
# we will add a role and rolebinding
kubectl create role pod-reader --verb=list --resource=pods
kubectl create rolebinding pod-reader --role=pod-reader --serviceaccount=default:sa-demo

# in the shell to the pod, run the below command again, pods will be listed
curl -H "Authorization: Bearer $(cat token)" --cacert ca.crt https://kubernetes/api/v1/namespaces/default/pods
```

If this worked, you have successfully created a service account and associated it with a pod. Containers in the pod can use the mounted secrets to authenticate and talk to the Kubernetes API server.