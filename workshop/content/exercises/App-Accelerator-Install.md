Enterprise Architects author and publish accelerator projects that provide developers and operators with ready-made, enterprise-conformant code and configurations. You can then use Application Accelerator to create new projects based on those accelerator projects.

```dashboard:open-url
url: {{ ENV_TAP_PRODUCT_DOCS_BASE_URL }}/GUID-tap-gui-plugins-application-accelerator.html
```

Let's first install Application Accelerator with the following commands and then have a closer look at the problems it solves and how it works.
To install Application Accelerator, we first have to remove the package from the `excluded_packages`.
```editor:select-matching-text
file: tap-values.yml
text: "  - accelerator.apps.tanzu.vmware.com"
```
```editor:replace-text-selection
file: tap-values.yml
text: ""
```

We can see the available configuration options via
```execute
tanzu package available list accelerator.apps.tanzu.vmware.com -n tap-install --kubeconfig kubeconfig.yaml
tanzu package available get accelerator.apps.tanzu.vmware.com/1.0.0 --values-schema -n tap-install --kubeconfig kubeconfig.yaml
```

Let's now add the required configuration at the end of the file.
```execute
cat << EOF >> tap-values.yml

accelerator: 
  domain: $TAP_INGRESS_DOMAIN               
  ingress:
    include: true
    enable_tls: false  
  server:
    service_type: ClusterIP
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