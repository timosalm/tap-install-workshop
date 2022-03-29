Our basic supply chain provides a quick path to deployment.
- Watching a Git repository or local directory for changes
- Building a container image out of the source code with Buildpacks
- Applying operator-defined conventions to the container definition
- Deploying the application to the same cluster

With the following command, we are able to extract it from the cluster and can have a closer look via VSCode.
```terminal:execute
command: |
 mkdir supply-chain-basic
 kubectl eksporter "clusterconfigtemplate,clusterimagetemplates,clusterruntemplates,clustersourcetemplates,clustersupplychains,clustertemplates,clusterdelivery,ClusterDeploymentTemplate" | kubectl slice -o supply-chain-basic/ -f-
clear: true
```

```editor:open-file
file: supply-chain-basic/clustersupplychain-source-to-url.yaml
line: 1
```

Cartographer uses the **ClusterSupplyChain** object to link the different Cartographer objects. App operators describe which "shape of applications" they deal with (via **spec.selector**) and what series of resources are responsible for creating an artifact that delivers it (via **spec.resources**).

The **Cartographer objects sit on top of tools** like Flux and kpack and are wrapped with their respective manifest files. This way, **Cartographer doesn’t care what tools are used under the hood**.

Let’s take a closer look at each Cartographer object and see how they wrap the individual components inside of it.
- **ClusterSourceTemplate** indicates how the supply chain could instantiate an object responsible for providing source code. In our example, we use Flux to pull source code from the source code repository.All ClusterSourceTemplate cares about is whether the **urlPath** and **revisionPath** are passed in correctly from the template.
- **ClusterImageTemplate** instructs how the supply chain should instantiate an object responsible for supplying container images. The outputof the underlying tool has to be passed into the cartographer's Object **imagePath**.
- **ClusterConfigTemplate** instructs the supply chain how to instantiate a Kubernetes object that knows how to make Kubernetes configurations available to further resources in the chain.
- **ClusterDeploymentTemplate** indicates how the delivery should configure the environment
- **ClusterTemplate** instructs the supply chain to instantiate a Kubernetes object that has no outputs to be supplied to other objects in the chain. It can for example be used to create any Kubernetes/Knative object, such as deployment, services, Knative services, etc.
- A **ClusterRunTemplate** differs from supply chain templates in many aspects (e.g. cannot be referenced directly by a ClusterSupplyChain, **outputs** provide a free-form way of exposing any form of results). It defines how an immutable object should be stamped out based on data provided by a **Runnable**.

Let's now deliver our first application trough the supply chain via a **Workload**, which allows the developer to pass information about the app to be delivered. 
We use a GIT repository as a source. With that the **Supply Chain is repeatable**, so each new commit to the codebase will trigger another execution of the supply chain and feveloper have to apply a Workload only once if they start with a new application or microservice
```terminal:execute
command: |
  tanzu apps workload create spring-sensors \
  --git-branch main \
  --git-repo https://github.com/sample-accelerators/spring-sensors-rabbit.git \
  --type web \
  --kubeconfig kubeconfig.yaml -n dev-space
clear: true
```

It's also possible to configure a Workload via a YAML file.
```editor:append-lines-to-file
file: workload.yaml
text: |
    apiVersion: carto.run/v1alpha1
    kind: Workload
    metadata:
      name: spring-sensors
      namespace: dev-space
      labels:
        apps.tanzu.vmware.com/workload-type: web
    spec:
      source:
        git:
          url: https://github.com/sample-accelerators/spring-sensors-rabbit.git
          ref:
            branch: main
```
It can then be applied with the tanzu cli with the following command.
```
tanzu apps workload create -f workload.yaml
```

I addition, it's also supported via the tanzu cli's app plugin to use source code from the filesystem instead of a GIT repository. 
```
tanzu apps workload create my-app \
  --local-path . \
  --source-image $REGISTRY/source \
  --type web
```

tanzu cli's app plugin also provides the functionality to stream logs a Workload from all the pods that are involved in the process.
```execute-2
tanzu apps workload tail spring-sensors --since 1h -n dev-space --kubeconfig kubeconfig.yaml
```

The kubectl tree command is also very helpful to see the current state of the supply chain.
```terminal:execute
command: kubectl tree workload spring-sensors -n dev-space
clear: true
```

Let's have a closer look at the configuration options the Workload provides.
```dashboard:open-url
url: https://cartographer.sh/docs/v0.2.0/reference/workload/#workload
```

There are also **params** which you can configure that will be used through the supplychain.
```terminal:execute
command: kubectl eksporter "clusterconfigtemplate,clusterimagetemplates,clusterruntemplates,clustersourcetemplates,clustersupplychains,clustertemplates,clusterdelivery,ClusterDeploymentTemplate" |  grep "param("
clear: true
```

Let's use the tanzu CLI to verify the workloads are deployed and running. 
```terminal:execute
command: tanzu apps workload list -n dev-space --kubeconfig kubeconfig.yaml
clear: true
```
Once the status shows **Ready**, let's see how to access our application.
```execute
tanzu apps workload get spring-sensors -n dev-space --kubeconfig kubeconfig.yaml
```

To disable auto scaling we can set the minScale annotation. With the tanzu CLI setting a annotation directly via a flag is not yet supported.
```execute
tanzu apps workload update spring-sensors -n dev-space --param "annotations=autoscaling.knative.dev/minScale=\"1\"" --kubeconfig kubeconfig.yaml
```
But the workload custom resource already supports it.

{% raw %}
```editor:insert-value-into-yaml
file: workload.yaml
path: metadata
value:
  annotations:
    autoscaling.knative.dev/minScale: "1"
```
{% endraw %}
##### GitOps
```dashboard:open-url
url: {{ ENV_TAP_PRODUCT_DOCS_BASE_URL }}/GUID-scc-ootb-supply-chain-basic.html#gitops
```

##### Convention Service

Convention Service provides a means for people in operational roles to express their hard-won knowledge and opinions about how applications should run on Kubernetes as a convention. 

The service is composed of two components:
- The **convention controller** provides the metadata to the convention server and executes the updates to Pod Template Spec as per the convention server’s requests.
- The **convention server** receives and evaluates metadata associated with a workload and requests updates to the Pod Template Spec associated with that workload. You can have one or more convention servers for a single convention controller instance. Convention Service currently supports defining and applying conventions for Pods.

```dashboard:open-url
url: {{ ENV_TAP_PRODUCT_DOCS_BASE_URL }}/GUID-convention-service-about.html
```

```terminal:execute
command: kubectl get ClusterPodConvention
clear: true
```

```terminal:execute
command: kubectl get PodIntent -n dev-space
clear: true
```

In the next section, we will have a look how **Tanzu Application Platform GUI** helps us to get more information about deployed workloads.