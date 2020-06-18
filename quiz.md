# Quiz Questions


### How can you build container images in OpenShift?

* **Docker**
* **Custom build process**
* **Source-to-Image**
* Compiling binaries


### Which statements about containers are correct?

* **An image is needed to run a container, and typically builds on other images**
* Containers provide easy access to system level resources such as memory usage, input devices or network I/O
* **Containers share the same Host OS**
* Updating a library in one container will also change the library in another container
* **From one image you can run many containers**
* **Developers use the same container image in development, testing and production**
* Restarting a container requires to stop services on the Host OS, sometimes even rebooting it
* **Container images are stored in an image registry such as Quay or Docker Hub**


### Which applications are easy to containerize?

* **A PHP content management system**
* A python application which captures photos from a camera
* **A Java backend that connects to a Kafka event stream**
* A network I/O analysis tool


### Which statements about Kubernetes and OpenShift are correct?

* Only one container can run in a Pod
* **Kubernetes helps to orchestrate a large number of containers**
* The developer experience of Kubernetes and OpenShift is the same
* **Services define a logical set of Pods to enable external traffic, load balancing and service discovery**


#### You have to deploy a Node.js application your team is developing to OpenShift. Which approach do you choose?

* Write a custom Dockerfile with a Node.js base image from DockerHub
* **Use the Node.js S2I builder image from OpenShift's Catalog**
* Write a custom Dockerfile with a Node.js base image from Red Hat Container Catalog
* Create a custom Node.js builder image as OpenShift does not support Node.js out-of-the-box


### Which statement is false about typical S2I builds on OpenShift?

* The S2I tool injects source code into a base image to create ready-to-run container images without Docker
* **New builds are usually not triggered when the builder image changes, e.g. due to security fixes**
* In the build step the application is compiled and packaged as a container image and pushed into the OpenShift registry
* In the deployment step the application is made available by starting Pods in which the containers run


### What are typical scenarios for creating a container image with a Dockerfile instead of S2I?

* **Installing new libraries, e.g. with `apt-get install`**
* To share a standardized build process among multiple applications
* **No suitable S2I builder is available**
* To speed up the build process


### Which resources does `oc new-app mysql` create?

Hint: mysql is a container image from Docker Hub

* **DeploymentConfig**
* Route
* **Service**
* **ImageStream**


### What is not true about Kubernetes resources?

* They are specified in YAML or JSON format
* **Pods, services and worker nodes are examples of Kubernetes resources**
* Contain both configuration and current state
* Are scoped by namespaces


###  What is true about networking in OpenShift?

* **An internal IP address is assigned to each container**
* IP addresses of pods are stable, IP addresses of services change frequently
* Pods expose their containers directly to cluster-external traffic
* **Routes define external-facing DNS names and ports for a service**
