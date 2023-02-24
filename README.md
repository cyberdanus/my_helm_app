## Urban Task

This repo contains a [Terraform](https://www.terraform.io) code for running a Kubernetes cluster on [Amazon Web Services (AWS)](https://aws.amazon.com/) using [Amazon Elastic Kubernetes Service (EKS)](https://aws.amazon.com/eks/) and GitHub Action to Build, Publish, and Deploy an application. You can see the application metrics in Prometheus.

## Table of contents

---

- [Urban Task](#urban-task)
- [Table of contents](#table-of-contents)
- [Quickstart](#quickstart)
  - [Building Infrastructure in AWS](#building-infrastructure-in-gcp)
    - [Building Infrastructure by pipeline](#building-infrastructure-by-pipeline)
    - [Building Infrastructure step by step](#building-infrastructure-step-by-step)
  - [Terraform code](#terraform-code)
  - [How use GitHub Actions to deploy application](#how-use-github-actions-to-deploy-application)
    - [Add GitHub Repository Secrets](#add-github-repository-secrets)
    - [Deploy App](#deploy-app)
    - [Workflow Steps](#workflow-steps)
    - [Docker image name](#docker-image-name)
  - [**Deploy configuration**](#deploy-configuration)
  - [Changes in the application](#changes-in-the-application)
- [Destroy infrustructure](#destroy-infrustructure)
- [Homework task for Urban](#homework-task-for-urban)
  - [Requirements](#requirements)
  - [What gets evaluated](#what-gets-evaluated)
- [List of decisions/compromises](#list-of-decisionscompromises)

---

## Quickstart

Please review the `Requirements` before starting.

<details><summary>Requirements</summary>

**Requirements**

* Terraform and kubectl are [installed](#software-dependencies) on the machine where Terraform manifests is executed.
* The Compute Engine and Kubernetes Engine APIs are [active](#enable-apis) on the project you will launch the cluster in.

**Amazon Cloud Account**
- You have to loginin your Amazon Cloud Account
- [Add billing on this Project](https://aws.amazon.com/premiumsupport/knowledge-center/tags-billing-cost-center-project/)

**Software Dependencies**
- [kubectl](https://github.com/kubernetes/kubernetes/releases) >= 1.26.x

**Terraform and Plugins**
- [Terraform](https://www.terraform.io/downloads.html)    >= 1.3.x
- [Terraform Provider for AWS] [terraform-provider-aws](https://registry.terraform.io/providers/hashicorp/aws/latest) >= 4.55.x

</details></br>

### Building Infrastructure in AWS

AWS Cloud Account
  * You have to login in your [AWS Cloud Account](https://console.aws.amazon.com/console/home?nc2=h_ct&src=header-signin)
  * [Add billing on this Project](https://aws.amazon.com/premiumsupport/knowledge-center/tags-billing-cost-center-project)

#### Building Infrastructure by pipeline

We can use the script `start.sh` from backend dir to create AWS S3 Terraform remote state backend for Infrustructure.  
You have to run the script from folder [backend](/backend/).  

  * Before start you have to connect to AWS cloud CLI in terminal:
    * `aws configure` - connect to your AWS Account with existing credentials
  * Clone repository
    * `git clone https://github.com/cyberdanus/eks_deployment.git`
  * Add GitHub Secrets to Action in your Repository
    * The necessary secrets are AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY. Also we can use TELEGRAM_TO and TELEGRAM_TOKEN secrets for ChatOPS
    * You have to [add this secrets](#how-use-github-actions-to-deploy-application) in your GitHub Repository

</br>

#### Building Infrastructure step by step

The second way is to build infrastructure step by step. (*Tested on Ubuntu 22*)

<details><summary>Use AWS Cloud CLI</summary>

* Go to [AWS Cloud Console](https://console.aws.amazon.com/console/home?nc2=h_ct&src=header-signin) and autorize.
* [Install the AWS cloud CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [Install the Terraform CLI](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)


```bash
# install AWS cloud CLI for Ubuntu
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# install Terraform CLI for Ubuntu
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common

# Connect to AWS CLI with existing credentials
aws configure
```

# Clone GitHub repository
git clone https://github.com/cyberdanus/eks_deployment.git

# Create Remote tfstate S3 Bucket to save tfstate-files
You have to run the terraform cli from folder [backend](/backend/).  
```bash
# Go to folder `backend` and run commands:
terraform init
terraform apply -auto-approve


```
# Create infrastructure with Terraform
You have to run the terraform cli from folder [eks_dep](/eks_dep/).  
```bash
# Go to folder `eks_dep` and run commands:
terraform init
terraform apply -auto-approve


```

</details>

<details><summary>Connect to EKS Cluster</summary>

Then you have to Connect to EKS Cluster

```bash
# Update the kubectl configuration:
aws eks --region $(terraform output -raw region) update-kubeconfig --name $(terraform output -raw cluster_name)

# test connetion
kubectl get nodes
```

</details></br>


### Terraform code

Terraform code in folders:
* `tf-code/infrustructure` - create infrustructure: Google Kubernetes Engine (GKE) Cluster, Network with Firewall and rules, Google Container Regygistry (GCR), and Service Account
* `tf-code/infrustructure` - Create Kubernetes Namespaces: test, dev, prod. Deploy Nginx Ingress and Prometheus with Grafana by helm deploy.
* `tf-code/modules/service-account` - by this Module create Service Account 

<details><summary>Infrustructure</summary>

* **container-registry.tf** - GCR to store docker images
* **k8s-cluster.tf** - GKE CLuster
* **main.tf** - TF requerments: backend, requiered providers and providers (google, kubernetes, helm), Datasources
* **network.tf** - VPC, Subnet, Router, NAT, Firewall
* **outputs.tf** - Output data
* **service-account.tf** - Service account to create GKE Cluster and Deploy by GitHub Action. Used module [modules/service-account](tf-code/modules/service-account) to create Service Account and add Roles. [Module documentation](tf-code/modules/service-account/README.md)
* **variables.tf** - Used variables. Set variables in file like [infr.tfvars](tf-code/variables/infr.tfvars)

</details>

<details><summary>Deploy</summary>

* **ingress.tf** - Ingress controller deploy in Namespace `ingress`
* **main.tf** - TF requerments: backend, requiered providers and providers (google, kubernetes, helm), Datasources
* **namaspaces.tf** - Create Namespaces in Cluster: `test`, `dev`, `prod`
* **prometheus.tf** - Prometheus deploy in Namespace `metrics`
* **variables.tf** - Used variables. Set variables in file like [deploy.tfvars](tf-code/variables/deploy.tfvars)

</details>

<details><summary>Module service-account</summary>

* **main.tf** - Create Service Account and Add Roles, Create SA-KEY
* **outputs.tf** - Output data
* **variables.tf** - Used variables. Set variables in file like [infr.tfvars](tf-code/variables/infr.tfvars)

</details></br>



### How use GitHub Actions to deploy application

When infrustructure ready you can use [GitHub Actions](https://github.com/cyberdanus/eks_deployment/blob/main/.github/workflows/main.yml) with workflow dispatch to deploy and destroy Kubernetes Cluster with ArgoCD wich inplements GitOPS strategy of application deployment.

</br>

#### Add GitHub Repository Secrets

GitHub Secrets link like this: `https://github.com/<Your-Account-Name>/<Your-Repository>/settings/secrets/actions`

  * **AWS_ACCESS_KEY_ID** - Service Account Key to connect in Cluster
  * **AWS_SECRET_ACCESS_KEY** - Your `project_id` in Google Cloud
  * **TELEGRAM_TO** - Cluster Name
  * **TELEGRAM_TOKEN** - Region of your Cluster
  * **SLACK_WEBHOOK**` - [Webhook URL](https://api.slack.com/apps/A02MHFFJK26/incoming-webhooks?) to connect in [Slack API](https://api.slack.com) and send messages
  * **SLACK_BOT_TOKEN** - Region of your Cluster   
  * **DOCKERHUB_TOKEN** - Region of your Cluster
  * **DOCKERHUB_USERNAME** - Region of your Cluster

<details><summary>Screenshots and Commands to get GitHub Repository Secrtets</summary>

</br>

* You can Get Secrets by the script [output.sh](scripts/output.sh). You have to run the script from folder `scripts\`.

* Example of output from script:

![start_output](documentation/pics/start_output.png)

* GitHub Secrets link like this: `https://github.com/<Your-Account-Name>/<Your-Repository>/settings/secrets/actions`
* Screenshot from GitHub Repository Secrets page

![repo_secrets](documentation/pics/repo_secrets.png)

</details></br>


#### Deploy App

<details><summary>Deploy App</summary>

You have to go in [GitHub Actions page](https://github.com/Aleh-Mudrak/urban/actions/workflows/build-push.yml) and run `Build and Deploy to GKE` like on picture bellow.

![Build and Deploy to GKE](documentation/pics/gha.png)

* Choose `Environment` (test|dev|prod)
* And `Replicas` of the application (1-5)

</br>

#### Workflow Steps
* **Checkout** - Clone GitHub repository
* **Check_input_Variables** - Check entered data on this step  
* **Slack_Notification_Start** - After that you recieve message in Slack about Start deploy and initial parameters on step  
* **Setup_gcloud** - Setup gcloud CLI and Configure Docker to use the gcloud command-line tool as a credential
* **get_gke_credentials** - Get the GKE credentials so we can deploy to the cluster
* **Setting_Environment_Variables** - Configure Setting Environment Variables to Build, Push, and Deploy the application
* **Build** - Build the application 
* **Publish** - Push to GCR this application Docker image.  
* **Deploy** - Deploy in Cluster this application.
* **Slack_Notification_Finish** - Last step send message to Slack with deploy results and link.

</br>

#### Docker image name
Docker image has image name: 
* `gcr.io/$PROJECT_ID/$APP_NAME:$PROJECT_VERSION`

Where 
* **PROJECT_ID** - Google Cloud ProgectID
* **APP_NAME** - Application Name
* **PROJECT_VERSION** - Created from `branch_name-commit_hash`:
  * **branch_name** - Get from started GHActions brunch
  * **commit_hash** - Short Commit Hash

</details></br>

### **Deploy configuration**

Deploy configuration files you can find in folder [application/deploy-app/](application/deploy-app/)

* `deploy.yml` - Deploy the application
* `ingress.yml` - Ingress service to connect the application from the Internet
* `promMetrics.yml` - Deploy a service-monitor to get metrics from the application
* `service.yml` - Service to connect the applicastion pods


</br><details><summary>Deploy results</summary>

![Slack output](documentation/pics/Slack_Output.png)
![Application on web](documentation/pics/http.png)
![Application prod diffirent Pods](documentation/pics/prod_hhtp.png)
![Pods in Lens](documentation/pics/LensPods.png)
![Metrics in Prometheus](documentation/pics/prometheus_metrics.png)
![Metrics in Grafana](documentation/pics/grafana.png)

</details></br>

<details><summary>Show the application the Internet</summary>

Add in your hosts file string like that: `34.69.160.165 taskurban.com`  
Command to change in the Linux: `sudo vim /etc/hosts`


Where 
* `34.69.160.165` - IP address from Slack message;
* `taskurban.com` - URL from Slack message.


![Slack message](documentation/pics/Slack_Output.png)

</details></br>


### Changes in the application


* Was added string in `application/package.json` file to run application by command `npm start`
  * String 7: `"start": "node app/index.js",`
* Added Prometheus-metrics code in file `application/app/index.ts` to get metrics in Prometheus
  * String 5-32:

<details><summary>added code to application/app/index.ts</summary>

```ts
const express = require('express')
const metrics = require('express-prometheus-metrics')
const app = express();

app.use(
  metrics({
    // The route to expose the metrics on
    metricsPath: '/metrics',

    // How often prometheus should collect the metrics
    interval: 60 * 1000,

    // Any routes that should be ignored
    excludeRoutes: [],

    // Percentiles for request duration summary
    requestDurationBuckets: [0.5, 0.9, 0.95, 0.99],

    // Time buckets for request duration histogram
    requestDurationHistogramBuckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],

    // Size buckets for request
    requestSizeBuckets: [5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000, 10000],

    // Size buckets for response
    responseSizeBuckets: [5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000, 10000],
  }),
)
```

</details></br>

* Created Dockerfile to build image
  * Added commands for Prometheus metrics:
    * `RUN npm add express-prometheus-metrics`
    * `RUN npm add pkginfo`



## Destroy infrustructure

To destroy infrastructure you can use the script [destroy.sh](scripts/destroy.sh) in folder `scripts`.  
You have to run the script from folder `scripts/`.  
(*it takes about 15-20 minutes*)



---

## Homework task for Urban

* [Application and Task](documentation/hw)

The goal of the task is to demonstrate how a candidate can create an environment with terraform. You should commit little and often to show your ways of working

### Requirements

- The environment should get created in Google Cloud Platform
- Create a VPC native Kubernetes cluster
- Host the provided Node.js application provided in the `app` folder in the created cluster with 3 replicas
- Expose the provided application to the public internet
- Include at least 1 custom module in Terraform
- Add the prometheus-client to the provided application and expose one metric on a `/metrics` endpoint
- Write down some thoughts about what compromises you've applied (if any) and how would you like to improve the solution

### What gets evaluated

- Code quality
- Solution architecture
- Whether the code is "production-ready" (i.e. the environment starts and works as expected)


---

## List of decisions/compromises

Any solution can be improved, but usually we don't have free time for this and we have to choose a more effective way to solve our tasks. In this task, I created the GKE infrastructure and described two ways to deploy it, and added scripts to get variables for GitHub Actions and to destroy it. I prefer to create easy-to-understand solutions by adding comments to the code and documentation where possible.

* The folders in the repo have been sorted and moved by category and logic.
* All parameters were in variables.tf as default.
  * Not important parameters were deleted from `infr.tfvars`
    * Cluster parameters
    * Network parameters
    * Service Account parameters
  * Bash script get initial parameters from `infr.tfvars`
* Used for_each to create multiple node pools in a cluster.
* Used for_each to create multiple firewall rules.
* Data parameters in the `main.tf` file used to connect in the Cluester on the step Deploy.
* Output data the same used to connect in the Cluster on the step Deploy and in the GitHub Actions.
* Prometheus scrape has been resolved. Issue was in the service labels.

![Prometheus metrics in Grafana](/documentation/pics/prometheus_metrics2.png)

Compromises:
* Start scripts can be improved:
  * Get variables from Google Secret Manager;
  * Add Secret GKE_SA_KEY in the GitHub Repository;
  * Add more checks
* Terraform:
  * Terraform Cloud is good solution to use with a GitHub repository;
  * The application and the GH Action have to be in one repo, TF-code in another;
  * TF-code Infrustructure and Deploy have to separate to diffirent git repository;
  * Can add output variables in Deploy part;
  * Can add option to disable deploy Prometheus;
  * Firewall rules can be moved to the Deploy TF-code part;
  * Can add more modules: 
    * Create GKE Cluster and Nodes; 
    * Network with VPC, Subnet, NAT, and Router; 
    * Firewall
  * You can use Terragrunt if you will use a lot of GKE Clusters .
  * Variables in Terraform code can be added into the objects.
* GitHub Actions can be improved with:
  * steps: test-application, cash, deploy by git tag-version;
  * Helm charts;
  * Some Terraform Secrets can be moved to GitHub Secrets by GH CLI.
* Prod and test+dev deploy have to be in different Clusters.
