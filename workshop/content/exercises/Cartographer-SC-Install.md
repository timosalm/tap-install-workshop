```dashboard:open-url
url: {{ ENV_TAP_PRODUCT_DOCS_BASE_URL }}/GUID-scc-install-scc.html
```

To install Cartographer and all other components required for the ootb basic supply chain, we first have to remove the following packages from the **excluded_packages**.
```execute
sed -i '/cartographer.\|ootb-delivery-basic.\|ootb-supply-chain-basic.\|fluxcd.source.controller.\|controller.conventions.apps.\|developer-conventions.\|controller.source.apps.\|spring-boot-conventions.\|ootb-templates./d' tap-values.yml
```
```editor:open-file
file: tap-values.yml
line: 1
```

We can see the available and for our basic configuration relevant options via
```execute
tanzu package available list ootb-supply-chain-basic.tanzu.vmware.com -n tap-install --kubeconfig kubeconfig.yaml
tanzu package available get ootb-supply-chain-basic.tanzu.vmware.com/0.6.1 --values-schema -n tap-install --kubeconfig kubeconfig.yaml
```

Let's now add the required configuration at the end of the file.
{% raw %}
```execute
cat << EOF >> tap-values.yml

supply_chain: basic

ootb_supply_chain_basic:
  registry:
    server: $CONTAINER_REGISTRY_HOSTNAME
    repository: tap-wkld
  gitops:
    ssh_secret: ""
EOF
```
{% endraw %}
```editor:open-file
file: tap-values.yml
line: 1
```

After that, we can update our profile installation.
```terminal:execute
command: tanzu package installed update tap -p tap.tanzu.vmware.com -v 1.0.2 --values-file tap-values.yml -n tap-install --kubeconfig kubeconfig.yaml

clear: true
```

Additionally, we have to setup a developer namespace by adding credentials for the target container registry and Role-Based Access Control (RBAC) rules to it.

```terminal:execute
command: |
  kubectl create ns dev-space
  tanzu secret registry add registry-credentials --username ${CONTAINER_REGISTRY_USERNAME} --password ${CONTAINER_REGISTRY_PASSWORD} --server ${CONTAINER_REGISTRY_HOSTNAME} --namespace dev-space --kubeconfig kubeconfig.yaml

clear: true
```
```execute
cat <<EOF | kubectl -n dev-space apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: tap-registry
  annotations:
    secretgen.carvel.dev/image-pull-secret: ""
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: e30K
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
secrets:
  - name: registry-credentials
imagePullSecrets:
  - name: registry-credentials
  - name: tap-registry
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: default
rules:
- apiGroups: [source.toolkit.fluxcd.io]
  resources: [gitrepositories]
  verbs: ['*']
- apiGroups: [source.apps.tanzu.vmware.com]
  resources: [imagerepositories]
  verbs: ['*']
- apiGroups: [carto.run]
  resources: [deliverables, runnables]
  verbs: ['*']
- apiGroups: [kpack.io]
  resources: [images]
  verbs: ['*']
- apiGroups: [conventions.apps.tanzu.vmware.com]
  resources: [podintents]
  verbs: ['*']
- apiGroups: [""]
  resources: ['configmaps']
  verbs: ['*']
- apiGroups: [""]
  resources: ['pods']
  verbs: ['list']
- apiGroups: [tekton.dev]
  resources: [taskruns, pipelineruns]
  verbs: ['*']
- apiGroups: [tekton.dev]
  resources: [pipelines]
  verbs: ['list']
- apiGroups: [kappctrl.k14s.io]
  resources: [apps]
  verbs: ['*']
- apiGroups: [serving.knative.dev]
  resources: ['services']
  verbs: ['*']
- apiGroups: [servicebinding.io]
  resources: ['servicebindings']
  verbs: ['*']
- apiGroups: [services.apps.tanzu.vmware.com]
  resources: ['resourceclaims']
  verbs: ['*']
- apiGroups: [scanning.apps.tanzu.vmware.com]
  resources: ['imagescans', 'sourcescans']
  verbs: ['*']
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: default
subjects:
  - kind: ServiceAccount
    name: default
EOF
```