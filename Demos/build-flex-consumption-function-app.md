# Build Flex Consumption Function App

This guide accompanies the '[Unleashing Extreme Scalability with Azure Functions](..\README.md)' presentation. In this session, we will build, deploy, and manage a Flex Consumption function app using the Azure CLI and the Azure Functions Core Tools. Azure also supports using the [Azure Portal,](https://portal.azure.com) the [Azure Developer CLI (azd)](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/), and the [Azure Tools pack](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-node-azure-pack) within [Visual Studio Code](https://learn.microsoft.com/en-us/azure/azure-functions/functions-develop-vs-code). Azure will soon add support for deployments via [Visual Studio](https://visualstudio.microsoft.com/), Azure DevOps Pipelines, and GitHub Actions.

## Prerequisites

- An Azure account with an active subscription. You can [create an account for free](https://azure.microsoft.com/free/?WT.mc_id=A261C142F) if you do not already have one.
- **[Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)**: used to create and manage resources in Azure. You will need version 2.60.0 or higher.
- **[Azure Function Core Tools](https://learn.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=v4%2Cmacos%2Ccsharp%2Cportal%2Cbash#install-the-azure-functions-core-tools)**: These are used to develop and test the functions on your local computer, deploy the project to Azure, and work with application settings.



## Clone the Random Content Repository

> 

## Create a Flex Consumption Function App

1. If you have not done so already, sign in to Azure:

```sh
az login
```

2. Use the `az functionapp list-flexconsumption-locations` command to review the list of regions that currently support Flex Consumption.

```sh
az functionapp list-flexconsumption-locations --ouptut table
```

3. Create a resource group in one of the currently supported regions:

```sh
az group create --name <RESOURCE_GROUP> --location <REGION>
```

In the command above, replace `<RESOURCE_GROUP>` with a unique value within your subscription and `<REGION>` with one of the currently supported regions.

4. Create a general-purpose storage account in the resource group and region:

```sh
az storage account create --name <STORAGE_NAME> --location <REGION> --resource-group <RESOURCE_GROUP> --sku Standard_LRS --allow-blob-public-access false
```

Replace `<STORAGE_NAME>` with a name appropriate to you and unique in Azure Storage. Names must contain only 3 to 24 characters (lowercase letters and numbers). `Standard_LRS` specifies a general-purpose account, which is supported by Azure Functions.

> [!IMPORTANT]
>
> The storage account stores function app data, including the application code itself. You are highly recommended not to use this storage account for any other purpose.

5. Create the function app in Azure:

```sh
az functionapp create --resource-group <RESOURCE_GROUP> --name <APP_NAME> --storage-account <STORAGE_NAME> --flexconsumption-location <REGION> --runtime dotnet-isolated --runtime-version 8.0
```

Replace <RESOURCE_GROUP> and <STORAGE_NAME> with the resource group and the account name you used in the previous steps. Also, replace `<APP_NAME>` with a globally unique name appropriate to you. The `<APP_NAME>` is also the default domain name server (DNS) domain for the function app.

## Deploy the Function App to Azure

In your project's root directory, run the following Azure Function Core Tools command:

```sh
func azure functionapp publish <APP_NAME>
```

Replace `<APP_NAME>` with the name of the app you created in the previous step. A successful deployment shows results similar to the following output (truncated for simplicity):

```sh
...

Getting site publishing info...
Creating archive for current directory...
Performing remote build for functions project.

...

Deployment successful.
Remote build succeeded!
Syncing triggers...
Functions in func-randomcontent-demo-use2:
    GetQuotes - [httpTrigger]
        Invoke url: https://func-randomcontent-demo-use2.azurewebsites.net/quotes/
    GetQuote - [httpTrigger]
        Invoke url: https://func-randomcontent-demo-use2.azurewebsites.net/quote/{quoteId}
```

## Enable Virtual Network Integration

One of the features of the Flex Consumption hosting option is its support for virtual network (VNet) integration.

1. Create a virtual network and subnet

```sh
az network vnet create --name <VNET_NAME> --resource-group <RESOURCE_GROUP> --address-prefix 10.0.0.0/16 --subnet-name <SUBNET_NAME> --subnet-prefixes 10.0.0.0/24
```

Replace `<VNET_NAME>` with the virtual network you want to create. Replace `<RESOURCE_GROUP>` with the resource group you created in the previous steps. Replace `<SUBNET_NAME>` with the name of the subnet you will use with your Flex Consumption function app.

2. Add the virtual network integration to the Flex Consumption function app:

```sh
az functionapp vnet-integration add --resource-group <RESOURCE_GROUP> --name <APP_NAME> --vnet <VNET_RESOURCE_ID> --subnet <SUBNET_NAME>
```

Replace `<RESOURCE_GROUP>` with the name of the previously created resource group, `<APP_NAME>` with the name of the previously created function app, `<VNET_RESOURCE_ID>` with the name of the previously created virtual network, and `<SUBNET_NAME>` with subnet within that virtual network.

> [!TIP]
>
> You can also define the virtual network integration when creating the Flex Consumption function app:
>
> ```sh
> az functionapp vnet-integration add --resource-group <RESOURCE_GROUP> --name <APP_NAME> --vnet <VNET_RESOURCE_ID> --subnet <SUBNET_NAME>
> ```

When choosing a subnet, these considerations apply:

- The subnet cannot already be used for other purposes, such as with private endpoints or service endpoints, or be delegated to any other hosting plan or service.
- You can share the same subnet with multiple apps running in a Flex Consumption plan. Because the networking resources are shared across all apps, one function app might impact the performance of others on the same subnet.
- In a Flex Consumption plan, a single function app might use up to 40 addresses, even when the app scales beyond 40 instances. While this rule of thumb is helpful when estimating the subnet size you need, it is not strictly enforced.

## Configure Instance Memory

The Flex Consumption function app will default be set to 2048 MB of instance memory. At any point, you can change the instance memory size setting:

```sh
az functionapp scale config set --resource-group <RESOURCE_GROUP> --name <APP_NAME> --instance-memory 4096
```

Replace `<RESOURCE_GROUP>` with the name of the resource group created in the previous steps. Replace `<APP_NAME>` with the name of the function app created in the previous steps.

> [!NOTE]
>
> As of this document, the Flex Consumption public preview only supports 2048 MB and 4096 MB. More sizes will be added eventually.

## Set the Always Ready Instance Count

A Flex Consumption function app is configured with no Always Ready instances by default. This can be changed at any time:

```sh
az functionapp scale config always-ready set --resource-group <RESOURCE_GROUP> --name <APP_NAME> --settings http=10
```

This will change the always-ready instance count for the HTTP triggers group to `10`.

Replace `<RESOURCE_GROUP>` with the name of the resource group created in the previous steps. Replace `<APP_NAME>` with the name of the function app created in the previous steps.



> [!TIP]
>
> You can set the always-ready instance count when creating the function app:
>
> ```sh
> az functionapp create --resource-group <RESOURCE_GROUP> --name <APP_NAME> --storage <STORAGE_NAME> --runtime <LANGUAGE_RUNTIME> --runtime-version <RUNTIME_VERSION> --flexconsumption-location <REGION> --always-ready-instances http=10
> ```



## Set HTTP Concurrency Limits

Unless you set specific limits, HTTP concurrency defaults for Flex Consumption plan apps are determined based on your instance size setting. You can change the concurrency limits for an existing app:

```sh
az functionapp scale config set -resource-group <RESOURCE_GROUP> -name <APP_NAME> --trigger-type http --trigger-settings perInstanceConcurrency=10
```

