# Lab 5: Add Services with the CLI and Infrastructure-as-Code

In this lab you will learn how to create a MySQL database in OpenShift 4.3 and deploy a Java backend application with a custom Dockerfile using the OpenShift Container Platform CLI and YAML resource definitions.

## MySQL database

First you will create a MySQL database in your Open Shift project.

If you are not connected to your OpenShift cluster, please re-run the commands from the `Log into the cluster with the CLI` section of Lab 1.

For convenience store the database name in a variable:

```bash
export DB_NAME=health_data
echo $DB_NAME
```

### Create the MySQL database

To store the data of our application between restarts we create a MySQL database with persistent storage using the `mysql-persistent` OpenShift template:

```bash
oc new-app \
--template=mysql-persistent \
--name=mysql \
-p MYSQL_DATABASE=$DB_NAME
```

![Created mysql](lab-05-images/db-created.png)

Check that the database *deployment* was created:

```bash
oc get deploymentconfigs
```

![List mysql deployment](lab-05-images/db-deployment-list.png)

```bash
oc describe deploymentconfig mysql
```

![Mysql deployment details](lab-05-images/db-deployment.png)

Check that a *persistent volume claim* was created to store the MySQL database:

```bash
oc get pvc
```

![Mysql pvc](lab-05-images/db-pvc.png)

Check that a *secret* where the database credentials are stored was created:

```bash
oc get secrets
```

![Mysql secret list](lab-05-images/db-secret-list.png)

```bash
oc describe secret mysql
```

![Mysql secret](lab-05-images/db-secret.png)

Check that the database is ready:

```bash
oc get pods
```

![Mysql pods deploying](lab-05-images/db-pods-deploying.png)

**Wait until only one `mysql-...` pod is left with status `Running` and `1/1` containers are ready!**

![Mysql pods deployed](lab-05-images/db-pods-deployed.png)

Tip: You can also check the status of the applications with:

```bash
oc status
```

### Construct the URL of the MySQL service

To communicate between pods within the cluster and to avoid hard-coding IP addresses in applications, each service gets its own hostname by OpenShift's internal DNS service.

The hostnames follow the format: `<service-name>.<namespace>.svc.cluster.local:<port>`

You can retrieve the parameters needed to construct a hostname for a service with:

```bash
oc describe service mysql
```

![Mysql service](lab-05-images/db-service.png)

Construct the database URL with using these parameters and save it in the `DB_URL` variable:

```bash
export DB_URL=mysql://<Name>.<Namespace>.svc.cluster.local:<Port>
echo $DB_URL
```

![Mysql URL](lab-05-images/db-url.png)

### Store the JDBC-URL in a secret

Secrets provide a mechanism to store and retrieve sensitive information such as passwords in OpenShift.

Next you will create a secret that stores a JDBC connection URL your backend can use to connenct to the MySQL database:

```bash
oc create secret generic mysql-jdbc \
    --from-literal="jdbc-url=jdbc:$DB_URL/$DB_NAME?sessionVariables=sql_mode=''"
```

Check that the secret was created:

```bash
oc get secrets
```

![Created secret](lab-05-images/db-secret-created.png)

### Import the database schema

Although the MySQL database is running in your cluster, it is still pretty empty. Next you will learn how to connect to the MySQL pod and run MySQL commands from within the container to populate the database with its first tables.

Therefore you first need to identify the name of the pod running your MySQL database. You can view a list of pods in your project with:

```bash
oc get pods
```
  
Then, open a remote shell session to the pod with the following command, replacing `$POD_NAME` with the name of your `mysql-...` pod:

```bash
oc rsh $POD_NAME
```

Within the pod's bash shell you can now run the `mysql` commad to start an interactive MySQL session:

```bash
mysql -u $MYSQL_USER -p$MYSQL_PASSWORD -h $HOSTNAME $MYSQL_DATABASE
```

*Note: These environment variables such as `MYSQL_USER` are passed to the `mysql`-pod by OpenShift. When you create your backend service in the next section you will see how you can specify environment variables for your pods.*

![MySQL bash session](lab-05-images/db-shell.png)

Next copy, paste and execute the commands from [https://github.com/IBM/example-health-jee-openshift/tree/master/example-health-api/samples/health_schema.sql](https://github.com/IBM/example-health-jee-openshift/tree/master/example-health-api/samples/health_schema.sql) within the bash session. The last lines of the output should look as following:

![MySQL created DB schema](lab-05-images/db-schema-created.png)

To verify that the schema has been created, run this command to show the list of exisiting tables:

```bash
show tables;
```

It should show the following list:

```bash
+-----------------------+
| Tables_in_health_data |
+-----------------------+
| Allergies             |
| Appointments          |
| MESSAGE               |
| Observations          |
| Organizations         |
| Patients              |
| Prescriptions         |
| Providers             |
| SEQUENCE              |
+-----------------------+
9 rows in set (0.00 sec)
```

Finally, close the session with `Ctrl + D` or type `exit` twice

---

## Backend service

After the MySQL database is now ready, you will deploy a backend service that connects to the database and frontend.

### Create a GitHub repository for the backend service

First, fork the Java backend GitHub repository:

[https://github.com/IBM/example-health-jee-openshift](https://github.com/IBM/example-health-jee-openshift)

### Update the Dockerfile

To build a container image on OpenShift we need to replace the Dockerfile in `example-health-api/Dockerfile` with the following Dockerfile (you can use GitHub's web editor to change the file, see Lab 4):

```Dockerfile
# 1. Package the application using a maven-based builder image

FROM maven:3.6-jdk-8 as builder

COPY . .
RUN mvn package


# 2. Build the Open Liberty image

FROM openliberty/open-liberty:javaee8-ubi-min

ENV INSTALL_DIR /opt/ol/wlp/
ENV SERVER_DIR /opt/ol/wlp/usr/servers/defaultServer/

# Add MySQL-Connector
RUN curl -o ${INSTALL_DIR}lib/mysql-connector-java-8.0.16.jar https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.16/mysql-connector-java-8.0.16.jar
COPY liberty-mysql/mysql-driver.xml ${SERVER_DIR}configDropins/defaults/

# Copy WAR-file from builder image
COPY --from=builder target/health-api.war ${SERVER_DIR}apps

# Copy configs
COPY liberty/server.xml $SERVER_DIR
COPY liberty/jvm.options $SERVER_DIR

# Expose port 9080 where the application runs on
EXPOSE 9080
```

This Dockerfile first packages the JEE application using a `maven` container image. Then it builds the actual container that runs the application based on an Open Liberty base image.

**Make sure to commit these changes to your Git repository!**

## Deploy the backend with YAML resource definitions

Instead of letting OpenShift create the resurces for you with the Web Console, you will create them with YAML resource definitions and the OpenShift CLI yourself.

Defining the infrastructure as code is helpful for re-using applications and promoting them accross different environments.

### Build step

On your computer create a new file with the YAML below named `build-resources.yaml`

Replace `<YOUR GITHUB URL>` with the URL of your GitHub repository

```yaml
apiVersion: v1
kind: BuildConfig
metadata:
  name: patient-api
  labels:
    app: patient-api
    component: backend
spec:
  output:
    to:
      kind: ImageStreamTag
      name: patient-api:latest
  source:
    type: Git
    git:
      ref: master
      uri: <YOUR GITHUB URL>
    contextDir: /example-health-api
  strategy:
    type: Docker
  triggers:
  - type: ConfigChange
---
apiVersion: v1
kind: ImageStream
metadata:
  name: patient-api
  labels:
    app: patient-api
    component: backend
spec:
  tags:
  - name: latest
```

To create these resources in OpenShift run:

```bash
oc apply -f build-resources.yaml
```

![Apply build resources](lab-05-images/build-resources-created.png)

This creates a *build config* and *image stream* in your OpenShift project and starts your first build.

You can start another build with:

```bash
oc start-build patient-api
```

![Build started](lab-05-images/build-started.png)

Check the logs of the first build by first retrieving the name of the pod and then retrieving the logs of the pod:

```bash
oc get pods

oc logs patient-api-1-build
```

You can check the status of a build with:

```bash
oc get builds
```

The output should show that the `patient-api-...` build is `Complete`:

![Build completed](lab-05-images/build-completed.png)

### Deployment step

After building the container image we need to deploy the image.

Therefore create a new file with the YAML below named `deployment-resources.yaml`:

```yaml
apiVersion: v1
kind: DeploymentConfig
metadata:
  name: patient-api
  labels:
    app: patient-api
    component: backend
spec:
  replicas: 1
  selector:
    deploymentconfig: patient-api
  template:
    metadata:
      labels:
        app: patient-api
        component: backend
        deploymentconfig: patient-api
    spec:
      containers:
      - name: patient-api
        image: patient-api:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 9080
        env:
        - name: ENV_MYSQL_URL
          valueFrom:
            secretKeyRef:
              name: mysql-jdbc
              key: jdbc-url
        - name: ENV_MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mysql
              key: database-user
        - name: ENV_MYSQL_PWD
          valueFrom:
            secretKeyRef:
              name: mysql
              key: database-password
      restartPolicy: Always
  triggers:
  - type: ConfigChange
  - type: ImageChange
    imageChangeParams:
      automatic: true
      containerNames:
      - patient-api
      from:
        kind: ImageStreamTag
        name: patient-api:latest
---
apiVersion: v1
kind: Service
metadata:
  name: patient-api
  labels:
    app: patient-api
    component: backend
spec:
  ports:
  - name: 9080-tcp
    port: 9080
    protocol: TCP
    targetPort: 9080
  selector:
    deploymentconfig: patient-api
---
apiVersion: v1
kind: Route
metadata:
  name: patient-api
  labels:
    app: patient-api
    component: backend
spec:
  port:
    targetPort: 9080-tcp
  to:
    kind: Service
    name: patient-api
    weight: 100
```

Then again apply these resource definitions to OpenShift with:

```bash
oc apply -f deployment-resources.yaml
```

This creates a *deployment config*, *service* and *route* in your OpenShift project.

![Apply deployment resources](lab-05-images/deployment-resources-created.png)

Verify that the deployment was successful:

```bash
oc get deploymentconfigs
```

```bash
oc get pods
```

Make sure that one pod is running.

---

## Add a patient with the OpenAPI Browser

The backend application comes with a browser UI for its OpenAPI specification.

To open the UI, oObtain the hostname assigned to the route:

```bash
oc get route patient-api
```

Navigate in your browser to `<hostname>/openapi/ui/`. An OpenAPI specification of the endpoints and operations supported by the Java EE application is shown:

![OpenAPI UI](lab-05-images/openapi-ui.png)

Populate database with some data by clicking on `/resources/v1/generate` (last item in the list) and then `Try it out`.

Paste the following JSON into the textfield:

```json
{
  "patients":[
    {"Id":"498c0c0f-2ef7-4a25-a5b3-dca47472e3fa","BIRTHDATE":"2011-01-16","DEATHDATE":"","SSN":"999-99-5908","DRIVERS":"","PASSPORT":"","PREFIX":"","FIRST":"Karri995","LAST":"McDermott739","SUFFIX":"","MAIDEN":"","MARITAL":"","RACE":"white","ETHNICITY":"swedish","GENDER":"F","BIRTHPLACE":"Santa Barbara  California  US","ADDRESS":"508 Donnelly Underpass Unit 8","CITY":"Costa Mesa","STATE":"California","ZIP":"92614"}
  ]
}
```

Then click `Execute` to send the request.

![Data ingestion response](lab-05-images/openapi-response.png)

---

## Open the front-end and point it to the API service

To open the front-end retrieve its URL with `oc get route`:

```bash
oc get route patient-ui
```

Open the URL in your web browser and click on `settings`:

![Backend settings](lab-05-images/ui-settings-nodejs.png)

Here you can change the backend URL to point to your just deployed `patient-api` service.

Since both services run within the OpenShift cluster they can communicate with each other using OpenShift's internal DNS service.

Run `oc describe service` to retrieve the information neccessary to construct the backend URL:

```bash
oc describe service patient-api
```

Fill `<Name>`, `<Namespace>` and `<Port>` in the URL and paste it into the `enter backend url here` field:

`http://<Name>.<Namespace>.svc.cluster.local:<Port>/resources/v1/`

Please make sure you add the trailing `/` to your URL!

![Backend settings with new URL](lab-05-images/ui-settings-url.png)

Then click on a whitespace next to the URL field, wait a few seconds to make sure the URL is updated and then click on `java` to change the backend type.

---

## Login to see the personal info

On the Login screen you can now login with username `karrim` and password `karrim`. These were automatically generated by the backend based on the first and last name of the user we added.

After logging in you will be able to see the user's personal information, which is retrieved by the `patient-ui` service from the `patient-api` and `mysql` services:

![Website Home](lab-05-images/ui-home.png)

---

## Optional: Explore the application with the OpenShift CLI and Web Console

Please take some time to further explore your microservices app using the OpenShift Container Platform CLI and Web Conosole.

You can retrieve a list of all CLI commands with:

```bash
oc --help
```

You can view YAML resource definitions with:

```bash
oc get deploymentconfig <my-deploymentconfig> -o yaml
oc get pod <my-pod> -o yaml
```

What properties did OpenShift populate that were not defined in the YAML resource definitions?

## Conclusion

In this lab we explored how:

* A MySQL database is created with `oc create app`
* Data is stored in `Persistent Volumes`
* `Secrets` are used to store sensitive information
* Information is passed with `environment variables` into containers
* Commands are ran within a container using `oc rsh`
* Two pods communicate with each other using a `Service` and OpenShift's `internal DNS`
* Infrastructure-as-Code works with `YAML resource definitions` and `oc apply`
