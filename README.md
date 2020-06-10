# Contrast Security + Azure App Services ARM template

ARM template to create a resource group, app services instance, and add the Contrast .NET and .NET Core extensions to it.

Environment variables required by the extensions are also setup with this template.

## Setup
1. Set a value for `hostingPlanName` in `WebSite.parameters.json`
1. (Optional) Adjust the value for `ResourceGroupName` in `Deploy-AzureResourceGroup.ps1` or pass it as a parameter
1. Update the 4 environment variable values beginning with `CONTRAST__` in `WebSite.json` (start around line 75) with values from your Contrast account
** Recommendation ** Use Azure Vault values instead

## Deploy
1. Install the AzureRM module in PowerShell: `Install-Module AzureRM`
1. Login to your Azure instance in Powershell: `Connect-AzureRmAccount`
1. Determine which location to deploy the resources to. To see a list, run `Get-AzureRMLocation | Format-Table`
1. Run the deployment script in PowerShell: `.\Deploy-AzureResourceGroup.ps1`
1. Specify a region