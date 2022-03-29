The Application Live View features of the Tanzu Application Platform include sophisticated components to give developers and operators a view into their running workloads on Kubernetes.

```dashboard:open-url
url: {{ ENV_TAP_PRODUCT_DOCS_BASE_URL }}/GUID-tap-gui-plugins-app-live-view.html
```

Let's first install Application Live View with the following commands and then have a closer look at the problems it solves and how it works.
To install Application Live View, we first have to remove the packages from the `excluded_packages`.
```editor:select-matching-text
file: tap-values.yml
text: "  - run.appliveview.tanzu.vmware.com"
after: 1
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