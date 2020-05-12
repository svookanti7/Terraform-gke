# gke-cluster-terraform
Sets up a managed auto-repairing, auto-upgrading, and auto-scaling on GKE using Terraform with Google Storage as the Terraform Remote Storage backend.

Here’s a brief description of the cluster we will deploy:

It will have a single multi-purpose pool of nodes. If we need specialized ones later, we can add another node pool and set a Kubernetes Taint as appropriate.
The node pool should be managed entirely by GKE. This means it gets automatic upgrades of Kubernetes versions and automatic node repairs (or replacement).
The node pool should also have auto-scaling enabled, because we are not able to forecast how large it needs to be. And we do not want to turn customers away due to lack of resources, right?
------------------------------------------------------------------------------------------------------------------------------------------------------
Prequisites
There are a few things you should do before we get started for real:

Install Terraform
Install kubectl
Install the Google Cloud SDK
Clone the companion repository for this blog post
---------------------------------------------------------------------------------------------------------------------------------------------------------------
Enabling the GKE API and creating a Service Account for Terraform
Terraform interacts with Google Cloud through its API (Application Programming Interface). And for security reasons, 
it first needs to be enabled for your project. So with your shiny new project defined, you can now enable it. Head over to Google Developers Console to enable the GKE API:

Enabling the GKE API
Enabling the GKE API
After hitting the button, you can access the Overview. It will tell you to create some credentials, and that is just what we will do.
--------------------------------------------------------------------------------------------------------------------------------------
For Terraform, we will want credentials of the Service Account type. Let’s head over to the IAM Admin console to do that. 
Hit Create Service Account and fill in the form like this:

Create Service Account dialog in Google Cloud IAM Admin console
Create Service Account dialog in Google Cloud IAM Admin console
Next step is to assign permissions. We want the “Kubernetes Engine Admin” and “Storage Admin” roles.
The Kubernetes one is obvious, but the Storage one is used for Terraform itself. We will configure it to use Google Cloud Storage as the remote state backend later. This means that several operations people can access it later, without file inconsistencies. It should look like this:
Service Account permissions for Terraform
Service Account permissions for Terraform
Finally, in the last step, the most important part is to create a key and downloading it in JSON format. Don’t miss this step!

Creating a key for the Service Account in JSON format
Creating a key for the Service Account in JSON format
Save it to wherever you cloned the repo accompanying this blog post. Call it account.json, because that is what the Terraform files expect. It will not be committed to version control, so don’t worry. Also, keep it safe! Whoever has access to the key will be able to act as the Service Account. If you should lose it, disable the key immediately from the IAM Admin console.
------------------------------------------------------------------------------------------------------------------------------------------------------------
Configuring Google Cloud SDK and Terraform
Not only do we want Terraform to use our Service Account, we want it to store its Remote State on Google Storage. This means that several team members can access the state. Because Terraform state is stored in the cloud, we can still manage our infrastructure even if the computer used for deploying it would go missing or similar.

Creating a bucket and enabling versioning
We will follow the official documentation on Google Storage for how to set up a bucket. Unfortunately, Google Storage console is currently unable to let us enable versioning, which is something Terraform strongly recommends (requires). For that, we need to download and install Google Cloud SDK, giving us access to the gsutil program. Once you have followed the instructions for the Google Cloud SDK, you can type the following command:

gcloud init

This will open a new browser window, asking you to log in. Do so, and answer all the questions in the terminal. Afterwards, you are ready to use the Google Cloud SDK.

The first thing we do with our Google Cloud SDK is enable versioning for our bucket. Type the following, exchanging BUCKET_ID with our value:


$ export BUCKET_ID=...your ID goes here...
$ gsutil versioning set on gs://${BUCKET_ID}
Enabling versioning for gs://gke-from-scratch-terraform-state/...
$ gsutil versioning get gs://${BUCKET_ID}
gs://gke-from-scratch-terraform-state: Enabled
----------------------------------------------------------------------------------------------------------------------------------------------------------------
If your output looks similar (apart from the bucket ID), you are good to go!

Configuring Google Cloud Storage as Terraform remote state
We will set up a Terraform Workspace for each of our deployments. Remember that there could be a reason for us to deploy several replicas of this environment (staging, disaster recovery, etc.). Let’s define one called “production”. There is a file in the accompanying repo called production.tfvars, which you can inspect, modify, and use.

Most values in the Terraform files have been written such that you just need to modify the production.tfvars file. However, one hard-coded value is in google.tf, on the line where the bucket is specified. The reason for this unfortunate limitation is technical: Terraform does not allow variables in that particular section. So change it in your google.tf file and then proceed.
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
To configure Terraform, we run:


$ export ENVIRONMENT=production
$ terraform workspace new ${ENVIRONMENT}
Created and switched to workspace "production"!

You're now on a new, empty workspace. Workspaces isolate their state,
so if you run "terraform plan" Terraform will not see any existing state
for this configuration.
$ terraform init -var-file=${ENVIRONMENT}.tfvars

Terraform gives us a lot of output, but among others, it will say that the “google” provider has been configured and that the “gcs” (remote state backend) provider has been set up. Success!

Using Terraform to create a GKE cluster
When you create a GKE cluster, you automatically wind up with an initial node pool. While that sounds nice, the problem is that we cannot manage it and set the configuration we want (auto-upgrades, auto-repair, and auto-scaling). So we will ask Terraform to create a cluster, then delete the initial node pool. It will then be asked to create a new node pool (that has the configuration options set), and then attach it to the cluster. This is so common (and the recommended approach) that Terraform has a special option for it.

Giving our Service Account required permissions
If we were to try to deploy now, we’d face a permissions error. We need to give our service account more permissions, and the roles/editor role. To do so, issue the following command:


$ export PROJECT=gke-from-scratch
$ export SERVICE_ACCOUNT=terraform
$ gcloud projects add-iam-policy-binding ${PROJECT} --member serviceAccount:${SERVICE_ACCOUNT}@${PROJECT}.iam.gserviceaccount.com --role roles/editor

Deploying the GKE cluster
Now that we have done all the preparations, it is almost embarrassingly easy to deploy the cluster:


$ terraform apply -var-file=${ENVIRONMENT}.tfvars

Answer “yes” when prompted, and after a few minutes, you have a cluster! You can head over to the Kubernetes Clusters list in Google Cloud console and see it:

Kubernetes cluster listing in Google Cloud console
Kubernetes cluster listing in Google Cloud console
Conclusion
In this post, we showed how to deploy an auto-repairing, auto-updating, auto-scaling cluster from scratch on Google Container Engine (GKE) using Terraform. The setup made use of a custom Service Account for security, and used Terraform Workspaces to make multi-cluster management possible.

When your Kubernetes journey brings you to the point where a fully-managed and compliant Kubernetes cluster makes sense, don’t hesitate to contact us at Elastisys.
---------------------------------------------------------------------------------------------------------------------------------------------------------------
