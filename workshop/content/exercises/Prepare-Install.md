TAP is available through pre-defined **profiles** or individual packages. These profiles are designed to simplify the installation experience and expedite the time to getting started with the platform. 
With the current version of TAP, there are two profiles available:
- **Full**: This profile contains all of the Tanzu Application Platform packages 
- **Light**: Contains packages that drive the Inner Loop developer experience of building and iterating on applications

We will you use the **Full** profile to install all of the TAP packages. 
To be able to configure the packages and explain the compontents they install step-by-step, we will use the ability to exclude packages from a profile configuration file.

Let's now ceate the profile configuration file and exclude all of the packages.
```execute
cat <<EOF > tap-values.yml 
profile: full

# indicates that the customer acknwoldges that the Customer Experience Improvement Program data collection 
# program has been explained to them. 
ceip_policy_disclosed: true # Installation fails if this is set to 'false'

# A list of package to skipp installing 
excluded_packages:
  - accelerator.apps.tanzu.vmware.com
  - api-portal.tanzu.vmware.com
  - run.appliveview.tanzu.vmware.com
  - build.appliveview.tanzu.vmware.com
  - buildservice.tanzu.vmware.com
  - cartographer.tanzu.vmware.com
  - cert-manager.tanzu.vmware.com
  - cnrs.tanzu.vmware.com
  - contour.tanzu.vmware.com
  - controller.conventions.apps.tanzu.vmware.com
  - developer-conventions.tanzu.vmware.com
  - fluxcd.source.controller.tanzu.vmware.com
  - grype.scanning.apps.tanzu.vmware.com
  - image-policy-webhook.signing.apps.tanzu.vmware.com
  - image-policy-webhook.signing.run.tanzu.vmware.com
  - learningcenter.tanzu.vmware.com
  - workshops.learningcenter.tanzu.vmware.com
  - metadata-store.apps.tanzu.vmware.com
  - ootb-delivery-basic.tanzu.vmware.com
  - ootb-supply-chain-basic.tanzu.vmware.com
  - ootb-supply-chain-testing-scanning.tanzu.vmware.com
  - ootb-templates.tanzu.vmware.com
  - scanning.apps.tanzu.vmware.com
  - service-bindings.labs.vmware.com
  - services-toolkit.tanzu.vmware.com
  - controller.source.apps.tanzu.vmware.com
  - spring-boot-conventions.tanzu.vmware.com
  - tap-gui.tanzu.vmware.com
  - tekton.tanzu.vmware.com
EOF
```

```editor:open-file
file: tap-values.yml 
line: 1
```

You can execute the installation via:
```terminal:execute
command: tanzu package install tap -p tap.tanzu.vmware.com -v 1.0.2 --values-file tap-values.yml -n tap-install --kubeconfig kubeconfig.yaml
clear: true
```

With the following command we can now see which packages were installed using the **PackageInstall** Custom Resource.
```terminal:execute
command: tanzu package installed list -n tap-install --kubeconfig kubeconfig.yaml
clear: true
```
As an alternative we can also use the kubectl CLI. 
```terminal:execute
command: kubectl get packageinstalls -n tap-install
clear: true
```

With the current version of TAP it's **not possible to exclude tap-telemetry.tanzu.vmware.com as a package**. There is a section available in the documentation on how to disable it.

To get details for a package, e.g. the `USEFUL-ERROR-MESSAGE`, we can use the following commands.
```terminal:execute
command: tanzu package installed get tap -n tap-install --kubeconfig kubeconfig.yaml
clear: true
```
```terminal:execute
command: kubectl describe packageinstalls tap -n tap-install
clear: true
```
With the eksporter krew plugin, it's easier to have a look at the spec, which references the Package CR, specifies a service account that will be used to install underlying package contents, and references a secret that includes values to be included in package's templating step.
```terminal:execute
command: kubectl eksporter packageinstalls tap -n tap-install
clear: true
```

We can also view the App CR created as a result of the PackageInstall creation.
```terminal:execute
command: kubectl tree packageinstalls tap -n tap-install
clear: true
```
```terminal:execute
command: kubectl describe app tap -n tap-install
clear: true
```

As a summary for package management with Carvel, here is a diagram that shows the relationship of all the CRDs we talked about in the last sections.
![Carvel Package Management](../images/carvel.png)

###### Overlays with PackageInstall
PackageInstalls expose the ability to customize package installation using annotations recognized by kapp-controller.

Since it is impossible for package configuration and exposed data values to meet every consumer’s use case, we have added an annotation which enables consumers to extend the package configuration with custom ytt paths.

The extension annotation is called `ext.packaging.carvel.dev/ytt-paths-from-secret-name` and can be suffixed with a .X, where X is some number, to allow for specifying it multiple times.

For example, `ext.packaging.carvel.dev/ytt-paths-from-secret-name.0: my-overlay-secret` will include the overlay stored in the secret my-overlay-secret during the templating steps of the package. 

```execute
cat << EOF
apiVersion: v1
kind: Secret
metadata:
  name: my-overlay-secret
  namespace: my-ns
stringData:
  add-ns-label.yml: |
    #@ load("@ytt:overlay", "overlay")
    #@overlay/match by=overlay.subset({"kind":"Namespace"}),expects="1+"
    ---
    metadata:
      #@overlay/match missing_ok=True
      labels:
        #@overlay/match missing_ok=True
        custom-lbl: custom-lbl-value 
EOF  
```