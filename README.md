---
page_type: sample
description: "Deploy dotnet application using GitHub Actions"
products:
- GitHub Actions
- Azure Web Apps
- Azure SQL Database
languages:
- dotnet
---

# ASP.NET Core and Azure SQL db for GitHub Actions

This repo contains a sample ASP.NET Core web application which uses an Azure SQL database as a backend. The web app can be deployed to Azure Web Apps by using GitHub Actions. The workflow for this is already present in the [.github](.github) folder.

For a sample on how to set up your first GitHub workflow from scratch, see [Create your first workflow](https://github.com/Azure/actions-workflow-samples)

The sample source code is based on the tutorial for building [ASP.NET Core and Azure SQL Database app in App Services](https://docs.microsoft.com/azure/app-service/containers/tutorial-dotnetcore-sqldb-app). 

## Setup an end-to-end CI/CD workflow

This repo contains two different GitHub workflows:
* [infraworkflow.yml](.github/workflows/infraworkflow.yml): To create the Azure Resources required for the sample by using an ARM template. This workflow will create the following resources:
    - Resource Group
    - App Service Plan
    - Web App (with one staging slot)
    - Azure SQL Server and the Azure SQL Database for the sample.
* [workflow.yml](.github/workflows/workflow.yaml): this workflow will build the sample app and publish it to a folder on the build agent and deploy this build code to the Web App staging slot. Next it will update the database and, finally, swap the slots.

To start, you can directly fork this repo and follow the instructions below to properly setup the workflows.

## Create Azure Resources (Infraworkflow.yml) setup
### 1. Setup environment variables
The [infraworkflow.yml](.github/workflows/infraworkflow.yaml) file for creating your azure resources, holds a couple of default values for the name of the resource group, as well as the name for the web app and SQL database. If you want to use other naming, you can change these values on lines 10 to 14. 

### 2. Setup your subscription ID

Additionaly this workflow uses a secret value for your Azure subscription ID. In your forked GitHub repository, go to Settings - Secrets and create a new secret called AZURE_SUBSCRIPTION_ID. Give it your subscription ID as a value. 

To find your current subscription ID, in an Azure cloud shell execute the following statements: 

```
az account list -o table
```

### 3. Create Azure credentials

You will also need to create a service principal and paste its details in another secret called AZURE_CREDENTIALS. This is called a deployment credential. The process for doing this is described [here](https://github.com/Azure/login#configure-deployment-credentials). 

Since the infraworkflow also deploys the resourcegroup, you will need a wider credential, which is allowed to perform action on the subscription level. For this you can create the service principal as such: 

```
az ad sp create-for-rbac --name "spname" --sdk-auth --role contributor --scopes /subscriptions/<subscription-id>
```

In case you don't want the service principal to have such wide permissions, you can create a regular resource group service principal with contributor rights as explained [here](https://github.com/Azure/login#configure-deployment-credentials) and create your resource group through the portal or through CLI statements. You will then also need to delete lines 39-48 in the infraworkflow.yml file. 

The deployment credential is used on line 27 in the infraworkflow.yml file.

### 4. Create database username and password secrets

Last, you will need 2 additional secrets for your SQL database username and password. Create 2 additional secrets for this called SQLADMIN_LOGIN and SQLADMIN_PASS. Make sure you choose a complex password, otherwise the create step for the SQL database server will fail. If it does, changing the secret value to a more complex one and re-running the workflow should fix this issue.

For further deatils, check https://docs.github.com/en/free-pro-team@latest/actions/reference/encrypted-secrets#creating-encrypted-secrets-for-a-repository

Finally, review the workflow and ensure the defined variables have the correct values and matches the pre-requisites you have just setup.

### 5. Execute the Create Resources workflow
 Go to your repo [Actions](../../actions) tab, under *All workflows* you will see the [Create Azure Resources](../../actions?query=workflow%3A"Create+Azure+Resources") workflow. 

Launch the workflow by using the *[workflow_dispatch](https://github.blog/changelog/2020-07-06-github-actions-manual-triggers-with-workflow_dispatch/)* event trigger.

This will create all the required resources in the Azure Subscription and Resource Group you configured.

This workflow will also trigger in case you make any changes to the [infraworkflow.yml](./.github/workflows/infraworkflow.yml) file or to any of the templates in the [templates](./templates/) folder/.

## Build and deploy app (Workflow.yml) setup
Now that the Azure resources got created, you can deploy the application to it. 

### 1. Setup environment variables
In case you changed any of the values of the environment variables in the infraworkflow.yml file, make sure you also change these values in the workflow.yml file to the same values.

### 2. Setup secrets
This workflow uses 1 additional secret as opposed to the infraworkflow.yml file and that is the web apps' publish profile. To get at this file, execute the following steps: 
- In the Azure portal navigate to the web app that got created by executing the infraworkflow.yml file. 
- On the overview page of the web app click on 'download publish profile'. 
- Copy paste the entire content of this publish profile to a new secret in your GitHub repo called AZURE_WEBAPP_PUBLISH_PROFILE. 

This secret is used in line 53 of the workflow.yml file. 

### 3. Execute the 'Build and deploy app' workflow
Go to your repo [Actions](../../actions) tab, under *All workflows* you will see the [Build and deploy app](../../actions?query=workflow%3A"Build+and+deploy+app") workflow. 

Launch the workflow by using the *[workflow_dispatch](https://github.blog/changelog/2020-07-06-github-actions-manual-triggers-with-workflow_dispatch/)* event trigger.

This will deploy all the code and database changes on the Azure resources.

This workflow will also trigger in case you make any changes to any of the files in this repository.

## Workflows YAML explained
As mentioned before, there are two workflows defined in the repo [Actions](../../actions) tab: the *Create Azure Resources* and the *Build and deploy app*.

### Create Azure Resources
Use this workflow to initially setup the Azure Resources by executing the ARM template which contain the resource definitions. This workflow is defined in the [infraworkflow.yaml](.github/workflows/infraworkflow.yaml) file, and has the following steps:

* Lines 2-6: triggers for this workflow.
* Lines 8-16: definition of environment variables.
* Lines 24-27: Login to azure CLI to gather environment and azure resources information by using the [Azure Login](https://github.com/Azure/login) action.
* Lines 30-33: Check out the source code by using the [Checkout](https://github.com/actions/checkout) action.
* Lines 36-44: Deploy the resources defined in the provided [ARM template](/templates/azuredeploy.resourcegroup.json) by using the [ARM Deploy](https://github.com/Azure/arm-deploy) action.
* Lines 47-56: Deploy the resources defined in the provided [ARM template](/templates/azuredeploy.json) by using the [ARM Deploy](https://github.com/Azure/arm-deploy) action.

### Build and deploy app
Use this workflow to deploy the code and database changes. This workflow is defined in the [workflow.yaml](.github/workflows/workflow.yaml) file, and has the following steps:

* Lines 3-10: triggers for this workflow.
* Lines 18-26: definition of environment variables.
* Line 34: Check out the source code by using the [Checkout](https://github.com/actions/checkout) action.
* Lines 37-40: Set the correct .Net version on the agent machine using the [setup-dotnet](https://github.com/actions/setup-dotnet) action.
* Lines 43-46: Execute dotnet build and publish statements.
* Lines 49-55: Update the App Service Web Application deployment for the *staging* slot by using the [Azure Web App Deploy](https://github.com/Azure/webapps-deploy) action.
* Lines 57-60: Login to azure CLI to gather environment and azure resources information by using the [Azure Login](https://github.com/Azure/login) action.
* Lines 62-67: Create the database connectionstring
* Lines 69-74: Execute the database update with entity framework using the above created connection string.
* Lines 76-79: Swap the web apps staging and production slots. 

## Contributing
Refer to the [Contributing page](/CONTRIBUTING.md)