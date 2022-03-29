Tanzu Application Platform GUI is a tool for developers to view a organization’s running applications and services.

```dashboard:open-url
url: {{ ENV_TAP_PRODUCT_DOCS_BASE_URL }}/GUID-tap-gui-about.html
```

Let's first install TAP GUI with the following commands and then have a closer look at the problems it solves and how it works.
To install TAP GUI, we first have to remove the package from the `excluded_packages`.
```editor:select-matching-text
file: tap-values.yml
text: "  - tap-gui.tanzu.vmware.com"
```
```editor:replace-text-selection
file: tap-values.yml
text: ""
```

We can see the available configuration options via
```execute
tanzu package available list tap-gui.tanzu.vmware.com -n tap-install --kubeconfig kubeconfig.yaml
tanzu package available get tap-gui.tanzu.vmware.com/1.0.2 --values-schema -n tap-install --kubeconfig kubeconfig.yaml
```

Let's now add the required configuration at the end of the file.
```execute
cat << EOF >> tap-values.yml

tap_gui:
  ingressEnabled: true
  ingressDomain: $TAP_INGRESS_DOMAIN
  service_type: ClusterIP
  app_config:
    backend:
      baseUrl: "http://tap-gui.${TAP_INGRESS_DOMAIN}"
      cors:
        origin: "http://tap-gui.${TAP_INGRESS_DOMAIN}"
    app:
      baseUrl: "http://tap-gui.${TAP_INGRESS_DOMAIN}"
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