Cloud Native Runtimes for Tanzu(CNRs) is a serverless application runtime for Kubernetes that is based on Knative and runs on a single Kubernetes cluster.
```dashboard:open-url
url: {{ ENV_TAP_PRODUCT_DOCS_BASE_URL }}/GUID-cloud-native-runtimes-install-cnrt.html
```

Let's first install CNRs with the following commands and then have a closer look at the problems it solves and how it works.
To install CNRs, we first have to remove the package from the `excluded_packages`.
```editor:select-matching-text
file: tap-values.yml
text: "  - cnrs.tanzu.vmware.com"
```
```editor:replace-text-selection
file: tap-values.yml
text: ""
```

We can see the available configuration options via
```execute
tanzu package available list cnrs.tanzu.vmware.com -n tap-install --kubeconfig kubeconfig.yaml
tanzu package available get cnrs.tanzu.vmware.com/1.1.0 --values-schema -n tap-install --kubeconfig kubeconfig.yaml
```

Let's now add the required configuration at the end of the file.
{% raw %}
```execute
cat << EOF >> tap-values.yml

cnrs:
  domain_name: "cnr.${TAP_INGRESS_DOMAIN}"
  domain_template: "{{.Name}}-{{.Namespace}}.{{.Domain}}"
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