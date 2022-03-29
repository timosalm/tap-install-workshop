# VMware Tanzu Application Platform Installation Workshop

## Prerequisites

## Workshop installation
Download the Tanzu CLI from https://network.tanzu.vmware.com/products/tanzu-application-platform to the root of the directory.
Create a public project called **tap-workshop** in your Harbor instance. There is a Dockerfile in the root directory of this repo. From that root directory, build a Docker image and push it to the project you created:
```
docker build . -t <your-harbor-hostname>/tap-workshop/tap-install-workshop
docker push <your-harbor-hostname>/tap-workshop/tap-install-workshop
```

The workshop demonstrates the binding of a sample application workload to a RabbitMQ Cluster provided by the RabbitMQ Cluster Operator for Kubernetes. Install the operator in your cluster via:
```
./install-rabbit-operator.sh
```

Copy values-example.yaml to values.yaml and set configuration values
```
cp values-example.yaml values.yaml
```
Run the installation script.
```
./install.sh
```

## Debug
```
kubectl logs -l deployment=learningcenter-operator -n learningcenter
```