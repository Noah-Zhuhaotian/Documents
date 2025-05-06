# Setting Up Secure Connections in Azure Data Factory with Key Vault

## Introduction

In today's data-driven world, establishing secure connections between your Azure resources is of paramount importance. This article will guide you through the process of creating a linked service in Azure Data Factory (DF1) that connects to Azure SQL Database (SQL1) using Microsoft SQL Server authentication, with the password securely stored in Azure Key Vault (KV1). We'll implement this solution following the principle of least privilege to ensure maximum security with minimal access rights. Additionally, we'll explore various automation options to streamline the deployment process.

![Secure Azure Data Integration Architecture](/assets/images/Azure/azure-data-integration.png)


## Understanding the Components

Before diving into the configuration, let's understand our resources:

- **DF1**: Azure Data Factory instance that orchestrates data movement and transformation
- **SQL1**: Azure SQL Database that stores our data
- **KV1**: Azure Key Vault that securely stores our sensitive credentials

## The Security Challenge

When setting up connections between services, it's common to hardcode credentials directly in connection strings. However, this approach poses significant security risks:

- Credentials visible to anyone with access to the configuration
- Difficult to update passwords across multiple services
- Lack of audit trail for credential access
- No centralized credential management

## Implementing the Least Privilege Solution

Following the principle of least privilege means granting only the minimum permissions necessary to perform required tasks. Here's how we'll implement this with our Azure resources:

### Step 1: Set Up a Managed Identity for Data Factory

First, we need to configure DF1 with a managed identity:

1. Navigate to your Azure Data Factory resource
2. Select "Properties" from the left menu
3. Under the "Managed Identity" section, toggle "System assigned" to "On"
4. Click "Save" to apply changes

This creates an identity for your Data Factory that Azure manages, eliminating the need for storing credentials for service-to-service authentication.

### Step 2: Store SQL Server Password in Key Vault

Next, we'll store the SQL Server authentication password in Key Vault:

1. Navigate to your Key Vault (KV1)
2. Select "Secrets" from the left menu
3. Click "+ Generate/Import"
4. Provide a name for your secret (e.g., "SQL1-Password")
5. Enter the SQL Server password as the value
6. Configure expiration date if needed
7. Click "Create"

### Step 3: Grant Data Factory Access to Key Vault

Now, we need to grant DF1's managed identity access to the specific secret in KV1:

1. In KV1, navigate to "Access policies"
2. Click "+ Add Access Policy"
3. Under "Secret permissions", select only "Get" permission
4. Under "Select principal", search for your Data Factory's managed identity name
5. Click "Add" to create the policy

This follows the principle of least privilege by granting DF1 only the ability to read secrets, without any write, list, or delete permissions.

### Step 4: Create a Linked Service in Data Factory

Finally, we'll create the linked service in DF1 that connects to SQL1 using the password from KV1:

1. In DF1, navigate to "Connections" → "Linked Services"
2. Click "+ New"
3. Select "Azure SQL Database"
4. Configure the following settings:
   - Name: (e.g., "LS_SQL1")
   - Account selection method: "Enter manually"
   - Server name: [Your SQL1 server name]
   - Database name: [Your SQL1 database name]
   - Authentication type: "SQL authentication"
   - Username: [Your SQL Server login]
   - Password: Select "Azure Key Vault"
   - AKV linked service: Create a new one pointing to KV1
   - Secret name: "SQL1-Password" (the name you created in Step 2)
5. Test connection to verify everything works
6. Click "Create" to save the linked service

## Automating the Configuration Process

While manual configuration is instructive, automation is key for consistent, repeatable deployments and to eliminate human error. Let's explore various automation options for implementing the secure connections between Azure Data Factory, SQL Database, and Key Vault.

### Option 1: Azure Resource Manager (ARM) Templates

ARM templates provide a declarative way to define your Azure infrastructure:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "dataFactoryName": {
      "type": "string",
      "defaultValue": "DF1"
    },
    "keyVaultName": {
      "type": "string",
      "defaultValue": "KV1"
    },
    "sqlServerName": {
      "type": "string",
      "defaultValue": "SQL1"
    },
    "sqlDatabaseName": {
      "type": "string",
      "defaultValue": "YourDatabaseName"
    },
    "sqlUsername": {
      "type": "string",
      "metadata": {
        "description": "SQL Server authentication username"
      }
    },
    "sqlPassword": {
      "type": "securestring",
      "metadata": {
        "description": "SQL Server authentication password"
      }
    },
    "secretName": {
      "type": "string",
      "defaultValue": "SQL1-Password"
    }
  },
  "variables": {
    "dataFactoryId": "[resourceId('Microsoft.DataFactory/factories', parameters('dataFactoryName'))]",
    "keyVaultId": "[resourceId('Microsoft.KeyVault/vaults', parameters('keyVaultName'))]"
  },
  "resources": [
    {
      "type": "Microsoft.DataFactory/factories",
      "apiVersion": "2018-06-01",
      "name": "[parameters('dataFactoryName')]",
      "location": "[resourceGroup().location]",
      "identity": {
        "type": "SystemAssigned"
      }
    },
    {
      "type": "Microsoft.KeyVault/vaults/secrets",
      "apiVersion": "2019-09-01",
      "name": "[concat(parameters('keyVaultName'), '/', parameters('secretName'))]",
      "dependsOn": [
        "[variables('keyVaultId')]"
      ],
      "properties": {
        "value": "[parameters('sqlPassword')]"
      }
    },
    {
      "type": "Microsoft.KeyVault/vaults/accessPolicies",
      "apiVersion": "2019-09-01",
      "name": "[concat(parameters('keyVaultName'), '/add')]",
      "dependsOn": [
        "[variables('dataFactoryId')]",
        "[variables('keyVaultId')]"
      ],
      "properties": {
        "accessPolicies": [
          {
            "tenantId": "[subscription().tenantId]",
            "objectId": "[reference(variables('dataFactoryId'), '2018-06-01', 'Full').identity.principalId]",
            "permissions": {
              "secrets": ["get"]
            }
          }
        ]
      }
    },
    {
      "type": "Microsoft.DataFactory/factories/linkedservices",
      "apiVersion": "2018-06-01",
      "name": "[concat(parameters('dataFactoryName'), '/LS_SQL1')]",
      "dependsOn": [
        "[variables('dataFactoryId')]",
        "[resourceId('Microsoft.KeyVault/vaults/accessPolicies', parameters('keyVaultName'), 'add')]"
      ],
      "properties": {
        "type": "AzureSqlDatabase",
        "typeProperties": {
          "connectionString": {
            "type": "SecureString",
            "value": "[concat('Server=tcp:', parameters('sqlServerName'), '.database.windows.net,1433;Database=', parameters('sqlDatabaseName'), ';User ID=', parameters('sqlUsername'), ';Password=', parameters('sqlPassword'), ';Trusted_Connection=False;Encrypt=True;Connection Timeout=30')]"
          }
        },
        "connectVia": {
          "referenceName": "AutoResolveIntegrationRuntime",
          "type": "IntegrationRuntimeReference"
        }
      }
    }
  ]
}
```

To deploy this ARM template:

```bash
az deployment group create --resource-group YourResourceGroup --template-file template.json --parameters parameters.json
```

### Option 2: Azure PowerShell Automation

PowerShell scripts provide greater flexibility and control over the deployment process:

```powershell
# Parameters
param(
    [string] $ResourceGroupName = "YourResourceGroup",
    [string] $DataFactoryName = "DF1",
    [string] $KeyVaultName = "KV1",
    [string] $SqlServerName = "SQL1",
    [string] $SqlDatabaseName = "YourDatabaseName",
    [string] $SqlUsername = "YourSqlUsername",
    [string] $SqlPassword = "YourSqlPassword",
    [string] $Location = "EastUS",
    [string] $SecretName = "SQL1-Password"
)

# Login to Azure
Connect-AzAccount

# 1. Ensure Data Factory uses a system-assigned managed identity
Write-Output "Configuring Data Factory managed identity..."
$dataFactory = Get-AzDataFactoryV2 -ResourceGroupName $ResourceGroupName -Name $DataFactoryName
if ($dataFactory.Identity -eq $null) {
    $dataFactory = Set-AzDataFactoryV2 -ResourceGroupName $ResourceGroupName -Name $DataFactoryName -Location $Location -IdentityType SystemAssigned
    Write-Output "System-assigned managed identity enabled"
} else {
    Write-Output "System-assigned managed identity already exists"
}

# Get the Data Factory service principal ID
$dataFactoryPrincipalId = $dataFactory.Identity.PrincipalId
Write-Output "Data Factory Principal ID: $dataFactoryPrincipalId"

# 2. Store SQL password in Key Vault
Write-Output "Storing SQL password in Key Vault..."
$secretValue = ConvertTo-SecureString -String $SqlPassword -AsPlainText -Force
$secret = Set-AzKeyVaultSecret -VaultName $KeyVaultName -Name $SecretName -SecretValue $secretValue
Write-Output "Password stored as secret: $($secret.Name)"

# 3. Grant Data Factory access to Key Vault
Write-Output "Granting Data Factory access to Key Vault..."
Set-AzKeyVaultAccessPolicy -VaultName $KeyVaultName -ObjectId $dataFactoryPrincipalId -PermissionsToSecrets get
Write-Output "Access policy set"

# 4. Create ADF linked service to Key Vault
Write-Output "Creating Key Vault linked service..."
$keyVaultLinkedServiceName = "LS_KeyVault"
$keyVaultLinkedServiceDefinition = @{
    properties = @{
        type = "AzureKeyVault"
        typeProperties = @{
            baseUrl = "https://$KeyVaultName.vault.azure.net/"
        }
    }
}
$keyVaultLinkedServiceJson = $keyVaultLinkedServiceDefinition | ConvertTo-Json -Depth 10
$keyVaultLinkedServiceJson | Out-File ".\KeyVaultLinkedService.json"
Set-AzDataFactoryV2LinkedService -ResourceGroupName $ResourceGroupName -DataFactoryName $DataFactoryName -Name $keyVaultLinkedServiceName -File ".\KeyVaultLinkedService.json"
Write-Output "Key Vault linked service created"

# 5. Create linked service to SQL database
Write-Output "Creating SQL linked service..."
$sqlLinkedServiceName = "LS_SQL1"
$sqlLinkedServiceDefinition = @{
    properties = @{
        type = "AzureSqlDatabase"
        typeProperties = @{
            connectionString = "Server=tcp:$SqlServerName.database.windows.net,1433;Database=$SqlDatabaseName;Trusted_Connection=False;Encrypt=True;Connection Timeout=30"
            userName = $SqlUsername
            password = @{
                type = "AzureKeyVaultSecret"
                store = @{
                    referenceName = $keyVaultLinkedServiceName
                    type = "LinkedServiceReference"
                }
                secretName = $SecretName
            }
        }
    }
}
$sqlLinkedServiceJson = $sqlLinkedServiceDefinition | ConvertTo-Json -Depth 10
$sqlLinkedServiceJson | Out-File ".\SqlLinkedService.json"
Set-AzDataFactoryV2LinkedService -ResourceGroupName $ResourceGroupName -DataFactoryName $DataFactoryName -Name $sqlLinkedServiceName -File ".\SqlLinkedService.json"
Write-Output "SQL linked service created"

Write-Output "Configuration complete! Successfully set up connections following least privilege principle"
```

### Option 3: Azure CLI Automation

Azure CLI provides a cross-platform command-line experience:

```bash
#!/bin/bash

# Set variables
RESOURCE_GROUP="YourResourceGroup"
LOCATION="eastus"
DATA_FACTORY_NAME="DF1"
KEY_VAULT_NAME="KV1"
SQL_SERVER_NAME="SQL1"
SQL_DATABASE_NAME="YourDatabaseName"
SQL_USERNAME="YourSqlUsername"
SQL_PASSWORD="YourSqlPassword"
SECRET_NAME="SQL1-Password"

# Login to Azure
echo "Logging in to Azure..."
az login

# 1. Ensure Data Factory has system-assigned managed identity
echo "Configuring Data Factory system-assigned managed identity..."
az datafactory identity assign --resource-group $RESOURCE_GROUP --factory-name $DATA_FACTORY_NAME

# Get Data Factory managed identity principal ID
ADF_PRINCIPAL_ID=$(az datafactory show --resource-group $RESOURCE_GROUP --factory-name $DATA_FACTORY_NAME --query 'identity.principalId' -o tsv)
echo "Data Factory Principal ID: $ADF_PRINCIPAL_ID"

# 2. Store SQL password in Key Vault
echo "Storing SQL password in Key Vault..."
az keyvault secret set --vault-name $KEY_VAULT_NAME --name $SECRET_NAME --value $SQL_PASSWORD

# 3. Grant Data Factory access to Key Vault
echo "Granting Data Factory access to Key Vault..."
az keyvault set-policy --name $KEY_VAULT_NAME --object-id $ADF_PRINCIPAL_ID --secret-permissions get

# 4. Create Key Vault linked service
echo "Creating Key Vault linked service..."
KV_LINKED_SERVICE_NAME="LS_KeyVault"

# Create Key Vault linked service definition file
cat > keyvault-linked-service.json <<EOF
{
  "name": "$KV_LINKED_SERVICE_NAME",
  "properties": {
    "type": "AzureKeyVault",
    "typeProperties": {
      "baseUrl": "https://$KEY_VAULT_NAME.vault.azure.net/"
    }
  }
}
EOF

# Deploy Key Vault linked service
az datafactory linked-service create --resource-group $RESOURCE_GROUP --factory-name $DATA_FACTORY_NAME --linked-service-name $KV_LINKED_SERVICE_NAME --properties @keyvault-linked-service.json

# 5. Create SQL linked service
echo "Creating SQL linked service..."
SQL_LINKED_SERVICE_NAME="LS_SQL1"

# Create SQL linked service definition file
cat > sql-linked-service.json <<EOF
{
  "name": "$SQL_LINKED_SERVICE_NAME",
  "properties": {
    "type": "AzureSqlDatabase",
    "typeProperties": {
      "connectionString": "Server=tcp:$SQL_SERVER_NAME.database.windows.net,1433;Database=$SQL_DATABASE_NAME;Trusted_Connection=False;Encrypt=True;Connection Timeout=30",
      "userName": "$SQL_USERNAME",
      "password": {
        "type": "AzureKeyVaultSecret",
        "store": {
          "referenceName": "$KV_LINKED_SERVICE_NAME",
          "type": "LinkedServiceReference"
        },
        "secretName": "$SECRET_NAME"
      }
    }
  }
}
EOF

# Deploy SQL linked service
az datafactory linked-service create --resource-group $RESOURCE_GROUP --factory-name $DATA_FACTORY_NAME --linked-service-name $SQL_LINKED_SERVICE_NAME --properties @sql-linked-service.json

echo "Configuration complete! Successfully set up connections following least privilege principle"
```

### Option 4: Terraform Automation

Terraform provides a powerful infrastructure-as-code approach:

```hcl
# Configure Azure provider
provider "azurerm" {
  features {}
}

# Variable definitions
variable "resource_group_name" {
  description = "Resource group name"
  default     = "YourResourceGroup"
}

variable "location" {
  description = "Azure region"
  default     = "eastus"
}

variable "data_factory_name" {
  description = "Data Factory name"
  default     = "DF1"
}

variable "key_vault_name" {
  description = "Key Vault name"
  default     = "KV1"
}

variable "sql_server_name" {
  description = "SQL Server name"
  default     = "SQL1"
}

variable "sql_database_name" {
  description = "SQL database name"
  default     = "YourDatabaseName"
}

variable "sql_username" {
  description = "SQL username"
  default     = "YourSqlUsername"
}

variable "sql_password" {
  description = "SQL password"
  sensitive   = true
}

variable "secret_name" {
  description = "Key Vault secret name"
  default     = "SQL1-Password"
}

# Use existing resource group
data "azurerm_resource_group" "rg" {
  name = var.resource_group_name
}

# Data Factory configuration
resource "azurerm_data_factory" "adf" {
  name                = var.data_factory_name
  location            = data.azurerm_resource_group.rg.location
  resource_group_name = data.azurerm_resource_group.rg.name
  identity {
    type = "SystemAssigned"
  }
}

# Reference existing Key Vault
data "azurerm_key_vault" "kv" {
  name                = var.key_vault_name
  resource_group_name = data.azurerm_resource_group.rg.name
}

# Store password in Key Vault
resource "azurerm_key_vault_secret" "sql_password" {
  name         = var.secret_name
  value        = var.sql_password
  key_vault_id = data.azurerm_key_vault.kv.id
}

# Set Key Vault access policy for Data Factory
resource "azurerm_key_vault_access_policy" "adf_policy" {
  key_vault_id = data.azurerm_key_vault.kv.id
  tenant_id    = azurerm_data_factory.adf.identity[0].tenant_id
  object_id    = azurerm_data_factory.adf.identity[0].principal_id

  secret_permissions = [
    "Get",
  ]
}

# Reference existing SQL database
data "azurerm_sql_server" "sql" {
  name                = var.sql_server_name
  resource_group_name = data.azurerm_resource_group.rg.name
}

data "azurerm_sql_database" "db" {
  name                = var.sql_database_name
  server_name         = data.azurerm_sql_server.sql.name
  resource_group_name = data.azurerm_resource_group.rg.name
}

# Create Key Vault linked service
resource "azurerm_data_factory_linked_service_key_vault" "ls_kv" {
  name                = "LS_KeyVault"
  resource_group_name = data.azurerm_resource_group.rg.name
  data_factory_name   = azurerm_data_factory.adf.name
  key_vault_id        = data.azurerm_key_vault.kv.id

  depends_on = [
    azurerm_key_vault_access_policy.adf_policy
  ]
}

# Create SQL linked service
resource "azurerm_data_factory_linked_service_azure_sql_database" "ls_sql" {
  name                = "LS_SQL1"
  resource_group_name = data.azurerm_resource_group.rg.name
  data_factory_name   = azurerm_data_factory.adf.name
  
  connection_string   = "Server=tcp:${data.azurerm_sql_server.sql.fully_qualified_domain_name};Database=${var.sql_database_name};"
  use_managed_identity = false
  username            = var.sql_username
  key_vault_password {
    linked_service_name = azurerm_data_factory_linked_service_key_vault.ls_kv.name
    secret_name         = var.secret_name
  }

  depends_on = [
    azurerm_data_factory_linked_service_key_vault.ls_kv,
    azurerm_key_vault_secret.sql_password
  ]
}
```

## Benefits of Automation

Implementing automation for your Azure Data Factory secure connections offers several advantages:

1. **Consistency and Repeatability**: Automation scripts ensure that every deployment uses the same configuration, avoiding human errors in manual setup.

2. **Version Control**: Automation scripts can be stored in source control systems, tracking the history of configuration changes over time.

3. **Scalable Deployment**: If you need to deploy the same configuration across multiple environments, automation scripts can be easily adapted to different environments.

4. **CI/CD Integration**: Any of the above scripts can be integrated into continuous integration and continuous deployment pipelines for end-to-end automation, you can click *[here](https://github.com/Noah-Zhuhaotian/Maintaince_templete/tree/main/Azure/Security/DF_SQL)* for reference!


## Best Practices for Automation

When using automation scripts, consider these best practices:

- Avoid hardcoding sensitive information in scripts, especially passwords
- Use parameter files or environment variables to pass configuration values
- Validate automation scripts in test environments before running in production
- Regularly review and update permission configurations to ensure least privilege is maintained
- Configure monitoring and alerts to promptly detect any abnormal access

## Security Benefits of This Approach

By implementing this solution, we gain several security benefits:

1. **Credential Isolation**: The SQL password is stored only in Key Vault, not in Data Factory configuration
2. **Centralized Secret Management**: Passwords can be rotated in one place without updating multiple services
3. **Minimal Permissions**: Data Factory has only the necessary permission to read the specific secret
4. **Auditability**: All access to the password is logged in Key Vault's audit logs
5. **No Credential Exposure**: The password is never exposed in plaintext during deployment or configuration

## Conclusion

Following the principle of least privilege is essential for maintaining a secure Azure environment. By using Azure Key Vault to store sensitive credentials and carefully managing access permissions, we've created a secure connection between Azure Data Factory and SQL Database that minimizes potential attack surfaces. The addition of automation options further enhances this security by ensuring consistent deployment and reducing the risk of human error.

This approach not only enhances security but also streamlines credential management, making your data pipeline more maintainable and audit-friendly. As your Azure infrastructure grows, applying these same principles and automation practices to other resource connections will help maintain a robust security posture.

Remember that security is a continuous process, not a one-time setup. Regularly review your access policies and rotate credentials to ensure your Azure resources remain secure.