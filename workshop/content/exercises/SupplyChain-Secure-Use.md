Our testing and scanning supply chain provides a more advanced path to deployment.
- Watching a Git Repository or local directory for changes
- Running tests from a developer-provided Tekton Pipeline
- Scanning the source code for known vulnerabilities using Grype
- Building a container image out of the source code with Buildpacks
- Scanning the image for known vulnerabilities
- Applying operator-defined conventions to the container definition
- Deploying the application to the same cluster

We can verify that we have the right supply chains installed via
```terminal:execute
command: tanzu apps cluster-supply-chain list
clear: true
```
or
```terminal:execute
command: kubectl get clustersupplychain
clear: true
```

With the following command, we are able to extract it from the cluster and can have a closer look via VSCode.
```terminal:execute
command: |
 mkdir supply-chain-testing-scanning
 kubectl eksporter "clusterconfigtemplate,clusterimagetemplates,clusterruntemplates,clustersourcetemplates,clustersupplychains,clustertemplates,clusterdelivery,ClusterDeploymentTemplate" | kubectl slice -o supply-chain-testing-scanning/ -f-
clear: true
```

```editor:open-file
file: supply-chain-testing-scanning/clustersourcetemplate-testing-pipeline.yaml
line: 1
```

###### Updates to the Developer Namespace

For source and image scans to happen, scan templates and scan policies must exist in the same namespace as the Workload. These define:
- **ScanTemplate**: how to run a scan, allowing one to change details about the execution of the scan (either for images or source code)
- **ScanPolicy**: how to evaluate whether the artifacts scanned are compliant, for example allowing one to be either very strict, or restrictive about particular vulnerabilities found.

```editor:append-lines-to-file
file: scan-policy.yaml
text: |
    apiVersion: scanning.apps.tanzu.vmware.com/v1beta1
    kind: ScanPolicy
    metadata:
      name: scan-policy
    spec:
      regoFile: |
        package policies

        default isCompliant = false

        # Accepted Values: "Critical", "High", "Medium", "Low", "Negligible", "UnknownSeverity"
        violatingSeverities := ["Critical","High","UnknownSeverity"]
        ignoreCVEs := []

        contains(array, elem) = true {
        array[_] = elem
        } else = false { true }

        isSafe(match) {
        fails := contains(violatingSeverities, match.Ratings.Rating[_].Severity)
        not fails
        }

        isSafe(match) {
        ignore := contains(ignoreCVEs, match.Id)
        ignore
        }

        isCompliant = isSafe(input.currentVulnerability)
```
```terminal:execute
command: kubectl apply -f scan-policy.yaml
clear: true
```

In order for source code testing to be present in the supply chain, a **Tekton Pipeline** must exist in the same namespace as the Workload.
```editor:append-lines-to-file
file: tekton-pipeline.yaml
text: |
    apiVersion: tekton.dev/v1beta1
    kind: Pipeline
    metadata:
      name: developer-defined-tekton-pipeline
    labels:
      apps.tanzu.vmware.com/pipeline: test     # (!) required
    spec:
      params:
        - name: source-url                       # (!) required
        - name: source-revision                  # (!) required
      tasks:
        - name: test
          params:
            - name: source-url
              value: $(params.source-url)
            - name: source-revision
              value: $(params.source-revision)
          taskSpec:
            params:
            - name: source-url
            - name: source-revision
            steps:
            - name: test
              image: maven:3-openjdk-11
              script: |-
                cd `mktemp -d`
                wget -qO- $(params.source-url) | tar xvz -m
                mvn test
```
```terminal:execute
command: kubectl apply -f tekton-pipeline.yaml -n dev-space
clear: true
```

To run our Workload with the new supply chain we have to mark the Workload as having tests enabled.
```execute
tanzu apps workload update spring-sensors -n dev-space   --label apps.tanzu.vmware.com/has-tests=true \
```

{% raw %}
```editor:insert-value-into-yaml
file: workload.yaml
path: metadata.labels
value:
  apps.tanzu.vmware.com/has-tests: "true"
```
{% endraw %}

Let's have a closer look via kubectl tree.
```terminal:execute
command: kubectl tree workload spring-sensors -n dev-space
clear: true
```

###### Viewing scan status
```terminal:execute
command: kubectl describe sourcescans spring-sensors -n dev-space
clear: true
```
```terminal:execute
command: kubectl describe imagescans spring-sensors -n dev-space
clear: true
```

There is also an **Insight CLI** available to query for image, source, package, and vulnerability relationships from a metadata store that saves software bills of materials (SBoMs) to a database.
```dashboard:open-url
url: {{ ENV_TAP_PRODUCT_DOCS_BASE_URL }}/GUID-scst-store-query_data.html
```

###### Container signing

The Sign part of the supply chain provides an admission WebHook that:
- Verifies signatures on container images used by Kubernetes resources.
- Enforces policy by allowing or denying container images from running based on configuration.
- Adds metadata to verified resources according to their verification status.
- It intercepts all resources that create Pods as part of their lifecycle.

This component uses **cosign** as its backend for signature verification and is compatible only with cosign signatures. 

**By default, once installed, this component does not include any policy resources and does not enforce any policy.** The operator must create a ClusterImagePolicy resource in the cluster before the WebHook can perform any verifications