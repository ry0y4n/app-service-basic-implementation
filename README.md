# App Services Basic Architecture

This repository contains the Bicep code to deploy an Azure App Services basic architecture.

## Deploy

The following are prerequisites.

## Prerequisites

1. Ensure you have an [Azure Account](https://azure.microsoft.com/free/)
1. Ensure you have the [Azure CLI installed](https://learn.microsoft.com/cli/azure/install-azure-cli)
1. Ensure you have the [az Bicep tools installed](https://learn.microsoft.com/azure/azure-resource-manager/bicep/install)

Use the following to deploy the infrastructure.

### Deploy the infrastructure

The following steps are required to deploy the infrastructure from the command line.

1. In your command-line tool where you have the Azure CLI and Bicep installed, navigate to the root directory of this repository (AppServicesRI)

1. Login and set subscription if it is needed

```bash
  az login 
  az account set --subscription xxxxx
```

1. Update the infra-as-code/parameters file

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "baseName": {
      "value": ""
    },
    "sqlAdministratorLogin": {
      "value": ""
    },
    "sqlAdministratorLoginPassword": {
      "value": ""
    }
  }
}
```

Note: Take into account that sql database enforce [password complexity](https://learn.microsoft.com/sql/relational-databases/security/password-policy?view=sql-server-ver16#password-complexity)

1. Run the following command to create a resource group and deploy the infrastructure. Make sure:

   - The BASE_NAME contains only lowercase letters and is between 6 and 12 characters. All resources will be named given this basename.
   - You choose a valid resource group name

```bash
   LOCATION=westus3
   BASE_NAME=<base-resource-name>
   RESOURCE_GROUP=<resource-group-name>

   az group create --location $LOCATION --resource-group $RESOURCE_GROUP

   az deployment group create --template-file ./infra-as-code/bicep/main.bicep \
     --resource-group $RESOURCE_GROUP \
     --parameters @./infra-as-code/bicep/parameters.json \
     --parameters baseName=$BASE_NAME
```

### Publish the web app

Deploy zip file from [App Service Sample Workload](https://github.com/Azure-Samples/app-service-sample-workload)

```bash
APPSERVICE_NAME=app-$BASE_NAME
az webapp deploy --resource-group $RESOURCE_GROUP --name $APPSERVICE_NAME --type zip --src-url https://raw.githubusercontent.com/Azure-Samples/app-service-sample-workload/main/website/SimpleWebApp.zip
```

### Validate the web app

Retrieve the web application URL and open it in your default web browser.

```bash
APPSERVICE_URL=https://$APPSERVICE_NAME.azurewebsites.net
echo $APPSERVICE_URL
```

## Optional Step: Service Connector

This implementation is using classic connection string to access the database. The app service has a variable called "AZURE_SQL_CONNECTIONSTRING" which is used by the application.
It is possible to use [Service Connector](https://learn.microsoft.com/azure/service-connector/overview). Service Connector helps you connect Azure compute services to other backing services. This service configures the network settings and connection information (for example, generating environment variables) between compute services and target backing services in management plane

You must open [Azure Cloud Shell on bash mode](https://learn.microsoft.com/azure/cloud-shell/quickstart), the command needs to connect the database and your computer is not allowed, only azure services are allowed.

```bash
   # Set variables on Azure Cloud Shell
   LOCATION=westus3
   BASE_NAME=<base-resource-name>
   RESOURCE_GROUP=<resource-group-name>
   APPSERVICE_NAME=app-$BASE_NAME
   RESOURCEID_DATABASE=$(az deployment group show -g $RESOURCE_GROUP -n databaseDeploy --query properties.outputs.databaseResourceId.value -o tsv)
   RESOURCEID_WEBAPP=$(az deployment group show -g $RESOURCE_GROUP -n webappDeploy --query properties.outputs.appServiceResourceId.value -o tsv)
   USER_IDENTITY_WEBAPP_CLIENTID=$(az deployment group show -g $RESOURCE_GROUP -n webappDeploy --query properties.outputs.appServiceIdentity.value -o tsv)
   USER_IDENTITY_WEBAPP_SUBSCRIPTION=$(az deployment group show -g $RESOURCE_GROUP -n webappDeploy --query properties.outputs.appServiceIdentitySubscriptionId.value -o tsv)
   
   # Delete current app service conection string, you could check on azure portal that the key was deleted
   az webapp config appsettings delete --name $APPSERVICE_NAME --resource-group $RESOURCE_GROUP --setting-names AZURE_SQL_CONNECTIONSTRING

   # Install service connector 
   az extension add --name serviceconnector-passwordless --upgrade

   az webapp connection create sql --connection sql_adventureconn --source-id $RESOURCEID_WEBAPP --target-id $RESOURCEID_DATABASE --client-type dotnet --user-identity client-id=$USER_IDENTITY_WEBAPP_CLIENTID subs-id=$USER_IDENTITY_WEBAPP_SUBSCRIPTION
   # The AZURE_SQL_CONNECTIONSTRING was created again but the connection string now includes "Authentication=ActiveDirectoryManagedIdentity"
   
```

## Clean Up

After you have finished exploring the AppService reference implementation, it is recommended that you delete the created Azure resources to prevent undesired costs from accruing.

```bash
az group delete --name $RESOURCE_GROUP -y
```
