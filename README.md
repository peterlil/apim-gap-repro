# apim-gap-repro
Repo for repro a difference in Portal functionality vs functionality in the PowerShell CmdLet `Set-AzApiManagementApi`.

## Problem statement and background
The PowerShell CmdLet `Set-AzApiManagementApi` fails and returns the error code `AuthorizationFailed` as per the image below when updating an API, and when executed by a user in a custom RBAC role defined below.

<img width="741" alt="image" src="https://user-images.githubusercontent.com/29121387/212117094-36ae76c0-aa9d-4892-b588-467117b38e34.png">

In this context the organization has one central team managing Azure API Management (apim) as a platform to be used by many workload teams. The workload teams publishes and maintains their own apis in the apim platform, but they should not be able to see nor modify any other workload teams's apis. To accomplish this, the devs of the workload teams are all assigned a role with only the necessary reading permissions on the apim instance, let's call this role `APIM Common Dev`, and it's defined as:
```json
{
  "Name": "APIM Common Dev",
  "Id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "IsCustom": true,
  "Description": "Read-only access to service",
  "Actions": [
    "Microsoft.ApiManagement/service/read",
    "Microsoft.Authorization/*/read",
    "Microsoft.Insights/alertRules/*",
    "Microsoft.ResourceHealth/availabilityStatuses/read",
    "Microsoft.Resources/deployments/*",
    "Microsoft.Resources/subscriptions/resourceGroups/read",
    "Microsoft.Support/*",
    "Microsoft.ApiManagement/service/policySnippets/read",
    "Microsoft.ApiManagement/service/loggers/read"
  ],
  "NotActions": [
    "Microsoft.ApiManagement/service/users/keys/read"
  ],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": [
    "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/apim-prod"
  ]
}
```

In addition to that role, a new role is created everytime a team creates a new api. This role is specific for their team and should only grant required access to maintain that specific api:
```json
{
  "Name": "APIM Role Scoped down to Echo API",
  "Id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "IsCustom": true,
  "Description": "Maintainer of Echo API",
  "Actions": [
    "Microsoft.ApiManagement/service/*/read",
    "Microsoft.ApiManagement/service/read",
    "Microsoft.Authorization/*/read",
    "Microsoft.Insights/alertRules/*",
    "Microsoft.ResourceHealth/availabilityStatuses/read",
    "Microsoft.Resources/deployments/*",
    "Microsoft.Resources/subscriptions/resourceGroups/read",
    "Microsoft.Support/*",
    "Microsoft.ApiManagement/service/apis/*",
    "Microsoft.ApiManagement/service/loggers/read"
  ],
  "NotActions": [],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": [
    "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/apim-prod/providers/Microsoft.ApiManagement/service/apim-prod/apis/echo-api",
    "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/apim-prod/providers/Microsoft.ApiManagement/service/apim-prod/loggers/echo-api"
  ]
}
```

When a dev is assigned to both the roles above and nothing else and executes the script below, it fails with the error.
```PowerShell
# Log in to Azure
Connect-AzAccount -UseDeviceAuthentication
# Get an apim context
$context = New-AzApiManagementContext -ResourceGroupName "$resourceGroupName" -ServiceName "$apimInstanceName"
# Get the Echo API
$apis = Get-AzApiManagementApi -Context $context
$api = $apis | Where-Object { $_.ApiId -EQ 'echo-api' }
# Change a property
$api.Description = ([int]($api.Description) + 1).ToString()
# Save the changes
$result = Set-AzApiManagementApi -InputObject $api -PassThru
```

That code should work, and when doing the exact same thing in the Azure Portal it works and the changes are persisted.

<img width="923" alt="image" src="https://user-images.githubusercontent.com/29121387/212129458-8669793a-c4a3-42d2-a29e-a8d714909016.png">

To follow the [documentation](https://learn.microsoft.com/en-us/rest/api/apimanagement/current-ga/apis/create-or-update?tabs=HTTP) and post an update with Postman is also not a problem.

<img width="1091" alt="image" src="https://user-images.githubusercontent.com/29121387/212131144-85e63795-03c4-4d10-b109-371c83403cb0.png">

## Investigation and results

`Set-AzApiManagementApi` clearly does something different than the other methods that are working. By using Fiddler the raw request was captured.
```http
PUT https://management.azure.com/subscriptions/<sub-id>/resourceGroups/apim-prod/providers/Microsoft.ApiManagement/service/apim-prod/apis/echo-api%3Brev%3D1?api-version=2021-08-01 HTTP/1.1
Content-Type: application/json; charset=utf-8
Content-Length: 459

{
  "properties": {
    "description": "8",
    "subscriptionKeyParameterNames": {
      "header": "Ocp-Apim-Subscription-Key",
      "query": "subscription-key"
    },
    "type": "http",
    "apiRevision": "1",
    "apiVersion": "",
    "isCurrent": true,
    "subscriptionRequired": true,
    "displayName": "Echo API",
    "serviceUrl": "http://echoapi.cloudapp.net/api",
    "path": "echo",
    "protocols": [
      "Https"
    ]
  }
}
```

When we compare that with the portal request and how the request is documented, we see one major difference in the URI.
```http
PATCH https://management.azure.com/subscriptions/<sub-id>/resourceGroups/apim-prod/providers/Microsoft.ApiManagement/service/apim-prod/apis/echo-api?api-version=2021-08-01

{
  "id": "/apis/echo-api",
  "name": "echo-api",
  "properties": {
      "displayName": "Echo API",
      "serviceUrl": "http://echoapi.cloudapp.net/api",
      "protocols": [
          "https"
      ],
      "description": "7",
      "path": "echo",
      "authenticationSettings": null,
      "subscriptionKeyParameterNames": null,
      "subscriptionRequired": true
  }
}
```

The difference that matters is that the CmdLet adds the revision to the URI: `echo-api%3Brev%3D1?api-version=2021-08-01` which the other does not `echo-api?api-version=2021-08-01`. \
But when adding the revision to the Postman request, it fails with the same error code.

### Preliminary Conclusion

This means that the CmdLet adds unnecessary information to the URI which causes it to break when using this custom role. If the CmdLet is executed with a user that has the role `Contributor` on the apim instance, it works. Does that mean that it is a combition of two problem?:
 - The CmdLet adds unnecessary revision information causing it to behave in a different way than the Azure Portal.
 - There's a problem with the RBAC permission inheritence. If you have write access on a certain level in the hierarchy, you should have write access to everything below that point. That's how it works if you get permissions granted at the resource group level, or the apim instance level. But for some reason that does not apply on the revisions below the apis.

## Steps for repro

### Install the pre-requisites

PowerShell pre-reqs

```PowerShell
# Use the latest AZ module
Install-Module -Name Az -Scope CurrentUser -Repository PSGallery -Force
# Install bicep to deploy an APIM instance
winget install -e --id Microsoft.Bicep
```

### Provision an APIM instance

Provision an APIM instance. I used this [this bicep template](https://github.com/peterlil/script-and-templates/tree/main/azure-services/api-management) to do so:

```PowerShell
# Log in to Azure
Connect-AzAccount -UseDeviceAuthentication

# clone the repo and run the bicep template
pushd /workspaces
git clone https://github.com/peterlil/script-and-templates.git
cd script-and-templates/azure-services/api-management

$location="Sweden Central"
$templateFile="api-management.bicep"
$resourceGroupName="apim-gap-repro-rg"
$apimInstanceName="apim-gap-repro-apim"
$apimPublisherName="John Doe"
$apimPublisherEmail="<publisher-email>" # TODO: Set this manually

New-AzSubscriptionDeployment `
    -Location $location `
    -TemplateFile $templateFile `
    -locationFromTemplate $location `
    -resourceGroupName $resourceGroupName `
    -apimInstanceName $apimInstanceName `
    -apimPublisherEmail $apimPublisherEmail `
    -apimPublisherName  $apimPublisherName

popd

```

### Create the read-only apim dev role

```PowerShell
$apimDevRoleName = "APIM ReadOnly dev"
$role = Get-AzRoleDefinition -Name "API Management Service Reader Role"
$subscriptionId=(Get-AzSubscription).Id

$role.Id = $null
$role.Name = $apimDevRoleName
$role.Description = "Read-only access to service"
$role.Actions.RemoveRange(0,$role.Actions.Count)
$role.NotActions.RemoveRange(0,$role.NotActions.Count)
$role.DataActions.RemoveRange(0,$role.DataActions.Count)
$role.NotDataActions.RemoveRange(0,$role.NotDataActions.Count)
$role.Actions.Add("Microsoft.ApiManagement/service/read")
$role.Actions.Add("Microsoft.Authorization/*/read")
$role.Actions.Add("Microsoft.Insights/alertRules/*")
$role.Actions.Add("Microsoft.ResourceHealth/availabilityStatuses/read")
$role.Actions.Add("Microsoft.Resources/deployments/*")
$role.Actions.Add("Microsoft.Resources/subscriptions/resourceGroups/read")
$role.Actions.Add("Microsoft.Support/*")
$role.Actions.Add("Microsoft.ApiManagement/service/policySnippets/read")
$role.Actions.Add("Microsoft.ApiManagement/service/loggers/read")
$role.NotActions.Add("Microsoft.ApiManagement/service/users/keys/read")
$role.AssignableScopes.Clear()
$role.AssignableScopes.Add("/subscriptions/$subscriptionId/resourceGroups/$resourceGroupName")
New-AzRoleDefinition -Role $role
```

### Create the limited Azure RBAC Role

```PowerShell
$customRoleName="APIM Role Scoped down to one API"
$subscriptionId=(Get-AzSubscription).Id
$apiMgmtResourceGroup=$resourceGroupName
$apiMgmtName=$apimInstanceName

$apimContext = New-AzApiManagementContext -ResourceGroupName $apiMgmtResourceGroup -ServiceName $apiMgmtName
$api = Get-AzApiManagementApi -Context $apimContext -Name "Echo Api" # Error when working with the specified module versions. Reverting to latest

# create a custom role for the new Api
$readerRoleName = "API Management Service Reader Role"
$role2 = Get-AzRoleDefinition -Name $readerRoleName
$role2.Id = $null
$role2.NotActions.Clear()
$role2.AssignableScopes.Clear()
$role2.AssignableScopes.Add("/subscriptions/$($subscriptionId)/resourceGroups/$($apiMgmtResourceGroup)/providers/Microsoft.ApiManagement/service/$($apiMgmtName)/apis/$($api.ApiId)")
$role2.AssignableScopes.Add("/subscriptions/$($subscriptionId)/resourceGroups/$($apiMgmtResourceGroup)/providers/Microsoft.ApiManagement/service/$($apiMgmtName)/loggers/$($api.ApiId)")
$role2.Name = $customRoleName
$role2.Actions.Add('Microsoft.ApiManagement/service/apis/*')
$role2.Actions.Add('Microsoft.ApiManagement/service/loggers/read')

# Microsoft.ApiManagement/service/apis/revisions/read
# Microsoft.ApiManagement/service/apis/revisions/delete

Write-Host "Role Actions: $($role2.Actions) " -ForegroundColor Blue

New-AzRoleDefinition -Role $role2

```

### Create user

User | Description
-----|------------
apim-limited-user | User with the custom role `APIM Role Scoped down to one API`

I created the user in the Azure Portal. #NoScript

### Assign the roles to the user

```PowerShell
New-AzRoleAssignment `
    -SignInName "$limitedUser@$tenantDomain" `
    -RoleDefinitionName $readerRoleName `
    -Scope "/subscriptions/$($subscriptionId)/resourceGroups/$($apiMgmtResourceGroup)/providers/Microsoft.ApiManagement/service/$($apiMgmtName)"

New-AzRoleAssignment `
    -SignInName "$limitedUser@$tenantDomain" `
    -RoleDefinitionName $customRoleName `
    -Scope "/subscriptions/$($subscriptionId)/resourceGroups/$($apiMgmtResourceGroup)/providers/Microsoft.ApiManagement/service/$($apiMgmtName)/apis/$($api.ApiId)"
Start-Sleep -Seconds 3
New-AzRoleAssignment `
    -SignInName "$limitedUser@$tenantDomain" `
    -RoleDefinitionName $customRoleName `
    -Scope "/subscriptions/$($subscriptionId)/resourceGroups/$($apiMgmtResourceGroup)/providers/Microsoft.ApiManagement/service/$($apiMgmtName)/loggers/$($api.ApiId)"
```

### Verify discrepencies
This script should fail.
```PowerShell
Connect-AzAccount -UseDeviceAuthentication

$context = New-AzApiManagementContext -ResourceGroupName "$resourceGroupName" -ServiceName "$apimInstanceName"

$apis = Get-AzApiManagementApi -Context $context

$api = $apis | Where-Object { $_.ApiId -EQ 'echo-api' }

Write-Host "original api description: $($api.Description)"
# edit some api informations
$api.Description = ([int]($api.Description) + 1).ToString()
Write-Host "new api description: $($api.Description)"

$result = Set-AzApiManagementApi -InputObject $api -PassThru
```

Then trying to change the api through CmdLet parameters instead of sending the full object. This works, which means that there should not really be a permission error in the above command.
```PowerShell
Connect-AzAccount -UseDeviceAuthentication

$context = New-AzApiManagementContext -ResourceGroupName "$resourceGroupName" -ServiceName "$apimInstanceName"

Set-AzApiManagementApi -Context $context -ApiId 'echo-api' -Description '201'
```

Make the update with Azure CLI, also works.
```bash
az login
resourceGroup=apim-gap-repro-rg
apimInstanceName=apim-gap-repro-apim

az apim api show -g $resourceGroup --service-name $apimInstanceName --api-id echo-api

az apim api update -g $resourceGroup --service-name $apimInstanceName --api-id echo-api --description "200"

```



