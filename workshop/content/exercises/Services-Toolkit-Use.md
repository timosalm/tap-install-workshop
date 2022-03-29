Tanzu Application Platform makes it easy to discover, curate, consume, and manage backing services, such as databases, queues, and caches, across single or multi-cluster environments. 

This experience is made possible in Tanzu Application Platform by using the **Services Toolkit** component. 

Within the context of Tanzu Application Platform, one of the most important use cases is binding an application workload to a backing service such as a PostgreSQL database or a RabbitMQ queue. 
This use case is made possible by the [Service Binding Specification](https://github.com/k8s-service-bindings/spec) for Kubernetes. 

Let's first check whether the RabbitMQ cluster the ```sensors-publisher``` application is running in the workshop namespace.
```execute
kubectl get RabbitmqCluster -n dev-space
```
```execute
kubectl get deployments sensors-publisher -n dev-space
```

```execute
tanzu service instance list -owide --kubeconfig kubeconfig.yaml -n dev-space
```

We are then able to add the ServiceBinding configuration to our workload.
```editor:insert-value-into-yaml
file: workload.yaml
path: spec
value:
  serviceClaims:
    - name: rmq
      ref:
        apiVersion: rabbitmq.com/v1beta1
        kind: RabbitmqCluster
        name: rmq-1
```
```execute
tanzu apps workload update -f workload.yaml --kubeconfig kubeconfig.yaml
```

For both, the credentials that are required for the connection to the RabbitMQ cluster are injected as environment variables into the containers via a service binding.
```execute
kubectl get ServiceBinding -n dev-space
```
If we have a closer look at one of the ServiceBinding objects, we can see references to a ResourceClaim for the RabbitMQ Cluster and the Knative Serving Service of our application.
```execute
kubectl get ServiceBinding spring-sensors-rmq -n dev-space -o yaml | yq e '.spec' -
```
Additionally, the name of the Kubernetes Secret that includes the credentials for the RabbitMQ cluster is available in the `status.binding` field.
```execute
kubectl get ServiceBinding spring-sensors-rmq -n dev-space -o yaml | yq e '.status.binding' -
```
Behind the scenes, based on those Custom Resource Definitions objects, the Kubernetes Secret that includes the credentials will be mounted to the application containers as a volume.


 It's also supported to direct reference a Kubernetes Secret resources to enable developers to connect their application workloads to almost any backing service, including backing services that are running external to the platform and do not adhere to the Provisioned Service specifications.
 ```dashboard:open-url
url: {{ ENV_TAP_PRODUCT_DOCS_BASE_URL }}/GUID-getting-started.html#use-case-3--binding-an-application-to-a-service-running-outside-kubernetes-32
```
