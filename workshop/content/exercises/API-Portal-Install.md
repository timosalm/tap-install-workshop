API portal for VMware Tanzu enables API consumers to find APIs they can use in their own applications. Consumers can view detailed API documentation and try out an API to see if it can meet their needs. 
```dashboard:open-url
url: {{ ENV_TAP_PRODUCT_DOCS_BASE_URL }}/GUID-api-portal-install-api-portal.html
```

Let's first install API Portal with the following commands and then have a closer look at the problems it solves and how it works.
To install API Portal, we first have to remove the package from the `excluded_packages`.
```editor:select-matching-text
file: tap-values.yml
text: "  - api-portal.tanzu.vmware.com"
```
```editor:replace-text-selection
file: tap-values.yml
text: ""
```

We can see the available configuration options via
```execute
tanzu package available list -n tap-install api-portal.tanzu.vmware.com --kubeconfig kubeconfig.yaml
tanzu package available get api-portal.tanzu.vmware.com/1.0.8  --values-schema -n tap-install --kubeconfig kubeconfig.yaml
```

Let's update our profile installation without additional configuration.
```terminal:execute
command: tanzu package installed update tap -p tap.tanzu.vmware.com -v 1.0.2 --values-file tap-values.yml -n tap-install --kubeconfig kubeconfig.yaml
clear: true
```

In the current 1.0.2 and earlier version, the Ingress resource for API Portal is missing. So we have to create it manually via:
```execute
cat <<EOF | kubectl apply -f -
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: api-portal
  namespace: api-portal
spec:
  routes:
  - services:
    - name: api-portal-server
      port: 8080
  virtualhost:
    fqdn: "api-portal.${TAP_INGRESS_DOMAIN}"
EOF
```
