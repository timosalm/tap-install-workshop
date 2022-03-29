Enterprise Architects author and publish accelerator projects that provide developers and operators with ready-made, enterprise-conformant code and configurations. You can then use Application Accelerator to create new projects based on those accelerator projects.

The Application Accelerator user interface (UI) enables you to discover available accelerators, configure them, and generate new projects to download.

On the Generate Accelerators page, supply **any configuration options needed** to generate the project. The Application Architect has defined these in the **accelerator.yaml** in the accelerator definition. Setting some options can cause others to appear that also must be specified.

###### Creating an Accelerator

```dashboard:open-url
url: https://docs.vmware.com/en/Application-Accelerator-for-VMware-Tanzu/1.0/acc-docs/GUID-creating-accelerators-accelerator-yaml.html
```

###### Application Accelerator plugin for tanzu CLI
```dashboard:open-url
url: https://docs.vmware.com/en/Application-Accelerator-for-VMware-Tanzu/1.0/acc-docs/GUID-acc-cli.html#accelerator-commands-3
```

```terminal:execute
command: tanzu accelerator list
clear: true
```

```terminal:execute
command: kubectl get Accelerator -n accelerator-system
clear: true
```

```terminal:execute
command: kubectl eksporter Accelerator hello-fun -n accelerator-system 
clear: true
```

{% raw %}
```terminal:execute
command: |
    tanzu accelerator generate hello-fun --server-url "http://accelerator.${TAP_INGRESS_DOMAIN}" --options="{\"deploymentType\": \"workload\",\"artifactId\":\"hello-vmware\"}"
clear: true
```
{% endraw %}
