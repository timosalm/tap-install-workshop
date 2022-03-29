To be able to get all the benefits for our application Kubernetes provides, we have to containerize it.

The most obvious way to do this, is to write a Dockerfile, run **docker build** and push it to the container registry of our choice via **docker push**.

##### Dockerfile

![Docker Process](../images/dockerfile.png)

The first line of the Dockerfile in the diagram defines the base container image from Dockerhub. It includes Alpine Linux as an operating system, additional libraries, and in this case OpenJDK 8 as a runtime environment for an Java application.
In the second line the default port of a Spring Boot web application is exposed to be able to access it from outside of the container and the  third line copies a fat JAR file of the application to the root of the container folder structure and renames it to app.jar.
Finally, the last line sets the start command for this container, which basically runs the JAR file with the embedded Tomcat using Java.

As you can see, in general it is relatively easy and requires little effort to containerize an application, but whether you should go into production with it, is another question, because it is hard to create an optimized and secure container image (or Dockerfile).

Let's have a look at a more advanced Dockerfile for the same application as a comparison. 
```execute
cat <<EOF > Dockerfile
FROM adoptopenjdk:11-jre-hotspot as builder
WORKDIR application
ARG JAR_FILE=target/*.jar
COPY \${JAR_FILE} application.jar
RUN java -Djarmode=layertools -jar application.jar extract

FROM adoptopenjdk:11-jre-hotspot
WORKDIR application
COPY --from=builder application/dependencies/ ./
COPY --from=builder application/spring-boot-loader/ ./
COPY --from=builder application/snapshot-dependencies/ ./
COPY --from=builder application/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
EOF
``` 
```editor:open-file
file: Dockerfile
line: 1
```

Docker containers consist of a base image and additional layers. Once the layers are built, they'll remain cached. Therefore, subsequent generations will be much faster. Changes in the lower-level layers also rebuild the upper-level ones. Thus, the infrequently changing layers should remain at the bottom, and the frequently changing ones should be placed on top.
In the first Dockerfile example, we used the fat jar approach. As a result, a single artifact embeds all the dependencies and the application source code. So, any change in the source code forces the rebuilding of the entire layer.

Spring Boot version 2.3.0 introduced two new features to improve the Docker image generation. The first one is **Layered jars**, which is used in the second example and helps us to get the most out of the Docker layer generation.
By using this Dockerfile, when we change our source code, we'll only rebuild the application layer. The rest will remain cached.
In additon, we seperated the build and the run stage using a method called [multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/) to reduce the image size, and used a newer OpenJDK version and different distribution.

The second new feature introduced in Spring Boot 2.3 to improve the Docker image generation is Cloud Native Buildpack support.

##### (Cloud Native) Buildpacks

Buildpacks were first conceived by Heroku in 2011. Since then, they have been adopted by Cloud Foundry and other PaaS.
And the new generation of buildpacks, the [Cloud Native Buildpacks](https://buildpacks.io), is an incubating project in the CNCF which was initiated by Pivotal and Heroku in 2018 .

Cloud Native Buildpacks (CNBs) detect what is needed to compile and run an application based on the application's source code (**detect phase**). 
The application is then compiled by the appropriate buildpack and a container image with best practices in mind is build with the runtime environment (**build phase**). 
**Builders** are an ordered combination of buildpacks with a base build image, a lifecycle, and reference to a run image. They take in your app source code and build the output app image. The build image provides the base environment for the builder (for eg. an Ubuntu Bionic OS image with build tooling) and a run image provides the base environment for the app image during runtime. A combination of a build image and a run image is called a **stack**.

The biggest benefits of CNBs are increased security, minimized risk, and increased developer productivity because they don't need to care much about the details of how to build a container.
Improvements of the Cloud Native Buildpacks compared to older generations are that they are able to also compile applications which is optional, they are more modular, the builds are faster and the containers are OCI compatible. An open industry standard for container formats and runtimes.

With Spring Boot 2.3 and later you can create a container image using the open source [Paketo Buildpacks](https://paketo.io) with the following commands for Maven ...
```
mvn spring-boot:build-image 
```
... and Gradle.
```
gradle bootBuildImage
```

Another option is to use the [pack CLI](https://buildpacks.io/docs/tools/pack/), a tool maintained by the Cloud Native Buildpacks project to support the use of buildpacks.
```
pack build
```

Let’s clean up our resources before we move on.
```terminal:execute
command: rm Dockerfile
clear: true
```

##### VMware Tanzu Build Service and kpack

With all the benefits of Cloud Native Buildpacks, one of the biggest challenges with container images still is to keep the operating system, used libraries, etc. up-to-date in order to minimize attack vectors by CVEs. For a single container image this is relatively easy and requires little effort, as we saw in the previous examples. However, at scale this looks different.

With VMware Tanzu Build Service (TBS), it's possible to specify a repository with the branch or revision as the source for the source code, as well as the files in a local directory. 
If a branch is specified and the source code in the repository changes or if there is a new version of the buildpack or the base operating system available (e.g. due to a CVE), a new container is automatically created and pushed to the target registry.
With an appropriate Continous Deployment pipeline, it is thus possible to deploy Security patches automatically.
This fully automated update functionality of the base container stack is a big competitive advantage compared to tools like Redhat's Source2Image and Google's Kaniko.

The main functionality of the TBS is also available as [kpack](https://github.com/pivotal/kpack) as open source software.
--> **TBS Pricing & Packaging** document for information on how it's licensed and commercial vs OSS features.

Let's now check whether TBS was successfully installed and try it out.
```terminal:execute 
command: tanzu package installed get buildservice -n tap-install --kubeconfig kubeconfig.yaml
clear: true
```

To interact with TBS (and kpack) we can use the kubectl CLI and custom resources or the **kp CLI**.
```terminal:execute
command: kp --help
clear: true
```

The first thing we need to do is tell TBS how to access the container registry to upload the built container image.
The **kp secret create** command enables you to create Kubernetes secrets in a more user-friendly way with special support for GCR(--gcr) and Dockerhub(--dockerhub).
```execute
kubectl create ns tbs-demo
REGISTRY_PASSWORD=$CONTAINER_REGISTRY_PASSWORD kp secret create registry-credentials --registry $CONTAINER_REGISTRY_HOSTNAME --registry-user $CONTAINER_REGISTRY_USERNAME -n tbs-demo
```
List all the available Secrets in the namespace via:
```terminal:execute
command: kp secret list -n tbs-demo
clear: true
```

Let's have a look at the different resources used by TBS and kpack.

A **Store** is a cluster level resource that provides a collection of buildpacks. Buildpacks are distributed and added to a store in buildpackages which are docker images containing one or more buildpacks.
```terminal:execute
command: kp clusterstore list
clear: true
```
TBS ships with a curated collection of Tanzu buildpacks for Java, Nodejs, Go, PHP, nginx, and httpd and Paketo buildpacks for procfile, and .NET Core. 
```terminal:execute
command: kp clusterstore status default
clear: true
```

A **Stack** resource is the specification for a cloud native buildpacks stack used during build and in the resulting app image.
```terminal:execute
command: kp clusterstack list
clear: true
```
kp clusterstack status default
A **CustomStack** resource is not available with kpack, and allows users to create a customized ClusterStack from Ubuntu 18.04 (Bionic Beaver) and UBI7/UBI8 non-minimal based OCI images.

A **Builder** uses a ClusterStore, a ClusterStack, and an order definition to construct a builder image.
This order definition determines the order in which groups of buildpacks will be tested during detection. Detection is a phase of the buildpack execution where buildpacks are tested, one group at a time, for compatibility with the provided application source code. 
```terminal:execute
command: kp clusterbuilder list
clear: true
```
```terminal:execute
command: kp clusterbuilder status full
clear: true
```
In addition to to the cluster scoped Builder resource, there is also a namespace scoped equivalent available.

**Image** resources provide a configuration for TBS to build and maintain a Docker image utilizing Tanzu, Paketo, and custom Cloud Native Buildpacks.
Build Service will monitor the inputs to the image resource to rebuild the image when the underlying source or buildpacks have changed.
```terminal:execute
command: kp image --help
clear: true
```
```terminal:execute
command: kp image create --help
clear: true
```

Let's now create our first Image resource that uses a JAR file of a Spring Boot application as source from our local machine. For this we first have to download an example and package the compiled Java code up in a JAR file.
```execute
git clone https://github.com/sample-accelerators/tanzu-java-web-app.git
cd tanzu-java-web-app
./mvnw package
ls target/
cd ..
```

Before we create the Image resource, we have to authenticate with our registry.
```terminal:execute
command: docker login $CONTAINER_REGISTRY_HOSTNAME -u $CONTAINER_REGISTRY_USERNAME -p $CONTAINER_REGISTRY_PASSWORD
clear: true
```
After that we can create the Image resource by specifing the target container-image tag and the path to our JAR file.
```terminal:execute
command: |
    kubectl create ns tbs-demo 
    kp image create tanzu-java-web-app --tag "${CONTAINER_REGISTRY_HOSTNAME}/tap-wkld/tanzu-java-web-app" --local-path tanzu-java-web-app/target/demo-0.0.1-SNAPSHOT.jar -n tbs-demo
clear: true
```

We can have a look at the Images available in a namespace with the kp CLI or kubectl ...
```terminal:execute
command: kp image list -n tbs-demo
clear: true
```
```terminal:execute
command: kubectl get images.kpack.io -n tbs-demo
clear: true
```

... and a closer look via:
```terminal:execute
command: kp image status tanzu-java-web-app -n tbs-demo
clear: true
```
```terminal:execute
command: kubectl describe images.kpack.io tanzu-java-web-app -n tbs-demo
clear: true
```

The creation of a Image resource triggers a new container build.
```terminal:execute
command: kp build list -n tbs-demo
clear: true
```
```terminal:execute
command: kubectl get build -n tbs-demo
clear: true
```

To only show build of a specific image with the kp CLI, you have to add the name as an argument.
```terminal:execute
command: kp build list tanzu-java-web-app -n tbs-demo
clear: true
```

You can get more information with the status command of the kp CLI (or via a kubectl describe) ...
```terminal:execute
command: kp build status tanzu-java-web-app -n tbs-demo
clear: true
```
... and the logs command (kubectl logs) to view the logs of the build process.
```terminal:execute
command: kp build logs tanzu-java-web-app -n tbs-demo
clear: true
```
All those commands will target the latest build by default, you can specify an older build with the "-b" flag.
```terminal:execute
command: kp build status tanzu-java-web-app -n tbs-demo -b 1
clear: true
```
After the build succeeded, it's possible to view the bill of materials of the container image.
```terminal:execute
command: kp build status tanzu-java-web-app --bom -n tbs-demo
clear: true
```

The concept behind an Image resource is not to create one for each change of your application, instead you should update it, if your source code is on your local machine or you are referencing a specific revision manually. If your Image is configured for a git brach it will automatically trigger a new container build for each commit.

To update your container image manually, you can use kp CLI patch command (or kubectl edit).
```terminal:execute
command: kp image patch tanzu-java-web-app -n tbs-demo --local-path .
clear: true
```
```terminal:execute
command: kp build list -n tbs-demo
clear: true
```

Let's now patch our image to use a Git repository as a source.
```terminal:execute
command: kp image patch tanzu-java-web-app --git https://github.com/sample-accelerators/tanzu-java-web-app.git --git-revision main -n tbs-demo
clear: true
```
```terminal:execute
command: kp build list -n tbs-demo
clear: true
```

Build Service **auto-rebuilds image resources** when one or more of the following build inputs change:
- New Buildpack versions are made available via updates to a Cluster Store or on Tanzu Network if automatic dependency updates are turned on.
- There is a new Cluster Stack (ie. base OS image) available, such as full, tiny, or base (manually by the operator or via automatic dependency updates).
- There is a new commit on a branch or tag Tanzu Build Service is tracking.

The REASON field of an Image Build describes why an image rebuild occurred. These reasons include:
- **CONFIG:**- Occurs when a change is made to commit, branch, Git repository, or build fields on the image's configuration file and you run kp image apply.
- **COMMIT:** Occurs when new source code is committed to a branch or tag that Build Service is monitoring for changes.
- **BUILDPACK:** Occurs when new buildpack versions are made available through an updated builder.
- **STACK:** Occurs when a new base OS image, called a run image, is available.
- **TRIGGER:** Occurs when a new build is manually triggered.

You can also manually trigger a rebuild with the same configuration.
```terminal:execute
command: kp image trigger tanzu-java-web-app -n tbs-demo
clear: true
```

To delete an Image, run the image delete command.
```terminal:execute
command: kp image delete tanzu-java-web-app -n tbs-demo
clear: true
```

TBS also supports **image signing** with [cosign](https://github.com/sigstore/cosign).

Let’s clean up our resources before we move on.
```execute
kubectl delete ns tbs-demo
````