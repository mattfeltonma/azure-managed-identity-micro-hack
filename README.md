# Managed Identity Micro-Hack

## Overview
[Managed Identities](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview) ease the burden of managing the credentials for the identities used by application components to authenticate Azure services and each other. These applications can be running in Azure or running in [on-premises or another cloud](https://docs.microsoft.com/en-us/azure/azure-arc/servers/managed-identity-authentication). Managed Identities come in two forms, [system-assigned and user-assigned](https://docs.microsoft.com/en-us/azure/azure-arc/servers/managed-identity-authentication). For a deep dive into the topic, check out [this blog post](https://journeyofthegeek.com/2019/08/07/deep-dive-into-azure-managed-identities-part-1/).

This micro-hack focuses on user-assigned managed identities because this type of managed identity commonly aligns with the way organizations handle their identity and access management processes.

The image below illustrates how a user-assigned managed identity is created and used.

![lab image](images/flow-general-diagram.png)

This repository includes code in the form of ARM templates and Azure CLI commands to walk you through how to create and use a managed identity. This micro-hack takes the "learn by doing" approach where the user will walk through the process of enabling an Azure Function with a managed identity in order for the function to retrieve a secret from an instance of Key Vault.

The end state architecture of this micro-hack is pictured below.

![lab image](images/demo-arch.svg)

## Prerequisites

1. This micro-hack will require you have sufficient permissions on the subscription to deploy resource groups, resources, and [create RBAC Roles and RBAC assignments](https://docs.microsoft.com/en-us/azure/role-based-access-control/overview). It is recommended you do this on a non-production subscription where you hold the Owner role at the subscription level.

2. You must have the az CLI commands installed and configured.

## Lab Setup

Before you begin the exercises, the lab environment must be deployed. The ARM templates within this repository will deploy an Azure Function, Azure Storage Account for the Function, an App Insights instance, an Azure Key Vault, and a secret within the Azure Key Vault.

This [simple Python function](https://github.com/mattfeltonma/azure-function-example) will be deployed to the Azure Function. It calls a public API to get the current time and gets a secret from Key Vault.

Deploy the lab environment by using the Deploy to Azure button below. Note that you must first create the resource group.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fmattfeltonma%2Fazure-managed-identity-demo%2Fmain%2Fazuredeploy.json)

## Test The Function

In this step you will test the Azure Function to ensure it deployed correctly.

Open the Azure Portal and retrieve the name that was given to the function. It will have prefix of func with a random eight-character GUID appending it similar to appabcd1234.

Open up a web browser and navigate to the function endpoint. The address will be:

https://FUNCTIONNAME.azurewebsites.net/api/pythonsample.

The function will deploy a result similar to the screenshot below.

![lab image](images/failed-function.png)

The function is fails because it is unauthorized to get the secret from the Key Vault. You will resolve this problem through this micro-hack.

Before proceeding to the exercises, create a variable to store the Function name. Enter the name of your Function into the myfunction placeholder.

**FUNCTION_APP_NAME=myfunction**

### Exercise 1

In the first exercise you will create a user-assigned management identity (UMI). You will view the properties of the UMI and find its service principal in Azure Active Directory.

1. Login to the the az CLI.

    **az login**

2. Set az CLI to use the appropriate subscription. Enter the name of your subscription into the subscription_name placeholder.

    **az account set --subscription subscription_name**

3. There are some variables that will be used throughout the micro-hack. Create a variable for your resource group name. Enter the name of the resource group you created in the placeholder of myresourcegroupname.

    **RESOURCE_GROUP_NAME=myresourcegroupname**

4. Create the user-assigned managed identity and name it demo-umi. 

    **az identity create --name demo-umi --resource-group $RESOURCE_GROUP_NAME**

    Take a moment and examine the properties returned. The clientId property maps to the appId attribute of the UMI's service principal in Azure AD. The principalId maps to the objectId of the UMI's service principal in Azure AD.

5. You will now look at the service principal for your managed identity. The most efficient way to locate the service principal in Azure AD is to search using the objectId. Recall that the principalId of the UMI maps to the objectId in Azure AD.

    Store the principalId property in a variable.

    **UMI_OBJECT_ID=$(az identity show --name=demo-umi --resource-group $RESOURCE_GROUP_NAME --query=principalId --output=tsv)**

6. Query Azure AD for the UMI's service principal.

    **az ad sp show --id=$UMI_OBJECT_ID**

    Review the attributes of the service principal. Notice that the servicePrincipalType is ManagedIdentity.

### Exercise 2

In this exercise you will create a custom RBAC role that allows for retreival of secrets from Azure Key Vault. You will then create a role assignment associating the custom role with the UMI and setting the scope as the resource group.

In the artifacts folder of this repository you will find a file named kv-secrets-reader.json which includes a custom RBAC role that will allow the function to retrieve the value of the secret stored in Key Vault. You will need to modify the file and replace YOURSUBSCRIPTIONID with the the id of the subscription you deployed the resources into.

1. Retrieve the subscription id of your subscription.

    **az account show --query id --output=tsv**

2. Modify the kv-secrets-reader.json file in the artifacts folder by replacing the YOURSUBSCRIPTIONID in the AssignableScopes section with the subscription id retrieved from last step.

3. Create the new custom role. Substitute the RELATIVE_PATH_TO_FILE with the path to the kv-secrets-reader.json file you modified.

    **az role definition create --role-definition RELATIVE_PATH_TO_FILE**

    For example:

    **az role definition create --role-definition artifacts/kv-secrets-reader.json**

4. Before you can create the role assignment you will need the id of the resource group you created.

    Create a variable with the resource group id. 

    **RG_ID=$(az group show --name $RESOURCE_GROUP_NAME --query id --output=tsv)**

5. Create the role assignment at the resource group scope.

**az role assignment create --assignee-object-id $UMI_OBJECT_ID --role "Custom - Key Vault Secrets Reader" --assignee-principal-type ServicePrincipal --scope $RG_ID**

### Exercise 3
You have now created an UMI, a custom RBAC role, and an assignment associating the role to the UMI at the resource group scope. The next step is to associate the UMI with the Azure Function. You will then populate the necessary application setting for the function.

1. Before you can assign the UMI to the Function App, you will need to get the UMI's resource id.

    Run this command to place the resource id in a variable.

    **UMI_ID=$(az identity show --name demo-umi --resource-group $RESOURCE_GROUP_NAME --query id --output tsv)**

2. Assign the UMI to the Function App.

    **az functionapp identity assign --resource-group $RESOURCE_GROUP_NAME --name $FUNCTION_NAME --identities $UMI_ID**

The last two steps are required due to the way the azure.identity library works in Python. When an UMI is being used, the method must specify the clientId of the UMI.

3. Create a variable with the UMI's clientId value. This will be the value set for the application setting.

    **UMI_CLIENT_ID=$(az identity show --name=demo-umi --resource-group $RESOURCE_GROUP_NAME --query=clientId --output=tsv)**

4. Set the MSI_CLIENT_ID application setting with the clientId value of the application setting.

    **az functionapp config appsettings set --name $FUNCTION_NAME --resource-group $RESOURCE_GROUP_NAME --settings "MSI_CLIENT_ID=$UMI_CLIENT_ID"**

It may take a few minutes for the new managed identity to take effect. Once you are ready, access the function again from a web browser. If you've completed all of the steps successfully, the results will look similar to the screenshot below.

![lab image](images/successful-function.png)

## Conclusion
In this micro-hack you created an user-assigned managed identity (UMI), created a custom RBAC role and assigned it to the UMI, and you associated the UMI to the Azure Function.

![lab image](images/flow-demo-diagram.svg)

## Clean-up
Delete the resource group to clean-up the resources created within this lab.


