```dashboard:open-url
url: {{ ENV_TAP_PRODUCT_DOCS_BASE_URL }}/GUID-scc-install-scc.html
```

To install all the additional components required for the ootb testing and scanning supply chain, we first have to remove the following packages from the **excluded_packages**.
```execute
sed -i '/ootb-supply-chain-testing-scanning.\|scanning.apps.\|grype.scanning.apps.\|metadata-store.apps.\|image-policy-webhook.signing./d' tap-values.yml
```
```editor:open-file
file: tap-values.yml
line: 1
```

We can see the available and for our testing and scanning configuration relevant options via
```execute
tanzu package available list ootb-supply-chain-testing-scanning.tanzu.vmware.com -n tap-install --kubeconfig kubeconfig.yaml
tanzu package available get ootb-supply-chain-testing-scanning.tanzu.vmware.com/0.6.1 --values-schema -n tap-install --kubeconfig kubeconfig.yaml
```
```execute
tanzu package available list metadata-store.apps.tanzu.vmware.com.tanzu.vmware.com -n tap-install --kubeconfig kubeconfig.yaml
tanzu package available get metadata-store.apps.tanzu.vmware.com.tanzu.vmware.com/0.5.1 --values-schema -n tap-install --kubeconfig kubeconfig.yaml
```
```execute
tanzu package available list grype.scanning.apps.tanzu.vmware.com -n tap-install --kubeconfig kubeconfig.yaml
tanzu package available get grype.scanning.apps.tanzu.vmware.com/0.5.1 --values-schema -n tap-install --kubeconfig kubeconfig.yaml
```

Let's now update our configuration for the new supply chain ...
```editor:select-matching-text
file: tap-values.yml
text: "supply_chain: basic"
```
```editor:replace-text-selection
file: tap-values.yml
text: "supply_chain: testing_scanning"
```

```editor:select-matching-text
file: tap-values.yml
text: "ootb_supply_chain_basic"
```
```editor:replace-text-selection
file: tap-values.yml
text: "ootb_supply_chain_testing_scanning"
```

... and add the missing configuration at the end of the file.
{% raw %}
```execute
cat << EOF >> tap-values.yml

metadata_store:
  app_service_type: ClusterIP # (optional) Defaults to LoadBalancer. Change to NodePort for distributions that don't support LoadBalancer

grype:
  namespace: dev-space
  targetImagePullSecret: registry-credentials
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