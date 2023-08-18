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
   BASE_NAME=<base-resource-name between 3 and 6 charcters>
   RESOURCE_GROUP=<resource-group-name>
   az group create --location $LOCATION --resource-group $RESOURCE_GROUP

   az deployment group create --template-file ./infra-as-code/bicep/main.bicep \
     --resource-group $RESOURCE_GROUP \
     --parameters @./infra-as-code/bicep/parameters.json \
     --parameters baseName=$BASE_NAME
```

### Publish the web app

First, we need to clone the [Simple Web App workload repository](https://github.com/Azure-Samples/app-service-sample-workload)

```bash
cd ..
git clone https://github.com/Azure-Samples/app-service-sample-workload.git
cd app-service-sample-workload
```

Deploy zip file

```bash
APPSERVICE_NAME=app-$BASE_NAME
az webapp deploy --resource-group $RESOURCE_GROUP --name $APPSERVICE_NAME --src-path ./website/SimpleWebApp/SimpleWebApp.zip
```

### Validate the web app

Retrieve the web application URL and open it in your default web browser.

```bash
APPSERVICE_URL=https://$APPSERVICE_NAME.azurewebsites.net
echo $APPSERVICE_URL
```

## Clean Up

After you have finished exploring the AppService reference implementation, it is recommended that you delete the created Azure resources to prevent undesired costs from accruing.

```bash
az group delete --name $RESOURCE_GROUP -y
```
