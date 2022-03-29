The Services Toolkit makes it easy to discover, curate, consume, and manage such backing services across single or multi-cluster environments. 
```dashboard:open-url
url: {{ ENV_TAP_PRODUCT_DOCS_BASE_URL }}/GUID-services-toolkit-install-services-toolkit.html
```

Let's first install Services Toolkit with the following commands and then have a closer look at the problems it solves and how it works.
To install Services Toolkit, we have to remove the packages from the `excluded_packages`.
```editor:select-matching-text
file: tap-values.yml
text: "  - service-bindings.labs.vmware.com"
after: 1
```
```editor:replace-text-selection
file: tap-values.yml
text: ""
```

Because there are currently now configuration options available, we can directly update our profile installation.
```terminal:execute
command: tanzu package installed update tap -p tap.tanzu.vmware.com -v 1.0.2 --values-file tap-values.yml -n tap-install --kubeconfig kubeconfig.yaml
clear: true
```

For our example, let's also create a RabbitMQ cluster provided by the RabbitMQ Cluster Operator for Kubernetes, which is used for asynchronous communication between our application and a ```sensors-publisher``` application that also has to be deployed in the workshop namespace.

```terminal:execute
command: |
    kapp -y deploy --app rmq-operator --file https://github.com/rabbitmq/cluster-operator/releases/download/v1.10.0/cluster-operator.yml
    kubectl apply -f - << EOF
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: resource-claims-rmq
      labels:
        resourceclaims.services.apps.tanzu.vmware.com/controller: "true"
    rules:
    - apiGroups: ["rabbitmq.com"]
      resources: ["rabbitmqclusters"]
      verbs: ["get", "list", "watch", "update"]
    ---
    apiVersion: services.apps.tanzu.vmware.com/v1alpha1
    kind: ClusterResource
    metadata:
      name: rabbitmq
    spec:
      shortDescription: RabbitMQ Message Broker
      resourceRef:
        group: rabbitmq.com
        kind: RabbitmqCluster
    EOF
clear: true
```

```terminal:execute
command: |
    cat << EOF |  kubectl apply -f -
    apiVersion: rabbitmq.com/v1beta1
    kind: RabbitmqCluster
    metadata:
      name: rmq-1
      namespace: dev-space
    spec:
      resources:
        requests:
          cpu: 100m
          memory: 500Mi
        limits:
          cpu: 100m
          memory: 500Mi
    EOF
clear: true
```

```terminal:execute
command: |
    cat << EOF |  kubectl apply -f -
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: sensors-publisher
      namespace: dev-space
      labels:
        app: sensors-publisher
    spec:
      replicas: 1
      template:
        metadata:
          name: sensors-publisher
          labels:
            app: sensors-publisher
        spec:
          containers:
          - name: sensors-publisher
            image: harbor.tap.amer.end2end.link/tap/spring-sensors-sensor
            imagePullPolicy: IfNotPresent
            volumeMounts:
            - mountPath: /bindings/rmq
              name: service-binding
            env:
              - name: SERVICE_BINDING_ROOT
                value: /bindings
          restartPolicy: Always
          volumes:
            - name: service-binding
              projected:
                sources:
                - secret:
                    name: rmq-1-default-user
      selector:
          matchLabels:
            app: sensors-publisher
    EOF
clear: true
```
