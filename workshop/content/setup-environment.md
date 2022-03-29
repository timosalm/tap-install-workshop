##### Prerequisites

All required tools for the installation are already installed! 

Set your Tanzu Network username and password in the following ENV variables in the terminal, ...
```terminal:input
text: export TANZU_NET_USER=
endl: false
```
```terminal:input
text: export TANZU_NET_PASSWORD=
endl: false
```
... the hostname, username, and password for the target container registry ...
```terminal:input
text: export CONTAINER_REGISTRY_HOSTNAME=
endl: false
```
```terminal:input
text: export CONTAINER_REGISTRY_USERNAME=
endl: false
```
```terminal:input
text: export CONTAINER_REGISTRY_PASSWORD=
endl: false
```
..., and your ingress domain (e.g. "tap.example.com").
```terminal:input
text: export TAP_INGRESS_DOMAIN=
endl: false
```

Setup a project/registry for the examples of this workshop with the name "tap-install-workshop-examples" in your container registry. 

Login to pivnet CLI. The api token you need for the login can be fetched from the ["Edit Profile page"](https://network.tanzu.vmware.com/users/dashboard/edit-profile) of the Tanzu Network after you logged in.
```terminal:input
text: pivnet login --api-token=
endl: false
```

For this workshop you should already have a cluster provisioned due to the time it takes. 
The prerequisites for the cluster can be found in the documentation here:
```dashboard:open-url
url: {{ ENV_TAP_PRODUCT_DOCS_BASE_URL }}/GUID-prerequisites.html
```

##### Importing the kubeconfig of the target cluster and setting the context

You can export the kubeconfig of your **Azure AKS** cluster via
```copy
az aks get-credentials --resource-group ${CLUSTER_NAME} --name ${CLUSTER_NAME} -f kubeconfig.yaml
```

You can export the kubeconfig of your **AWS EKS** cluster via
```copy
aws eks update-kubeconfig --name ${CLUSTER_NAME} --kubeconfig kubeconfig.yaml
```
The kubeconfig is using the aws CLI, so you have to login via `aws configure` in the workshop environment!

You can export the kubeconfig of your **GKE** cluster via
```copy
KUBECONFIG=kubeconfig.yaml gcloud container clusters get-credentials ${CLUSTER_NAME} --region $REGION
```

**Hint:** If you want to do that in this environment, the az, aws, and gcloud CLIs are installed!

After you have exported your kubeconfig, you can easily add, import it, and set the context via
```terminal:execute
command: vim kubeconfig.yaml
clear: true
```
```execute
kubectl konfig import -s kubeconfig.yaml
kubectl ctx
```
Rename your context to `tap-install`
```copy
kubectl config rename-context <old-name> tap-install
```
Set the default namespace to `default` in the context.
```execute
kubectl config set-context tap-install --namespace=default
```