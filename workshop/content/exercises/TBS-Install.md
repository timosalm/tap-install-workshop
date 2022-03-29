VMware Tanzu Build Service (TBS) automates container creation, management, and governance at enterprise scale.
```dashboard:open-url
url: {{ ENV_TAP_PRODUCT_DOCS_BASE_URL }}/GUID-tanzu-build-service-install-tbs.html
```

Let's first install TBS with the following commands and then have a closer look at the problems it solves and how it works.
To install TBS, we first have to remove the package from the `excluded_packages`.
```editor:select-matching-text
file: tap-values.yml
text: "  - buildservice.tanzu.vmware.com"
```
```editor:replace-text-selection
file: tap-values.yml
text: ""
```

We can see the available configuration options via
```execute
tanzu package available list buildservice.tanzu.vmware.com -n tap-install --kubeconfig kubeconfig.yaml
tanzu package available get buildservice.tanzu.vmware.com/1.4.3 --values-schema -n tap-install --kubeconfig kubeconfig.yaml
```

Let's now add the required configuration at the end of the file.
```execute
cat << EOF >> tap-values.yml

buildservice:
  kp_default_repository: ${CONTAINER_REGISTRY_HOSTNAME}/tap/build-service
  kp_default_repository_username: $CONTAINER_REGISTRY_USERNAME
  kp_default_repository_password: $CONTAINER_REGISTRY_PASSWORD
  tanzunet_username: $TANZU_NET_USER
  tanzunet_password: $TANZU_NET_PASSWORD
  enable_automatic_dependency_updates: true
  descriptor_name: tap-1.0.0-full
EOF
```
```editor:open-file
file: tap-values.yml
line: 1
```

Before we install TBS via an update to our profile installation, you have to **configure the target repository "/tap/build-service" in your container registry** of choice.

After that, we can update our profile installation.
```terminal:execute
command: tanzu package installed update tap -p tap.tanzu.vmware.com -v 1.0.2 --values-file tap-values.yml -n tap-install  --kubeconfig kubeconfig.yaml
clear: true
```