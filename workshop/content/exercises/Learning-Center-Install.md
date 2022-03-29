Learning Center provides a platform for creating and self-hosting workshops.The UI can embed slide content, an integrated development environment (IDE), a web console for accessing the Kubernetes cluster, and other custom web applications.

```dashboard:open-url
url: {{ ENV_TAP_PRODUCT_DOCS_BASE_URL }}/GUID-learning-center-install-learning-center.html
```

Let's first install Learning Center and an example workshop with the following commands and then have a closer look how it works.
We first have to remove the packages from the `excluded_packages`.
```editor:select-matching-text
file: tap-values.yml
text: "  - learningcenter.tanzu.vmware.com"
after: 1
```
```editor:replace-text-selection
file: tap-values.yml
text: ""
```
```editor:select-matching-text
file: tap-values.yml
text: "excluded_packages:"
```
```editor:replace-text-selection
file: tap-values.yml
text: ""
```

We can see the available configuration options for the Learning Center package via
```execute
tanzu package available list learningcenter.tanzu.vmware.com --namespace tap-install --kubeconfig kubeconfig.yaml
tanzu package available get learningcenter.tanzu.vmware.com/0.1.0 --values-schema -n tap-install --kubeconfig kubeconfig.yaml
```

Let's now add the required configuration at the end of the file.
```execute
cat << EOF >> tap-values.yml

learningcenter:
  ingressDomain: "learning-center.${TAP_INGRESS_DOMAIN}"
EOF
```
```editor:open-file
file: tap-values.yml
line: 1
```

After that, we can update our profile installation.
```terminal:execute
command: tanzu package installed update tap -p tap.tanzu.vmware.com -v 1.0.2 --values-file tap-values.yml -n tap-install --kubeconfig kubeconfig.yaml
clear: true
```