As mentioned, with **secretgen-controller** it's possbile to generate various types of Secrets in-cluster as well as export and import Secrets across namespaces.

##### Generating Secrets
The generation of the following Secret types is supported: Certificates (CAs and leafs), Passwords, RSA Keys, and SSH Keys.

We will create the following Password Secret via kapp now.
```execute
cat <<EOF > secgc-password.yaml
apiVersion: secretgen.k14s.io/v1alpha1
kind: Password
metadata:
  name: postgresql-password
spec: {}
EOF
```
```editor:open-file
file: secgc-password.yaml
line: 1
```
```terminal:execute
command: kapp deploy -a postgresql-password -f secgc-password.yaml
clear: true
```

Via the following command we should now be able to see the generated Secret object.
```terminal:execute
command: kapp inspect -a postgresql-password --tree
clear: true
```

Let's have a closer look at the generated secret.
```terminal:execute
command: kubectl get secret postgresql-password -o yaml
clear: true
```
By default generated secrets have a predefined set of keys. In a lot of cases Secret consumers expect a specific set of keys within data. With the **Secret Template** functionality, it's possible to customize the keys within a secret.

For our Password Secret this can be done like this:
```execute
cat <<EOF > secgc-password.yaml
apiVersion: secretgen.k14s.io/v1alpha1
kind: Password
metadata:
  name: postgresql-password
spec:
  secretTemplate:
    type: Opaque
    stringData:
      postgresql-pass: \$(value)
EOF
```
```editor:open-file
file: secgc-password.yaml
line: 1
```
```execute
kapp delete -a postgresql-password -y
kapp deploy -a postgresql-password -f secgc-password.yaml
```
```terminal:execute
command: kubectl get secret postgresql-password -o yaml
clear: true
```

##### Exporting and Importing Secrets to other Namespaces

Access to Secret is commonly scoped to its containing namespace. In some cases an owner of a Secret may want to export it to other namespaces for consumption by other resrouces in the system. 
With the two CRDs **SecretExport** and **SecretImport** (and "placeholder secrets"), secretgen-controller enables sharing of Secrets across namespaces.

For the installation of TAP, the credentials to fetch container images from Tanzu Network are required in several namespaces. 
For a better installation experience for our customers, the Tanzu CLI will be used instead of applying the CRDs manually to share those credentials.

Let's first create a namespace called tap-install to group objects related to the TAP installation together logically ...
```terminal:execute
command: kubectl create ns tap-install
clear: true
```
... and set the following environment variables for the credentials.
```execute
export INSTALL_REGISTRY_HOSTNAME=registry.tanzu.vmware.com
export INSTALL_REGISTRY_USERNAME=$TANZU_NET_USER
export INSTALL_REGISTRY_PASSWORD=$TANZU_NET_PASSWORD
```

With the `--export-to-all-namespaces` flag of the `tanzu secret registry add` command, in addition to a Secret of type kubernetes.io/dockerconfigjsona, a **SecretExport** resource with the same name will be created, which makes the secret available across all namespaces in the cluster.
```execute
tanzu secret registry add tap-registry \
  --username ${INSTALL_REGISTRY_USERNAME} --password ${INSTALL_REGISTRY_PASSWORD} \
  --server ${INSTALL_REGISTRY_HOSTNAME} \
  --export-to-all-namespaces --yes --namespace tap-install --kubeconfig kubeconfig.yaml
```
Let's have a look at the created SecretExport.
```terminal:execute
command: kubectl get SecretExport.secretgen.carvel.dev tap-registry -n tap-install -o yaml
clear: true
```

As an example, let's now create a a **SecretImport** to share this Secret with a different namespace.
```execute
cat <<EOF > secgc-import.yaml
apiVersion: secretgen.carvel.dev/v1alpha1
kind: SecretImport
metadata:
  name: tap-registry
spec:
  fromNamespace: tap-install
EOF
cat secgc-import.yaml
```
```terminal:execute
command: kubectl apply -f secgc-import.yaml
clear: true
```

Now there should be the 'tap-registry' Secret in the default namespace created by the SecretImport.
```terminal:execute
command: kubectl get secret tap-registry
clear: true
```

**Placeholder Secrets** are an alternative to SecretImport to import image pull secrets exported via SecretExport.

A placeholder secret is:
- a plain Kubernetes Secret
- with kubernetes.io/dockerconfigjson type
- has secretgen.carvel.dev/image-pull-secret annotation

Example of a placeholder secret:
```execute
cat << EOF
apiVersion: v1
kind: Secret
metadata:
  name: tap-registry
  annotations:
    secretgen.carvel.dev/image-pull-secret: ""
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: e30K
EOF
```

Execute the following command to clean up the directory.
```execute
kapp delete -a postgresql-password -y
kubectl delete SecretImport tap-registry
rm secgc-password.yaml
rm secgc-import.yaml
```