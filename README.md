# Azure Managed Application - Publisher Azure Key Vault Secret Example

This repo shows an example of Azure Managed Application that gets a secret from publisher's Azure Key Vault and stores it in a Key Vault within the Managed Resource Group.

## Create Azure Key Vault in Publisher's Azure Subscription

<https://docs.microsoft.com/en-us/azure/azure-resource-manager/managed-applications/key-vault-access>

Create Azure Key Vault in publisher's subscription and add a secret

```bash
az keyvault create --resource-group avama --name avamakv --location eastus --enabled-for-template-deployment true
az keyvault secret set --vault-name avamakv --name "publisherSecret" --value "***this is publisherSecret value***"
```

Error when Appliance Resource Provider is not granted proper role to the Key Vault

```json
{
    "error": {
        "code": "KeyVaultParameterReferenceAuthorizationFailed",
        "message": "The client 'efa7e07b-0100-499d-b1fb-ae609506004b' with object id 'efa7e07b-0100-499d-b1fb-ae609506004b' does not have permission to perform action 'MICROSOFT.KEYVAULT/VAULTS/DEPLOY/ACTION' on the specified KeyVault resource '/subscriptions/06b230b6-ec16-422c-a319-487cbe82501a/resourceGroups/avama/providers/Microsoft.KeyVault/vaults/avamakv'. Please see https://aka.ms/arm-keyvault for usage details."
    }
}
```

<https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/key-vault-parameter?tabs=azure-cli#grant-access-to-the-secrets>

Grant custom role to "Appliance Resource Provider" for the Key Vault

```bash
# Create custom role definition
az role definition create --role-definition keyvault-customrole.json

# Get objectId of the Appliance Resource Provider service principal
az ad sp list --display-name "Appliance Resource Provider" -o tsv --query "[].objectId"

# Assign the role to Appliance Resource Provider
az role assignment create --role "Key Vault resource manager template deployment operator" --scope "/subscriptions/06b230b6-ec16-422c-a319-487cbe82501a/resourceGroups/avama/providers/Microsoft.KeyVault/vaults/avamakv" --assignee-object-id "efa7e07b-0100-499d-b1fb-ae609506004b" --assignee-principal-type ServicePrincipal
```

## Modify mainTemplate.json to use Publisher's Azure Key Vault

Modify the following section of the mainTemplate.json to reference publisher's Azure Key Vault by setting proper value for the subscriptionId (e.g. 06b230b6-ec16-422c-a319-487cbe82501a), resourceGroup (e.g. avama), Azure Key Vault Name (e.g. avamakv), and secret name in that Key Vault (e.g. publisherSecret)

```json
"publisherSecret": {
    "reference": {
        "keyVault": {
            "id": "[resourceId('06b230b6-ec16-422c-a319-487cbe82501a','avama','Microsoft.KeyVault/vaults','avamakv')]"
        },
        "secretName": "publisherSecret"
    }
},
```

## Create Azure Managed App Definition using "Service Catalog"

> Important: When building your Azure Application ARM templates for submission to Azure Marketplace, please make sure to carefully follow all of the guidelines and practices described [here](https://github.com/Azure/azure-quickstart-templates/blob/master/1-CONTRIBUTION-GUIDE/best-practices.md) and be ready to make fixes and changes based on [manual review feedback](https://docs.microsoft.com/en-us/azure/marketplace/partner-center-portal/azure-apps-review-feedback).

Test the application using <http://github.com/Azure/arm-ttk> tool

```cmd
C:\Code\GitHub\Azure\arm-ttk\arm-ttk\Test-AzTemplate.cmd
```

Zip and AzCopy to storage

```bash
tar -a -c -f ama-keyvault.zip createUiDefinition.json mainTemplate.json viewDefinition.json

azcopy copy ./ama-keyvault.zip "https://YOUR_STORAGE_ACCOUNT.blob.core.windows.net/YOUR_STORAGE_CONTAINER/ama-keyvault.zip?SHARED_ACCESS_SIGNATURE_WITH_WRITE_PERMISSION"
```

Create managed application definition

```bash
az group create --name avama --location eastus

az managedapp definition create --name "ama-definition" --location eastus --resource-group avama --lock-level ReadOnly --display-name "Hello World App Definition" --description "Azure Managed App Hello World Example" --authorizations "YOUR_AAD_GROUP_PRINCIPAL_ID:b24988ac-6180-42a0-ab88-20f7382dd24c" --package-file-uri "https://YOUR_STORAGE_ACCOUNT.blob.core.windows.net/ama-aks/ama-aks.zip"
```

## Test

First, try deploying using the created Service Catalog definition and it should work successfully.

Next, try something that shouldn't work. For example, deployment using Service Catalog definition created in tenant 72f988bf and pointing to Key Vault in tenant dd74924a

As expected, deployment fails with error message "KeyVaultParameterReferenceNotInTheSameTenant"

```json
{
  "code": "DeploymentFailed",
  "message": "At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/DeployOperations for usage details.",
  "details": [
    {
      "code": "KeyVaultParameterReferenceNotInTheSameTenant",
      "message": "The specified KeyVault '/subscriptions/06b230b6-ec16-422c-a319-487cbe82501a/resourceGroups/avama/providers/Microsoft.KeyVault/vaults/avamakv' is not in current tenant '72f988bf' for subscription '06b230b6-ec16-422c-a319-487cbe82501a'. Please see https://aka.ms/arm-keyvault for usage details."
    }
  ]
}
```

Secret access using user assigned managed identity from Azure Container Instance

```bash
az container create --resource-group mrg-avkeyvault --name aci2 --image "mcr.microsoft.com/azure-cli" --assign-identity "/subscriptions/06b230b6-ec16-422c-a319-487cbe82501a/resourcegroups/mrg-avkeyvault/providers/Microsoft.ManagedIdentity/userAssignedIdentities/avkeyvault-identity" --command-line "/bin/bash -c 'sleep 100000'"

az login --identity --username "/subscriptions/06b230b6-ec16-422c-a319-487cbe82501a/resourcegroups/mrg-avkeyvault/providers/Microsoft.ManagedIdentity/userAssignedIdentities/avkeyvault-identity" --allow-no-subscriptions

az keyvault secret list --vault-name avkeyvaultqgsyf4zscfjog

az keyvault secret show --vault-name avkeyvaultqgsyf4zscfjog --name publisherSecret
```
