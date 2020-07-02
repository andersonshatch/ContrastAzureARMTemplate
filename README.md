# Contrast Security + Azure App Service ARM template

ARM template to create a resource group, app service instance, and add the Contrast .NET or .NET Core extension to it.

Environment variables required by the extensions are also setup with this template.

## Components
1. The ARM template itself: [`WebSite.json`](WebSite.json)
1. The parameters to the template, using Azure Vault references: [`WebSite.parameters.json`](WebSite.parameters.json)
1. Minimal parameters to the template, if you do not want to use Azure Vault: [`WebSite.prompt-parameters.json`](WebSite.prompt-parameters.json)
1. A PowerShell script to deploy the template: [`Deploy-AzureResourceGroup.ps1`](Deploy-AzureResourceGroup.ps1)

### Contrast specifics in the template
As the provided ARM template generates many resources in addition to Contrast specifics, if you want to integrate the Contrast specific parts into an existing ARM template, the following links highlight key sections which you'll need to integrate:

- [Declaration of Parameters](/WebSite.json#L38:L81)
- [Parameter Reference to Vault Keys](/WebSite.parameters.json#L8:L39)
- [Declaration of Template Variables](/WebSite.json#L84:L97)
- [Addition of Contrast Extension](/WebSite.json#L135:L144)
- [Setup of Contrast and App Insight Environment Variables](/WebSite.json#L145:L172)

### Caveats
- **Important:** The `appSettings` variables should be set **after** the extension is installed. Setting them before will likely cause installation of the extension to fail with a `Conflict`, or `Canceled` state, [as setting the variables causes the service to restart](https://stackoverflow.com/questions/45106303/app-insights-status-monitor-extension-failing-to-deploy-with-arm-template) just as the extension installation starts.

    This template sets them after, by [deploying them as a dependent sub-resource to the App Service](/WebSite.json#L145:L171). 

    Note: settings deployed in this way **will overwrite** any settings provided in the service `siteConfig.appSettings` section.
- At this time, compatibility with App Insights is untested by Contrast engineering
- There is no compatibility when the `Snapshot Debugger` or `SQL Commands` Insight Options are enabled

## Setup
1. Define 4 secrets in Azure Key Vault: `contrastApiKey`, `contrastAgentServiceKey`, `contrastAgentUsername` and `contrastURL` -- values for these can be found by logging in to Contrast and navigating to Organization Settings -> API
(contrastURL should be scheme and host/port only, do not include `/Contrast`)
1. Set a value for `hostingPlanName` in `WebSite.parameters.json`
1. Update the 4 key vault references in `WebSite.parameters.json`

If you are not using Azure Key Vault, rename the minimal `WebSite.prompt-parameters.json` to `WebSite.parameters`, and then you will be prompted for the 4 values when running the deploy below.

## Deploy with PowerShell
1. Install the AzureRM module in PowerShell: `Install-Module AzureRM`
1. Login to your Azure instance in Powershell: `Connect-AzureRmAccount`
1. Determine which location to deploy the resources to. To see a list, run `Get-AzureRMLocation | Format-Table`
1. (Optional) Adjust the value for `ResourceGroupName` in `Deploy-AzureResourceGroup.ps1` or pass it as a parameter
1. Run the deployment script in PowerShell: `.\Deploy-AzureResourceGroup.ps1`
1. Specify a region
1. When prompted for `contrastDotnetOrDotnetCore`, enter `Dotnet` if your application is using the .NET Framework, and `DotnetCore` if your application is using .NET Core

This will create the required resources. To complete integration, you must publish your .NET or .NET Core application to this app services instance, and your application should then show up in Contrast.

## Deploy from Azure DevOps Pipeline
1. Add the following task to your Pipeline YAML file:

    ```yaml
    - task: AzureResourceManagerTemplateDeployment@3
      inputs:
        deploymentScope: 'Resource Group'
        azureResourceManagerConnection: '<Name of Azure Service Connection>'
        subscriptionId: '<Your Azure Subscription ID>'
        action: 'Create Or Update Resource Group'
        resourceGroupName: '<Your Resource Group Name>'
        location: '<Azure Deployment Location>'
        templateLocation: 'Linked artifact'
        csmFile: '<Path/to/WebSite.json>'
        csmParametersFile: '<Path/To/WebSite.parameters.json>'
        deploymentMode: 'Incremental'
        overrideParameters: '-contrastDotnetOrDotnetCore <Dotnet|DotnetCore>'
    ```
1. Replace the following 7 parameters in the above snippet:

||Placeholder|Description
|---|---|---|
|1|`<Name of Azure Service Connection>`|Name of the Azure Service Connection your pipeline can use|
|2|`<Your Azure Subscription ID>`|The Azure Subscription ID to use for this deployment|
|3|`<Your Resource Group Name>`|The Resource Group to deploy this to|
|4|`<Azure Deployment Location>`|The Azure Region to deploy this to|
|5|`<Path/to/WebSite.json>`|The path to the ARM template in your repository|
|6|`<Path/To/WebSite.parameters.json>`|The path to the ARM parameters file in your repository|
|7|`<Dotnet\|DotnetCore>`|Which Contrast agent extension to use: .NET Framework (`Dotnet`) or .NET Core (`DotnetCore`)|

The following are optional, but **recommended** parameters which can be added to the `overrideParameters` section:
||Parameter Name|Value Description
|---|---|---|
|1|`-contrastServerEnvironment`|Environment value sent to Contrast when servers are created. Valid values are `DEVELOPMENT`, `QA` or `PRODUCTION`. Default: `QA`|
|2|`-contrastApplicationGroup`|A comma separated list of application groups the application should onboard to|
|3|`-contrastServerName`|Name for the server entry in Contrast (default: hostname)|
