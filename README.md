# apim-gap-repro
Repo for repro a difference in Portal functionality vs functionality in the PowerShell CmdLet `Set-AzApiManagementApi`.

## Problem statement and background
The PowerShell CmdLet `Set-AzApiManagementApi` fails and returns the error code `AuthorizationFailed` as per the image below when updating the description of an API, when executed by a user in a custom RBAC role defined below.

<img width="741" alt="image" src="https://user-images.githubusercontent.com/29121387/212117094-36ae76c0-aa9d-4892-b588-467117b38e34.png">

In this context the organization has one central team managing Azure API Management (apim) as a platform to be used by many workload teams. The workload teams publishes and maintains their own apis in the apim platform, but they should not be able to see nor modify any other workload teams's apis. To accomplish this, the devs of the workload teams are all assigned a role with only the necessary reading permissions on the apim instance, let's call this role `APIM Common Dev`, and it's defined as:
```json
{
  "Name": "APIM Common Dev",
  "Id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "IsCustom": true,
  "Description": "Read-only access to service and APIs",
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

When following the [documentation](https://learn.microsoft.com/en-us/rest/api/apimanagement/current-ga/apis/create-or-update?tabs=HTTP) and posting an update with Postman is also no problem.

<img width="1091" alt="image" src="https://user-images.githubusercontent.com/29121387/212131144-85e63795-03c4-4d10-b109-371c83403cb0.png">


## Steps for repro

### Install the pre-requisites

 PowerShell pre-reqs

```PowerShell
# Use the latest
Install-Module -Name Az -Scope CurrentUser -Repository PSGallery -Force
Install-Module Microsoft.Graph -Scope CurrentUser


# Below is when I tried to specify exact versions
#$modules = @(
#    [pscustomobject]@{Name='Az.Accounts';Version='2.9.1';Repository='PSGallery'}
#    [pscustomobject]@{Name='Az.ApiManagement';Version='3.0.0';Repository='PSGallery'}
#    [pscustomobject]@{Name='Az.KeyVault';Version='4.5.0';Repository='PSGallery'}
#    [pscustomobject]@{Name='Az.Resources';Version='';Repository='PSGallery'}
#)
#
## Install missing modules and fail prereqs check if wrong version is installed.
#$preReqsOk = $true
#$modules | foreach-object {
#    $moduleList = Get-Module -Name $_.Name -ListAvailable
#    if($moduleList.Length -eq 0) {
#        "The module $($_.Name) was not found."
#        if([string]::IsNullOrEmpty($_.Version)) {
#            "Installing latest version of $($_.Name)"
#            Install-Module -Name $_.Name -Scope CurrentUser -Repository $_.Repository -Force
#        }
#        else {
#            "Installing version $($_.Version) of $($_.Name)"
#            Install-Module -Name $_.Name -Scope CurrentUser -Repository $_.Repository -RequiredVersion $_.Version -Force
#        }
#    }
#    else {
#        if ($moduleList.Length -eq 1) {
#            if ($moduleList[0].Version -eq $_.Version -or [string]::IsNullOrEmpty($_.Version) ) {
#                "The module $($_.Name) is already installed."
#            }
#            else {
#                "Version $($moduleList[0].Version) of module $($_.Name) is installed but version $($_.Version) is required."
#                $preReqsOk = $false
#            }
#        }
#        else {
#            throw "Not implemented"
#        }
#    }
#}
#
#if(!$preReqsOk) {
#    throw "Failed pre-requisites check."
#}

```

Bicep pre-reqs

```bash
# Fetch the latest Bicep CLI binary
curl -Lo bicep https://github.com/Azure/bicep/releases/latest/download/bicep-linux-x64
# Mark it as executable
chmod +x ./bicep
# Add bicep to your PATH (requires admin)
sudo mv ./bicep /usr/local/bin/bicep
# Verify you can now access the 'bicep' command
bicep --help
# Done!
```

### (If needed) Uninstall all Az Modules 

If wrong versions are installed, uninstall all Az modules using the script block below and redo the pre-reqs installation as per above. 

Fetched from [this article](https://learn.microsoft.com/en-us/powershell/azure/uninstall-az-ps?view=azps-9.3.0)

```PowerShell
Get-InstalledModule -Name Az -AllVersions -OutVariable AzVersions

($AzVersions |
  ForEach-Object {
    Import-Clixml -Path (Join-Path -Path $_.InstalledLocation -ChildPath PSGetModuleInfo.xml)
  }).Dependencies.Name | Sort-Object -Descending -Unique -OutVariable AzModules

$AzModules |
  ForEach-Object {
    Remove-Module -Name $_ -ErrorAction SilentlyContinue
    Write-Output "Attempting to uninstall module: $_"
    Uninstall-Module -Name $_ -AllVersions
  }

Remove-Module -Name Az -ErrorAction SilentlyContinue
Uninstall-Module -Name Az -AllVersions
```


### Provision an APIM instance

Provision an APIM instance. I used this [this bicep template](https://github.com/peterlil/script-and-templates/tree/main/azure-services/api-management) to do so:

```PowerShell
# Make sure the modules are loaded
#$modules | foreach-object {
#    if([string]::IsNullOrEmpty($_.Version)) {
#        Import-Module -Name $_.Name -Force
#    }
#    else {
#        Import-Module -Name $_.Name -RequiredVersion $_.Version -Force
#    }
#}

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
$role.Description = "Read-only access to service and APIs"
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

### Create two users

User | Description
-----|------------
apim-read-user | User with the built-in role `API Management Service Reader Role`
apim-limited-user | User with the custom role `APIM Role Scoped down to one API`


```PowerShell
Connect-MgGraph -Scopes "User.ReadWrite.All"

$readUser='apim-read-user'
$limitedUser='apim-limited-user'
$passwordProfile = @{Password = '<password>'} # Todo
$tenantDomain='<tenant-domain>' # Todo

New-MgUser `
    -AccountEnabled `
    -DisplayName $readUser `
    -PasswordProfile $passwordProfile `
    -UserPrincipalName "$readUser@$tenantDomain" `
    -MailNickName $readUser

#New-MgUser_CreateExpanded1: Code: generalException
#Message: Unexpected exception occurred while authenticating the request.

```

### Assign the roles to the users

```PowerShell

New-AzRoleAssignment `
    -SignInName "$readUser@$tenantDomain" `
    -RoleDefinitionName $readerRoleName `
    -Scope "/subscriptions/$($subscriptionId)/resourceGroups/$($apiMgmtResourceGroup)/providers/Microsoft.ApiManagement/service/$($apiMgmtName)"


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


#Get-AzRoleAssignment `
#    -SignInName "$readUser@$tenantDomain" `
#    -Scope "/subscriptions/$($subscriptionId)/resourceGroups/$($apiMgmtResourceGroup)/providers/Microsoft.ApiManagement/service/$($apiMgmtName)/apis/$($api.ApiId)"
#
#Remove-AzRoleAssignment `
#      -SignInName "$readUser@$tenantDomain" `
#      -Scope "/subscriptions/$($subscriptionId)/resourceGroups/$($apiMgmtResourceGroup)/providers/Microsoft.ApiManagement/service/$($apiMgmtName)/apis/$($api.ApiId)" `
#      -RoleDefinitionName 'API Management Service Reader Role'
#
#
#Get-AzRoleAssignment `
#    -SignInName "$readUser@$tenantDomain" `
#    -Scope "/subscriptions/$($subscriptionId)/resourceGroups/$($apiMgmtResourceGroup)/providers/Microsoft.ApiManagement/service/$($apiMgmtName)/loggers/$($api.ApiId)"
#
#Remove-AzRoleAssignment `
#      -SignInName "$readUser@$tenantDomain" `
#      -Scope "/subscriptions/$($subscriptionId)/resourceGroups/$($apiMgmtResourceGroup)/providers/Microsoft.ApiManagement/service/$($apiMgmtName)/loggers/$($api.ApiId)" `
#      -RoleDefinitionName 'API Management Service Reader Role'
```

### Verify discrepencies

Create an AAD App first to get a client id

```PowerShell
#Import-Module -Name Az.Accounts -MinimumVersion 2.8.0 -Force
#Import-Module -Name Az.ApiManagement -MinimumVersion 3.0.0 -Force
#Import-Module -Name Az.KeyVault -MinimumVersion 4.5.0 -Force

Connect-AzAccount -UseDeviceAuthentication

$context = New-AzApiManagementContext -ResourceGroupName "$resourceGroupName" -ServiceName "$apimInstanceName"

# $importApiParameter = @{
#     Context             = $context
#     ApiId               = "de-lsmtest-soap-api-v1" #"de-lsm-soap-api-v1"
#     SpecificationFormat = "Wsdl"
#     SpecificationPath   = $WsdlSpecificationPath
#     Path                = $ApiPath
#     ApiType             = 'Soap'
#     WsdlServiceName     = $WsdlServiceName
#     WsdlEndpointName    = $WsdlEndpointName
# }

# $api = Import-AzApiManagementApi @importApiParameter

$apis = Get-AzApiManagementApi -Context $context

$api = $apis | Where-Object { $_.ApiId -EQ 'echo-api' }

Write-Host "original api description: $($api.Description)"
# edit some api informations
$api.Description = ([int]($api.Description) + 1).ToString()
Write-Host "new api description: $($api.Description)"

# # call is needet to change back some api settings which are auto changed by Import-AzApiManagementApi
# # AND to change description and 'SubscriptionRequired' option
$result = Set-AzApiManagementApi -InputObject $api -PassThru
```

```PowerShell
Connect-AzAccount -UseDeviceAuthentication

$context = New-AzApiManagementContext -ResourceGroupName "$resourceGroupName" -ServiceName "$apimInstanceName"

Set-AzApiManagementApi -Context $context -ApiId 'echo-api' -Description '201'

```


```batch
az login
resourceGroup=apim-gap-repro-rg
apimInstanceName=apim-gap-repro-apim

az apim api show -g $resourceGroup --service-name $apimInstanceName --api-id echo-api

az apim api update -g $resourceGroup --service-name $apimInstanceName --api-id echo-api --description "200"

```
