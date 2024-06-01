---
title: "Azure SQL - Create SQL User From Managed Identity"
datePublished: Fri May 31 2024 07:14:36 GMT+0000 (Coordinated Universal Time)
cuid: clwucmlsd000s0akzdejebwfi
slug: azure-sql-create-sql-user-from-managed-identity
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/oyXis2kALVg/upload/3dee44ee54543f169d68122b695fca53.jpeg
tags: azure, azure-pipelines, azure-sql-database, azure-managed-identities

---

When using Managed Identities in Azure, a common requirement is creating a SQL user for your app's Managed Identity in your Azure SQL Database from some CI/CD pipeline. Sometimes apps need to talk to a database - who knew?

[According to the documentation](https://learn.microsoft.com/en-us/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?view=azuresql#permissions), this requires that you grant the SQL Server identity additional permissions, which in turn can only be granted by either a [Global Administrator](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#global-administrator) or [Privileged Role Administrator](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#global-administrator).

Once the SQL Server identity has these permissions, one can execute the following query to add the application's identity to the database:

```sql
-- Create the user with details retrieved from Entra ID
CREATE USER [ManagedIdentityName] FROM EXTERNAL PROVIDER

-- Assign roles to the new user
ALTER ROLE db_datareader ADD MEMBER [ManagedIdentityName]
ALTER ROLE db_datawriter ADD MEMBER [ManagedIdentityName]
```

Common strategies for enabling this at scale are:

1. Create a single User Assigned Managed Identity with the required permissions & associate it to all SQL Servers
    
2. Create a dedicated Entra ID group for SQL identities & grant it the [Directory Readers](https://learn.microsoft.com/en-us/azure/azure-sql/database/authentication-aad-directory-readers-role-tutorial?view=azuresql) role
    

There exists, however, an alternative approach which doesn't require any of the above pre-configuration.

Given the Client ID of your app's Managed Identity, you can use the following PowerShell to convert the client ID into the SQL Server's User SID format:

```powershell
$managedIdentitySid = "0x" + [System.BitConverter]::ToString(([guid]$managedIdentityClientId).ToByteArray()).Replace("-", "")
```

And then add the user to the SQL Database as follows:

```sql
-- Create the user using its SID - does not need to talk to Entra ID
CREATE USER [ManagedIdentityName] WITH SID = <managedIdentitySid>, TYPE = E;

-- Assign roles to the new user
ALTER ROLE db_datareader ADD MEMBER [ManagedIdentityName]
ALTER ROLE db_datawriter ADD MEMBER [ManagedIdentityName]
```

Here is an example of an Azure Pipeline using the above approach:

```yaml
- task: AzureCLI@2
  displayName: 'Get Managed Identity Client ID'
  inputs:
    azureSubscription: ${{ parameters.serviceConnection }}
    scriptType: 'pscore'
    scriptLocation: 'inlineScript'
    inlineScript: |
      $clientId = az identity show --name $(managedIdentityName) --resource-group $(resourceGroupName) --query clientId -o tsv
      echo "##vso[task.setvariable variable=managedIdentityClientId]$clientId"

- powershell: |
    $clientId = '$(managedIdentityClientId)'
    $sid = "0x" + [System.BitConverter]::ToString(([guid]$clientId).ToByteArray()).Replace("-", "")

    Write-Host "##vso[task.setvariable variable=managedIdentitySid]$sid"
  displayName: Convert Managed Identity Client ID to SID

# The Azure DevOps pipeline Service Connection (Azure AD Service Principal) needs to be a SQL Server Administrator (AAD Group) or Database Owner
# The SqlAzureDacpacDeployment@1 task can only run on windows agents
- task: SqlAzureDacpacDeployment@1
  displayName: 'Add User + Roles for Managed Identity'
  inputs:
    azureSubscription: ${{ parameters.serviceConnection }}
    AuthenticationType: 'servicePrincipal'
    ServerName: '$(sqlServerName).database.windows.net'
    DatabaseName: '$(sqlDatabaseName)'
    TaskNameSelector: 'InlineSqlTask'
    SqlInline: |
      IF NOT EXISTS (SELECT [name] FROM [sys].[database_principals] WHERE [name] = '$(managedIdentityName)')
      CREATE USER [$(managedIdentityName)] WITH SID = $(managedIdentitySid), TYPE = E;
      GO
      ALTER ROLE [db_datareader] ADD MEMBER [$(managedIdentityName)];
      ALTER ROLE [db_datawriter] ADD MEMBER [$(managedIdentityName)];
      GO
    IpDetectionMethod: 'AutoDetect'
```

I can't take credit for the PowerShell script, I found it by happenstance in [this StackOverflow answer](https://stackoverflow.com/a/76996864) by [Thomas Vercoutre](https://github.com/CrazyTuna). Thomas you legend! 👏