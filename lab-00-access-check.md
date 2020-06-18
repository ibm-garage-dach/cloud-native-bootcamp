# Lab 0: Cloud and Cluster access check

Before we can start with any labs, the access to the IBM Cloud and the OpenShift cluster have to be validated.

## Log into the IBM Cloud and cluster UI

Log into [IBM Cloud](https://cloud.ibm.com/login).

Navigate to **Menu** &rarr; **OpenShift** &rarr; **Clusters** as shown below.

![Open OCP Cluster](lab-01-images/open-ocp-clusters.png)

You should see a list of cluster, containing at least one OpenShift cluster with version 4.3.

![List of OCP clusters](lab-00-images/cluster-list.png)

Select a cluster. You should see the overview of the cluster, with a summary about the cluster, worker node status and some cluster insights.

![cluster details](lab-00-images/cluster-details.png)

Next, open the **OpenShift Web Console**, by clicking on the blue button on the top right.

You should be presented the OpenShift landing page:

![OCP Landing Page](lab-01-images/ocp-landing.png)

## Log into the cluster with the CLI

Now let's log into the cluster with the cli to perform the first operations on the cluster.

Therefore, click on your name in the top right corner and then select **Copy Login Command**. Click on **Display Token** and copy the command below the *"Log in with this token"* section.

![OCP Copy Login Command](lab-00-images/copy-login.png)

Open up a local terminal, paste the login command and execute it.

```bash
$ oc login https://... --token=XXX

Logged into "https://c100-e.eu-de.containers.cloud.ibm.com:30026" as "IAM#<your account>" using the token provided.

You have access to '<number of projects>' projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
```

If all of this worked, you are ready to start with the labs.
