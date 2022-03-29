**kapp-controller** provides package management and continuous delivery capabilities by utilizing `kapp` to track the resources it’s deploying.

With **secretgen-controller** it's possbile to generate various types of Secrets in-cluster as well as export and import Secrets across namespaces.

Before we have a closer look at both, we first have to install them on our target cluster if we use another **distribution than TKG 1.5**. It's possbile to install them via the instructions on https://carvel.dev, but for our customers we are offering a better installation experience via the **Cluster Essentials for VMware Tanzu**.