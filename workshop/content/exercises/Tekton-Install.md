Tekton is a cloud-native, open-source framework for creating CI/CD systems. It allows developers to build, test, and deploy across cloud providers and on-premise systems.
```dashboard:open-url
url: {{ ENV_TAP_PRODUCT_DOCS_BASE_URL }}/GUID-tekton-install-tekton.html
```

Let's first install Tekton with the following commands and then have a closer look at the problems it solves and how it works.
To install Tekton, we first have to remove the package from the `excluded_packages`.
```editor:select-matching-text
file: tap-values.yml
text: "  - tekton.tanzu.vmware.com"
```
```editor:replace-text-selection
file: tap-values.yml
text: ""
```

Because there are currently now configuration options available, we can directly update our profile installation.
```terminal:execute
command: tanzu package installed update tap -p tap.tanzu.vmware.com -v 1.0.2 --values-file tap-values.yml -n tap-install --kubeconfig kubeconfig.yaml
clear: true
```