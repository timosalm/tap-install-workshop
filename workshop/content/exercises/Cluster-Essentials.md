The instructions on how to install the **Cluster Essentials for VMware Tanzu** can be found in the TAP product documentation
```dashboard:reload-dashboard
name: "Product Documentation"
url: {{ ENV_TAP_PRODUCT_DOCS_BASE_URL }}/GUID-install-general.html#tanzu-cluster-essentials
```

Set the kubectl context to the cluster where you want to install VMware Tanzu Application Platform on.
```execute
kubectl ctx tap-install
```

For this installation, we are using the pivnet CLI to download the Cluster Essentials for VMware Tanzu archive from Tanzu Network.
```dashboard:open-url
url: https://network.tanzu.vmware.com/products/tanzu-cluster-essentials/
```

```terminal:execute
command: pivnet download-product-files --product-slug='tanzu-cluster-essentials' --release-version='1.0.0' --product-file-id=1105818
clear: true
```
Create a new directory and unpack the archive.
```terminal:execute
command: mkdir tanzu-cluster-essentials && tar -xvf "tanzu-cluster-essentials-linux-amd64-1.0.0.tgz" -C tanzu-cluster-essentials
clear: true
```

Let's have a look at the install script, which uses the Carvel tools ytt, imgpkg, and kbld.
```editor:open-file
file: tanzu-cluster-essentials/install.sh
line: 1
```

Set the following ENV variables for the installation:
```execute
export INSTALL_BUNDLE=registry.tanzu.vmware.com/tanzu-cluster-essentials/cluster-essentials-bundle@sha256:82dfaf70656b54dcba0d4def85ccae1578ff27054e7533d08320244af7fb0343
export INSTALL_REGISTRY_HOSTNAME=registry.tanzu.vmware.com
export INSTALL_REGISTRY_USERNAME=$TANZU_NET_USER
export INSTALL_REGISTRY_PASSWORD=$TANZU_NET_PASSWORD
```

Change the directory to the folder with the installation files and run the `install.sh` script
```execute 
cd tanzu-cluster-essentials && ./install.sh && cd ..
```

The folder also contains the kapp CLI for installation. In our case the CLI is already installed in the environment.

Let's now check that the Pods of the kapp-controller, and secretgen-controlle are running.
```execute 
kubectl get pods -n kapp-controller && kubectl get pods -n secretgen-controller
```

With both running, we can now have a closer look at them.