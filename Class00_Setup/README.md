# Class 00: Setup

Before starting our Workshop, we have some setting up to do. To begin with, we need to prepare BigQuery for running our queries.

There are two options here: for those who don't want to use a credit card to setup an account, there's the option of using [BigQuery Sandbox](https://cloud.google.com/bigquery/docs/sandbox) which will allow us to run queries in the Web UI for free (given [quotas](https://cloud.google.com/bigquery/docs/sandbox#limits)). Keep in mind though that in this option some exercises and functionalities of this workshop might not work; for the most part it should just be fine though.

In order to fully make use of exercises and functionalities here presented, the second option would be to create an account in Google Cloud Platform (which requires the usage of a credit card). For newcomers, Google offers a $300 Credit [Free-Tier](https://cloud.google.com/free/) which is more than enough to cover all exercises suggested in this workshop.

Still, if you by any chance already used all those credits, as Google offers free usage quotas every month, it's possible to finish this workshop and still be within the quotas specifications.

## GCP Projects

If you created an account in GCP, it's recommended to create a new project in your account to work with BigQuery and set up IAM requirements.

In your [GCP Console](https://console.cloud.google.com), click on "Select Project" on the left-upper coner and then choose "Create New Project" as shown in the image below:

![](./images/gcp_new_project.png)

Give your project an easy and well enough descriptive name for its main purpose (the ID has to be globally unique). For our workshop, calling it something like "bqworkshop" should already be good enough (remember to choose an appropriate organization if the option applies to you):

![](./images/gcp_create_project.png)

After setting up BigQuery, you can head over to its web UI https://console.cloud.google.com/bigquery where we'll be running most of our examples, as shown below:

![](./images/bq_webui.png)

These steps being complete, it's time to start [Class 01](../Class01_Introduction/README.md).

Let's go ;)!
