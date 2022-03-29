As mentioned **kapp-controller** provides package management and continuous delivery capabilities by utilizing `kapp` to track the resources it’s deploying.

kapp-controller has three primary use cases, which we will talk about:
- Continuous Delivery
- Package Consumption
- Package Authoring

##### Continuous Delivery
kapp-controller's **App** CRD provides a declarative way to install, manage, and upgrade applications on a Kubernetes cluster.
```execute
cat <<EOF > kappctl-spring-petclinic.yaml
apiVersion: kappctrl.k14s.io/v1alpha1
kind: App
metadata:
  name: spring-petclinic-kappctl
spec:
  serviceAccountName: default-ns-sa
  fetch:
  - git:
      url: https://github.com/vmware-tanzu/carvel-simple-app-on-kubernetes
      ref: origin/develop
      subPath: config-step-2-template
  template:
  - ytt:
      inline:
        paths:
          values.yaml: |
            #@data/values
            ---
            hello_msg: "Hello TAP"             
  deploy:
  - kapp: {}
EOF
```
```editor:open-file
file: kappctl-spring-petclinic.yaml
line: 1
```
If we deploy the App, kapp-controller will fetch the GitHub repo, template the subPath for us using ytt to override the default configuration values, and then create the resources for us.

Before we deploy it, we first habe to create a service account that allows us to change any resource in the default namespace. It used as the serviceAccountName in the app spec above.
```terminal:execute
command: kapp deploy -a default-ns-rbac -f https://raw.githubusercontent.com/vmware-tanzu/carvel-kapp-controller/develop/examples/rbac/default-ns.yml -y
clear: true
```
```terminal:execute
command: kubectl apply -f kappctl-spring-petclinic.yaml
clear: true
```
```terminal:execute
command: kubectl get app.kappctrl.k14s.io spring-petclinic-kappctl
clear: true
```
```terminal:execute
command: kubectl get all
clear: true
```

Let's execute the following command to clean up the directory and look at the **Package consumption** and **authoring** use-cases for the TAP installation in the next section.
```execute
kubectl delete -f kappctl-spring-petclinic.yaml
rm kappctl-spring-petclinic.yaml
kapp delete -a default-ns-rbac
```