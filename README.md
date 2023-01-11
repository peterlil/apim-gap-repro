# apim-gap-repro
Repo for repro a difference in Portal functionality vs functionality in PowerShell modules

## Steps for repro

### Install the pre-requisites

 PowerShell pre-reqs

```PowerShell

$modules = @(
    [pscustomobject]@{Name='Az.Accounts';Version='2.9.1';Repository='PSGallery'}
    [pscustomobject]@{Name='Az.ApiManagement';Version='3.0.0';Repository='PSGallery'}
    [pscustomobject]@{Name='Az.KeyVault';Version='4.5.0';Repository='PSGallery'}
    [pscustomobject]@{Name='Az.Resources';Version='';Repository='PSGallery'}
)

# Install missing modules and fail prereqs check if wrong version is installed.
$preReqsOk = $true
$modules | foreach-object {
    $moduleList = Get-Module -Name $_.Name -ListAvailable
    if($moduleList.Length -eq 0) {
        "The module $($_.Name) was not found."
        if([string]::IsNullOrEmpty($_.Version)) {
            "Installing latest version of $($_.Name)"
            Install-Module -Name $_.Name -Scope CurrentUser -Repository $_.Repository -Force
        }
        else {
            "Installing version $($_.Version) of $($_.Name)"
            Install-Module -Name $_.Name -Scope CurrentUser -Repository $_.Repository -RequiredVersion $_.Version -Force
        }
    }
    else {
        if ($moduleList.Length -eq 1) {
            if ($moduleList[0].Version -eq $_.Version -or [string]::IsNullOrEmpty($_.Version) ) {
                "The module $($_.Name) is already installed."
            }
            else {
                "Version $($moduleList[0].Version) of module $($_.Name) is installed but version $($_.Version) is required."
                $preReqsOk = $false
            }
        }
        else {
            throw "Not implemented"
        }
    }
}

if(!$preReqsOk) {
    throw "Failed pre-requisites check."
}

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

### Create the lmited Azure RBAC Role

```PowerShell
$roleName="APIM Role Scoped down to one API"
$subscriptionId=(Get-AzSubscription).Id
$apiMgmtResourceGroup=$resourceGroupName
$apiMgmtName=$apimInstanceName

$apimContext = New-AzApiManagementContext -ResourceGroupName $apiMgmtResourceGroup -ServiceName $apiMgmtName
Get-AzApiManagementApi -Context $apimContext -Name "EchoApi"


# create a custom role for the new Api
$role2 = Get-AzRoleDefinition -Name "API Management Service Reader Role"
$role2.Id = $null
$role2.NotActions.Clear()
$role2.AssignableScopes.Clear()
$role2.AssignableScopes.Add("/subscriptions/$($subscriptionId)/resourceGroups/$($apiMgmtResourceGroup)/providers/Microsoft.ApiManagement/service/$($apiMgmtName)/apis/$($api.ApiId)")
$role2.AssignableScopes.Add("/subscriptions/$($subscriptionId)/resourceGroups/$($apiMgmtResourceGroup)/providers/Microsoft.ApiManagement/service/$($apiMgmtName)/loggers/$($api.ApiId)")
$role2.Name = $roleName
$role2.Actions.Add('Microsoft.ApiManagement/service/apis/*')
$role2.Actions.Add('Microsoft.ApiManagement/service/loggers/read')
Write-Host "Role Actions: $($role2.Actions) " -ForegroundColor Blue

New-AzRoleDefinition -Role $role2

```