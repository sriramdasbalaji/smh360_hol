# SmartHotel360-AKS- DevSpaces

## Overview
**SmartHotel360** is a fictitious smart hospitality company showcasing the future of connected travel.
The heart of this application is the cloud – best-in-class tools, data platform, and AI – and the code is built using a microservice oriented architecture orchestrated with multiple Docker containers. There are various services developed in different languages: .NET Core 2.0, Java and Node.js. These services use different data stores like SQL Server, Azure SQL DB, Azure CosmosDB, and Postgres.

In production, all these microservices run in a Kubernetes cluster, powered by Azure Container Service (ACS).

## What’s covered in this lab?

In this lab, you  will
* Create Azure Service Principal
* Provision the required Azure Resources includes Azure ACR, AKS, Log analytics etc..

* Build the images of **SmartHotel360** back-end APIs and Push them to Azure ACR.
* Pull the API images form ACR and run in AKS cluster
* Deploy the **SmartHotel360** public website



## Prerequisites
* Bash command line, which can be accomplished natively on Mac or Linux or using [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/install-win10) on Windows.
* [Docker](http://www.docker.com/) to build the containers.
* [Visual Studio 2017 Preview](https://www.visualstudio.com/vs/preview/) with the ASP.NET Web workload installed.
* [Azure Dev Spaces Extension](https://docs.microsoft.com/en-us/azure/dev-spaces/get-started-netcore-visualstudio#get-the-visual-studio-tools) for Visual Studio.
* [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
* [Kubernetes CLI](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

*  **Microsoft Azure Account**: You will need a valid and active Azure account for the Azure labs. If you do not have one, you can sign up for a [free trial](https://azure.microsoft.com/en-us/free/){:target="_blank"}

   * If you are a Visual Studio Active Subscriber, you are entitled for a $50-$150 credit per month. You can refer to this [link](https://azure.microsoft.com/en-us/pricing/member-offers/msdn-benefits-details/) to find out more including how to activate and start using your monthly Azure credit.

   * If you are not a Visual Studio Subscriber, you can sign up for the FREE [Visual Studio Dev Essentials](https://www.visualstudio.com/dev-essentials/)program to create **Azure free account** (includes 1 year of free services, $200 for 1st month).

## Setup the environment
### **Clone the repository**
Clone the source repository from https://github.com/Microsoft/SmartHotel360-AKS-DevSpaces-Demo.git

### **Create Azure Service Principal** :
 If you already have a service principal, you can re-use it, and if you don't and create one for this demo, you can re-use it to create other AKS clusters in the future. 

1. Login to your Azure subscription and run the below command in [Azure Cloud Shell](https://docs.microsoft.com/en-us/azure/cloud-shell/overview) to create a service principal.

   `az ad sp create-for-rbac --name ServicePrincipalName --password PASSWORD`
   
   ![](images/serviceprincipal.png)

1. Take a note of the output. We will require this details later.

### **Provision the Azure Resources** :
In this step you'll create all of the Azure resources required by the demo. This consists of an AKS Cluster and an Azure Container Registry (ACR) instance. The AKS Cluster is pre-configured to use Microsoft Operations Management Suite (OMS) and Log Analytics to enable the rich Container Health Dashboard capabilities.

1. Install the**Azure Dev Spaces** preview extension for the Azure CLI by entering the following command in the terminal.

   ```bash
   az extension add --name dev-spaces-preview`
   ```
1. Open the `setup\00-set-vars.sh` file in a text editor.
1. Set the variables in the script below (the exports).

   ```bash
   export AKS_SUB=<azure subscription id> \ 
   export AKS_RG=<resource group name> \ 
   export AKS_NAME=<AKS cluster name> \ 
   export ACR_NAME=<Azure Container Registry name> \ 
   export AKS_REGION=eastus \ 
   export SPN_CLIENT_ID=<service principal app id> \ 
   export SPN_PW=<service principal password>
   ```

   Save the file once you're happy with your edits. If you open a new instance of your terminal window or close before ending the process, you can re-run `setup/00-set-vars.sh ` to reset the environment variables the other scripts will use.

   {% include note.html content= "The only regions in which AKS and Azure Dev Spaces are currently supported are Canada East and Eaat US. So when creating a new AKS cluster for this scenario use either **canadaeast** or **eastus** for the **AKS_REGION** variable." %}

1. Open a bash terminal. CD into the setup folder of this repository.
1. Some Linux distributions require setting execute permissions on .sh files prior to executing them. To be safe, running the command below results in the bash scripts being enabled with execution privileges.
   ```bash
   chmod +x ./00-set-vars.sh
   chmod +x ./01-aks-create.sh
   chmod +x ./02-deploy-apis.sh
   chmod +x ./03-deploy-web.sh
   chmod +x ../src/SmartHotel360-Azure-backend/deploy/k8s/build-push.sh
   chmod +x ../src/SmartHotel360-Azure-backend/deploy/k8s/deploy.sh
   ```
 1. Run the below command to set the environment variables for the current session.

      ```bash
      source 00-set-vars.sh
      ```
      You can verify the environment variables by running `env` command in the terminal.
1. Run the below command to setup the azure account in the terminal. You need to log in interactively from your web browser.

    ```bash
    az login
    ```

    You get a code to use in the next step as shown below

     ![](images/azlogin.png)

     Use a web browser to open the page [https://aka.ms/devicelogin](https://aka.ms/devicelogin) and enter the code to authenticate.

     Log in with your account credentials in the browser.

1. Run the below command to provision the required Azure resources. 

   ```bash
   source 01-aks-create.sh
   ```


1. It takes about 20-30 minutes to provision the environment. Once the cluster is created, take note of the URL value for the `HTTPApplicationRoutingZoneName` property in the response JSON payload. Copy this URL, as it will be used later when deploying the microservices.

   ```json
    "properties": {
        "addonProfiles": {
        "httpApplicationRouting": {
            "config": {
            "HTTPApplicationRoutingZoneName": "de44228e-2c3e-4bd8-98df-cdc6e54e272a.eastus.aksapp.io"
            }
     ```
   Now that the AKS cluster has been created we can publish the SmartHotel360 microservice source code into it.

## Exercise 1: Deploy the SmartHotel360 Backend APIs to AKS

In this section you'll build the images containing the SmartHotel360 back-end APIs and publish them into ACR, from where they'll be pulled and pushed into AKS when you do your deployment. We've scripted the complex areas of this to streamline the setup process, but you're encouraged to look in the .sh files to see (or improve upon) what's happening.

1. The end-to-end setup script makes use of some of the `export` environment variables you set earlier, so make sure they're still set by using the `echo` command to make sure they're still set. If you don't see values when you `echo` these environment variables, re-run `setup/00-set-vars.sh`.

   ```bash
   echo ${AKS_NAME}
   echo ${AKS_RG}
   echo ${ACR_NAME}
   echo ${AKS_SUB}
   ```
1. Change directory into the `setup` directory (if not already there) and run this command:

     ```bash
     source 02-deploy-apis.sh
     ```
   The script will take some time to execute, but when it is complete the `az aks browse` command will be executed and the Kubernetes dashboard will open in your browser.

1. When the dashboard opens (you may need to hit refresh as it may 404 at first), some of the objects in the cluster may not be fully ready. Hit refresh until these are all green and at 100%. Within a few minutes the cluster will show 100% for all of the objects in it.

    ![](images/workloadstatus.png)

In this exercise you deployed the back-end APIs to AKS cluster.

## Exercise 2: Set up Ingress
In order to route traffic to the various APIs within the AKS cluster, you'll need to set up a front door, or **ingress**.

1. If you forgot to note the `HTTPApplicationRoutingZoneName` property earlier, execute the command below to get the JSON representation of your cluster.

   ```bash
   az resource show --api-version 2018-03-31 --id /subscriptions/${AKS_SUB}/resourceGroups/${AKS_RG}/providers/Microsoft.ContainerService/managedClusters/${AKS_NAME}
   ```

1. You'll see the `HTTPApplicationRoutingZoneName` property in the JSON in the terminal window. Copy this value as it will be needed in the next step.

    ```json
    "properties": {
    "addonProfiles": {
    "httpApplicationRouting": {
        "config": {
        "HTTPApplicationRoutingZoneName": "de44228e-2c3e-4bd8-98df-cdc6e54e272a.eastus.aksapp.io"
        }
    ```
1.  Open the `ingress.yaml` file from the path `/src/SmartHotel360-Azure-backend/deploy/k8s/` and and see line 14, which has the `host` property set as follows:
    ```json
    spec:
     rules:
     - host: sh360.<guid>.<region>.aksapp.io
      http:
    ```
1. Replace the `host` property with the `sh360`. followed by value you copied earlier.

   ```json
   spec:
     rules:
    - host: sh360.de44228e-2c3e-4bd8-98df-cdc6e54e272a.eastus.aksapp.io
      http:
   ```

1. Execute the command below to set the AKS cluster ingress.

   ```bash
   kubectl apply -f ../src/SmartHotel360-Azure-backend/deploy/k8s/ingress.yaml
   ```

1. Open up the `src/SmartHotel360-public-web/manifests/ingress.yaml` file and see line 9, which has the `host` property set as follows:

    ```yaml
    spec:
      rules:
      - host: sh360.<guid>.<region>.aksapp.io
        http:
    ```

1. Replace the `host` property with the `sh360.` followed by value you copied earlier. 

    ```yaml
    spec:
      rules:
      - host: sh360.de44228e-2c3e-4bd8-98df-cdc6e54e272a.eastus.aksapp.io
        http:
    ```


## Exercise 3: Deploy the Public Web App

Now that the back-end APIs are in place the public web app can be pushed into the cluster, too. The spa app makes calls to the APIs running in the cluster and answers HTTP requests at the ingress URL you used earlier.

1. The end-to-end setup script makes use of some of the `export` environment variables you set earlier, so make sure they're still set by using the `echo` command to make sure they're still set. If you don't see values when you `echo` these environment variables, re-run `setup/00-set-vars.sh`. 

1. CD into the `setup` directory if you're not already there, and execute the command below. 

    ```bash
    source 03-deploy-web.sh
    ```

    The command may take a few minutes to complete. 

1. Once the command completes, paste the string you applied to the `host` property in the `ingress.yaml` files (the `sh360.<guid>.<region>.aksapp.io` string) into a browser, and you'll see the SmartHotel360 public web site load up. 

    ![Public web](images/publicweb.png)

## Exercise 4: Save the queries in Azure portal


There are three queries provided in the [`queries`](../queries) folder of the repository:

* CPU chart of the entire cluster over time
* Error log containing "0 results found" log entry
* Bar chart over time of the error log containing "0 results found" log entry

To make it easy to run these queries in next exercises, paste them in the Log Analytics Query Explorer and click the **Save** button, then give the query a name and category. 

To launch the Log Analytics Query Explorer open the Log Analytics resource created earlier and select Log Search. You launch the Advanced Analytics portal from a link on the Log Search page.

![Save the query](images/querylog.png)

Then they're readily available in the **Saved Queries** folder in the Query Explorer. 

![Running the query](images/savedqueries.png)


## Exercise 5: Using AKS and Azure Dev Spaces to make development and debugging apps easier

 Now the **SmartHotel360** web app is setup with back-end APIs deployed to AKS kubernetes cluster. In this exercise we will see how [Azure Dev Spaces](https://docs.microsoft.com/en-us/azure/dev-spaces/azure-dev-spaces) helps development teams be more productive on Kubernetes.

 Azure Dev Spaces provides a rapid, iterative Kubernetes development experience for teams. With minimal dev machine setup, you can iteratively run and debug containers directly in Azure Kubernetes Service (AKS). Develop on Windows, Mac, or Linux using familiar tools like Visual Studio, Visual Studio Code, or the command line. 

 {% include important.html content= "Azure Dev Spaces is in currently in preview, and is supported only by AKS clusters in the **East US, West Europe,** and **Canada East** regions. Previews are made available to you on the condition that you agree to the supplemental terms of use. Some aspects of this feature may change prior to general availability (GA)." %}

1. 