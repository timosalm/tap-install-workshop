After we have containerized our application and pushed it to the container registry of our choice with the help of VMware Tanzu Build Service, 
we are now able to get all the benefits Kubernetes provides for our application by deploying it to a cluster.

Let's use imperative commands for it ...
```terminal:execute
command: |-
    kubectl create ns cnr-demo
    kubectl create deployment tanzu-java-web-app --image "${CONTAINER_REGISTRY_HOSTNAME}/tap-wkld/tanzu-java-web-app" -n cnr-demo
    kubectl expose deployment tanzu-java-web-app --port 8080 --type ClusterIP -n cnr-demo
    kubectl create ingress tanzu-java-web-app --rule="tanzu-java-web-app.${TAP_INGRESS_DOMAIN}/*=tanzu-java-web-app:8080" -n cnr-demo
clear: true
```
... and have a look at all YAML that was created for that minimal configuration.
```terminal:execute
command: |-
    kubectl eksporter deployment,service,ingress -n cnr-demo
clear: true
```

**Cloud Native Runtimes for VMware Tanzu** (CNRs) simplify deploying and operating microservices on Kubernetes. They are a set of capabilities that enable developers to leverage the power of Kubernetes for **Serverless** use cases without first having to master the Kubernetes API.

##### Serverless

Before we have a closer look at CNRs, let's get a common understanding of what Serverless is.
```dashboard:open-dashboard
name: slides
```

##### Knative

CNRs includes Knative, an open source community project that provides a simple, consistent layer over Kubernetes that solves common problems of deploying software, connecting disparate systems together, upgrading software, observing software, routing traffic, and scaling automatically. This layer creates a firmer boundary between the developer and the platform, allowing the developer to concentrate on the software they are directly responsible for.

The major subprojects of Knative are *Serving* and *Eventing*.
- **Serving** is responsible for deploying, upgrading, routing, and scaling. 
- **Eventing** is responsible for connecting disparate systems. Dividing responsibilities this way allows each to be developed more independently and rapidly by the Knative community.

##### Knative Serving
Knative Serving defines four objects that are used to define and control how a serverless workload behaves on the cluster: *Service*, *Configuration*, *Revision*, and *Route*.

**Configuration** is the statement of what the running system should look like. You provide details about the desired container image, environment variables, and the like. Knative converts this information into lower-level Kubernetes concepts like *Deployments*. In fact, those of you with some Kubernetes familiarity might be wondering what Knative is adding. After all, you can just create and submit a *Deployment* yourself, no need to use another component for that.

Which takes us to **Revisions**. These are snapshots of a *Configuration*. Each time that you change a *Configuration*, Knative first creates a *Revision*, and in fact, it is the *Revision* that is converted into lower-level primitives.
It’s the ability to selectively target traffic that makes *Revisions* a necessity. In vanilla Kubernetes, you can roll forward and can roll back, but you can’t do so with traffic. You can only do it with instances of the Service.

A **Route** maps a network endpoint to one or more *Revisions*. You can manage the traffic in several ways, including fractional traffic and named routes.

A **Service** combines a *Configuration* and a *Route*. This compounding makes common cases easier because everything you will need to know is in one place.

Let's first setup a demo namespace and then use kn, the "official" CLI for Knative, to demonstrate some *Knative Serving* capabilities.
```execute
cat <<EOF | kubectl apply -n cnr-demo -f -
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
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "tap-registry"}]}' -n cnr-demo
```

We can create a *Service* using kn service create. 
```terminal:execute
command: kn service create intro-knative-example --image gcr.io/knative-samples/helloworld-go --env TARGET="First" -n cnr-demo
clear: true       
```

To test our deployment send a request to the application after deployment is ready to serve traffic.
```terminal:execute
command: curl -L $(kn service describe intro-knative-example -o url -n cnr-demo)
clear: true
```

In addition to the kn CLI, we can have a closer look by using the underlying Kubernetes custom resources.
```terminal:execute
command: kubectl eksporter services.serving.knative.dev,configurations,routes -n cnr-demo
clear: true
```

Let's now update our *Service* ...
```terminal:execute
command: kn service update intro-knative-example --env TARGET="Second" -n cnr-demo
clear: true
```
... and send another request to the application after the updated deployment is ready to serve traffic.
```terminal:execute
command: curl -L $(kn service describe intro-knative-example -o url -n cnr-demo)
clear: true
```

If you run the following commands you can see that our update command created a second *Revision*. The 1 and 2 suffixes indicate the generation of the *Service*.
```terminal:execute
command: kn revision list -n cnr-demo
clear: true
```
```terminal:execute
command: kubectl get revisions -n cnr-demo
clear: true
```
Did the second replace the first *Revision*? If you’re an end user sending HTTP requests to the URL, yes, it appears as though a total replacement took place. But from the point of view of a developer, both *Revisions* still exist.

You can look more closely at each of these with `kn revision describe <revision-name>`.
```terminal:execute
command: kn revision describe intro-knative-example-00002 -n cnr-demo
clear: true
```
It’s worth taking a slightly closer look at the *Conditions*.

###### Changing the image

Let's now change the container image of the application implemented in Golang to the same application written in Rust ...
```terminal:execute
command: kn service update intro-knative-example --image gcr.io/knative-samples/helloworld-rust -n cnr-demo
clear: true
```
and send another request to the updated application deployment.
```terminal:execute
command: curl -L $(kn service describe intro-knative-example -o url -n cnr-demo)
clear: true
```
Changing the environment variable caused the creation of a second *Revision*. Changing the image caused a third *Revision* to be created. But because you didn’t change the variable, the third *Revision* also says `Hello world: Second`. In fact, almost any update you make to a *Service* causes a new *Revision* to be stamped out. Almost any? What’s the exception? It’s *Routes*. Updating these as part of a *Service* won’t create a new *Revision*.

###### Splitting traffic
Let's now validate that *Route* updates don’t create new *Revisions* by splitting traffic evenly between the last two *Revisions*. 
```terminal:execute
command: kn service update intro-knative-example --traffic intro-knative-example-00001=50 --traffic intro-knative-example-00002=50 -n cnr-demo
clear: true
```
The `--traffic` parameter allows us to assign percentages to each *Revision*. The key is that the percentages must all add up to 100. 
As you can see there are still three revisions.
```terminal:execute
command: kn revision list -n cnr-demo
clear: true
```

Let's send a request and ...
```terminal:execute
command: curl -L $(kn service describe intro-knative-example -o url -n cnr-demo)
clear: true
```
... if you send another request you should see that it works and a slightly different message will be returned from the other *Revision*.
```terminal:execute
command: curl -L $(kn service describe intro-knative-example -o url -n cnr-demo)
clear: true
```
*Hint: If there are timeout issues e.g. due to the scale from zero to one instance, just rerun the command.*


You don’t explicitly need to set traffic to 0% for a *Revision*. You can achieve the same by leaving out *Revisions*.
Finally, you can switch over all the traffic using `@latest` as your target.
```terminal:execute
command: kn service update intro-knative-example --traffic @latest=100 -n cnr-demo
clear: true
```

###### Autoscaling
Scale to zero is enabled by default, but you don't always need it. 

You can configure the autoscaling of your applications via the **minScale** and **maxScale** annotations on a *Service* or a *Revision*.
For a *Revision* you can do that with the `kubectl annotate` command.
```terminal:execute
command: kubectl annotate revision intro-knative-example-00003 autoscaling.knative.dev/minScale=1 -n cnr-demo
clear: true
```
For a *Service* resource, the annotation has to be set in the *RevisionTemplateSpec*.

Another option is to set the annotation via the kn CLI ...
```terminal:execute
command: kn service update intro-knative-example --annotation autoscaling.knative.dev/minScale=2 -n cnr-demo
clear: true
```
```terminal:execute
kubectl get pods -n cnr-demo
```

... or the minimum and maximum scale options.
```terminal:execute
command: kn service update intro-knative-example --scale-min 0 --scale-max 5 -n cnr-demo
clear: true
```

There are several other configuration options to tweak the autoscaler available which we will not cover here, and the Knative Pod Autoscaler isn’t the only autoscaler you can use with Knative Serving, it’s just the one you get by default. Out of the box, you can e.g. also use the Horizontal Pod Autoscaler (HPA), which is a subproject of the Kubernetes project. 

##### Knative Eventing
Knative Eventing provides tools for routing events from event producers to sinks, enabling developers to use an event-driven architecture with their applications.

Knative Eventing resources are loosely coupled, and can be developed and deployed independently of each other. Any producer can generate events before there are active event consumers that are listening. Any event consumer can express interest in an event or class of events, before there are producers that are creating those events.
These events conform to the **CloudEvents** specifications, which enables creating, parsing, sending, and receiving events in any programming language.

Because **Knative Eventing is not yet integrated very well into TAP**, we'll just have a look at a very basic example.

Let's first check the current state using the kn CLI.
```terminal:execute
command: kn trigger list -n cnr-demo && kn source list -n cnr-demo && kn broker list -n cnr-demo
clear: true
```

Under the hood, these commands are groping around for *Trigger*, *Source*, and *Broker* records in Kubernetes. You haven’t done anything yet, so there are none.
A **Trigger** combines information about an event filter and an event subscriber together, a **Source** is a description of something that can create events. In reality, *Triggers* don’t really do anything in themselves. They’re records that get acted on by a **Broker**, with potentially many *Triggers* per *Broker*. A **Subscriber** here is anything that Knative Eventing knows how to send stuff to.

Let's start with the creation of a *Broker* ...
```terminal:execute
command: kn broker create default -n cnr-demo
clear: true
```
... and continue with a *Subscriber*. The simplest thing to put here is a basic web app that can receive *CloudEvents* and perhaps help you to inspect those. 
```terminal:execute
command: kn service create cloudevents-player --image ruromero/cloudevents-player:latest --env BROKER_URL=$(kn broker describe default -o url -n cnr-demo) -n cnr-demo
clear: true
```
After the deployment of the *Service* there should be a URL to access the application in the logs. If you open the URL, you should see a form with fields for all the required attributes of a *CloudEvent* to send an event to the *Broker*.
To also consume the events from the broker with our application and view them in the blank area on the right, we have to create a *Trigger*.
```terminal:execute
command: kn trigger create cloudevents-player --sink cloudevents-player -n cnr-demo
clear: true
```
If you now head back to your browser and try sending an event. You’ll see that the event is both sent and received.
The simple fact here is that I’ve cheated by making the application both the *Source* and *Sink* for events.
The nomenclature of "sinks" and "sources" is already widespread outside of Knative in lots of contexts. *Sources* are where events come from, *Sinks* are where events go.
You didn’t explicitly define a *Source* and only defined a *Sink* in the *Trigger*. This already hints at how flexible *Knative Eventing* actually is.

With the following command, you can have a look at the preinstalled *Sources*.
```terminal:execute
command: kn source list-types
clear: true
```
Knative Eventing provides only three reference *Sources*: the **PingSource**, which produces CloudEvents on a schedule you provide, the **ApiServerSource**, which can observe changes made to raw Kubernetes records and convert these into CloudEvents, and the **ContainerSource** which is a simple wrapper for existing systems that can run on raw Kubernetes and learn how to send events to a URL provided externally.
All the other *Sources* you can see, come with CNRs.

Knative Eventing provides additional more advanced features for e.g. brokering and filtering and Flows which are out of the scope of this workshop.

##### Configuration of CNRs

To configure the CNRs Knative Serving and Knative Eventing with configurations that are not exposed to the Carvel Package you can edit the following Config Maps.
```terminal:execute
command:  kubectl get cm -n knative-serving
clear: true
```
```terminal:execute
command:  kubectl get cm -n knative-eventing
clear: true
```

As an example, let's have a look at the autoscaler configuration.
```terminal:execute
command:  kubectl get cm config-autoscaler -n knative-serving -o yaml
clear: true
```

If you update your TAP installation those will be reset to default if you don't apply them as an [Overlay of a PackageInstall](https://carvel.dev/kapp-controller/docs/v0.32.0/package-install-extensions/)

##### Streaming, Batch, and Functions runtimes

The CNRs product team is currently working on the following additional runtimes:
- **Streaming**: A polyglot runtime that can simplify the orchestration of diverse data processing architecture patterns. By reimagining the building blocks of Spring Cloud Data Flow with polyglot-friendly and Kubernetes-native principles, the Streaming runtime will bridge the gap between application development and data ‘organization silos’.
- **Batch:** Scheduled jobs to complete tasks
- **Functions:** A polyglot serverless function experience for Kubernetes, leveraging Knative and new Cloud Native Buildpacks.

Let's now clean up the environment for the next section, where we have a look on how to automate our path to production.
```terminal:execute
command: kubectl delete ns cnr-demo
clear: true
```

