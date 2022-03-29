kapp-controller's **Package** CRD provides a way to write and share software as Packages so that Kubernetes users can easily discover and install them.

kapp-controller's **PackageRepository** CRD allows you to easily discover a collection of Packages.

Let's add the TAP PackageRepository to our target Kubernets cluster via the tanzu CLI to have a closer look.
```execute
tanzu package repository add tanzu-tap-repository \
  --url registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.0.2 \
  --namespace tap-install --kubeconfig kubeconfig.yaml
tanzu package repository get tanzu-tap-repository --namespace tap-install --kubeconfig kubeconfig.yaml
```

Of course, we can also use the kubectl CLI to get information about the TAP PackageRepository ...
```terminal:execute
command: kubectl describe PackageRepository tanzu-tap-repository --namespace tap-install
clear: true
```
... and the eksporter krew plugin which removes several fields, e.g. for the status information.
```terminal:execute
command: kubectl eksporter PackageRepository tanzu-tap-repository --namespace tap-install
clear: true
```
As you can see in this case the Carvel tool **imgpkg** is used for bundling the package repository but there are several other options available.

We can now have a look what packages were added to the cluster via the following commands.
```terminal:execute
command: tanzu package available list --namespace tap-install --kubeconfig kubeconfig.yaml
clear: true
```
```terminal:execute
command: kubectl get packages -n tap-install
clear: true
```

The **Package** resource is created for every new version of a package and it carries information about how to fetch, template, and deploy the it.
```terminal:execute
command: kubectl eksporter packages tap.tanzu.vmware.com.1.0.2 -n tap-install
clear: true
```

Part of the spec is the refName attribute, which references a **PackageMetadata** CR.
Package Metadata are attributes of a single package that do not change frequently and that are shared across multiple versions of a single package. It contains information similar to a project’s README.md.
```terminal:execute
command: kubectl eksporter packagemetadata  tap.tanzu.vmware.com -n tap-install
clear: true
```

In the next section, we will install the first TAP Packages by using the **PackageInstall** Custom Resource.