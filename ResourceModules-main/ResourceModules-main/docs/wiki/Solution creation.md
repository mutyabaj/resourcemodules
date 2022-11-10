This section shows you how you can orchestrate a deployment using multiple resource modules.

> **Note:** In the below examples, we assume you leverage Bicep as your primary DSL (domain specific language).

---

### _Navigation_

- [Upstream workloads](#upstream-workloads)
- [Orchestration overview](#orchestration-overview)
  - [Publish-location considerations](#publish-location-considerations)
    - [Outline](#outline)
    - [Comparison](#comparison)
- [Template-orchestration](#template-orchestration)
  - [To be considered](#to-be-considered)
  - [GitHub Samples](#github-samples)
    - [Local files](#local-files)
    - [Bicep Registry](#bicep-registry)
    - [Template Specs](#template-specs)
  - [Azure DevOps Samples](#azure-devops-samples)
    - [Azure Artifacts](#azure-artifacts)
    - [Bicep Registry](#bicep-registry-1)
    - [Template Specs](#template-specs-1)
- [Pipeline-orchestration](#pipeline-orchestration)
  - [GitHub Sample](#github-sample)
    - [Multi-repository](#multi-repository)
    - [Repository structure](#repository-structure)
  - [Azure DevOps Sample](#azure-devops-sample)
    - [Multi-repository](#multi-repository-1)
    - [Repository structure](#repository-structure-1)

---

# Upstream workloads

There are several open-source repositories that leverage the CARML library today. Alongside the examples, we provide you with below, the referenced repositories are a good reference on how you can leverage CARML for larger solutions.

| Repository | Description |
| - | - |
| [AVD Accelerator](https://github.com/Azure/avdaccelerator) | AVD Accelerator deployment automation to simplify the setup of AVD (Azure Virtual Desktop) |
| [AKS Baseline Automation](https://github.com/Azure/aks-baseline-automation) | Repository for the AKS Landing Zone Accelerator program's Automation reference implementation |
| [DevOps Self-Hosted](https://github.com/Azure/DevOps-Self-Hosted) | - Create & maintain images with a pipeline using the Azure Image Builder service <p> - Deploy & maintain Azure DevOps Self-Hosted agent pools with a pipeline using Virtual Machine Scale Set|

# Orchestration overview

When it comes to deploying multi-module solutions (applications/workloads/environments/landing zone accelerators/etc.), we can differentiate two types of orchestration methods:

- **_Template-orchestration_**: These types of deployments reference individual modules from a 'main' Bicep or ARM/JSON template and use the capabilities of this template to pass parameters & orchestrate the deployments. By default, deployments are run in parallel by the Azure Resource Manager, while accounting for all dependencies defined. With this approach, the deploying pipeline only needs one deployment job that triggers the template's deployment.

   <img src="./media/SolutionCreation/templateOrchestration.png" alt="Template orchestration" height="250">

- **_Pipeline-orchestration_**: This approach uses the platform specific pipeline capabilities (for example, pipeline jobs) to trigger the deployment of individual modules, where each job deploys one module. By defining dependencies in between jobs you can make sure your resources are deployed in order. Parallelization is achieved by using a pool of pipeline agents that run the jobs, while accounting for all dependencies defined.

Both the _template-orchestration_, as well as _pipeline-orchestration_ may run a validation and subsequent deployment in the same _Azure_ subscription. This subscription should be the subscription where you want to host your production solution. However, you can extend the concept and for example, deploy the solution first to an integration and then a production subscription.

   <img src="./media/SolutionCreation/pipelineOrchestration.png" alt="Pipeline orchestration" height="400">

## Publish-location considerations

For your solution, it is recommended to reference modules from a published location, to leverage versioning and avoid the risk of breaking changes.

CARML supports publishing to different locations, either through the use of the CI environment or by locally running the same scripts leveraged by the publishing step of the CI environment pipeline, as explained next.

In either case, you may effectively decide to configure only a subset of publishing locations as per your requirements.

To help you with the decision, the following content provides you with an overview of the possibilities of each target location.

### Outline
- **Template Specs**<p>
  A [Template Spec](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-specs?tabs=azure-powershell) is an Azure resource with the purpose of storing & referencing Azure Resource Manager (ARM) templates. <p>
  When publishing Bicep modules as Template Specs, the module is compiled - and the resulting ARM template is uploaded as a Template Spec resource version to a Resource Group of your choice.
  For deployment, it is recommended to apply a [template-orchestrated](#Orchestration-overview) approach. As Bicep supports the Template-Specs as linked templates, this approach enables you to fully utilize Azure's parallel deployment capabilities.
  > **Note:** Even though the published resource is an ARM template, you can reference it in you Bicep template as a remote module like it would be native Bicep.
  > **Note:** Template Spec names have a maximum of 90 characters

- **Bicep Registry**<p>
  A [Bicep Registry](https://docs.microsoft.com/en-us/azure/azure-resource-manager/bicep/private-module-registry) is an Azure Container Registry that can be used to store & reference Bicep modules.<p>
  For deployment, it is recommended to apply a [template-orchestrated](#Orchestration-overview) approach. As Bicep supports the Bicep registry as linked templates, this approach enables you to fully utilize Azure's parallel deployment capabilities.

- **Azure DevOps Universal Packages**<p>
  A [Universal Package](https://docs.microsoft.com/en-us/azure/devops/artifacts/quickstarts/universal-packages) is a packaged folder in an Azure DevOps artifact feed.<p>
  As such, it contains the content of a CARML module 'as-is', including the template file(s), ReadMe file(s) and test file(s). <p>
  For deployment, it is recommended to use Universal Packages only for a [pipeline-orchestrated](#Orchestration-overview) approach - i.e., each job would download a single package and deploy it. <p>
  Technically, it would be possible to also use Universal Packages for the template-orchestrated approach, by downloading all packages into a specific location first, and then reference them. Given the indirect nature of this approach, this is however not recommended. (:large_orange_diamond:)
  > **Note:** Azure DevOps Universal Packages enforce _semver_. As such, it is not possible to overwrite an existing version.

### Comparison

The following table provides you with a comparison of the locations described above:

| Category | Feature | Template Specs | Bicep Registry | Universal Packages |
| - | - | - | - | - |
| Portal/UI |
| | Template can be viewed |:white_check_mark: | | |
| | Template can be downloaded | | | |
| |
| Deployment |
| | Supports [template-orchestration](./Solution%20creation#Orchestration-overview) | :white_check_mark: | :white_check_mark: | :large_orange_diamond: |
| | Supports [pipeline-orchestration](./Solution%20creation#Orchestration-overview) | :white_check_mark: | :white_check_mark: | :white_check_mark:  |
| | Supports single endpoint | | :white_check_mark: | :white_check_mark: |
| |
| Other |
| | Template can be downloaded/restored via CLI | :white_check_mark: | :white_check_mark: | :white_check_mark: |
| | Allows referencing latest [minor](./The%20CI%20environment%20-%20Publishing#how-it-works) | :white_check_mark: | :white_check_mark: | |
| | Allows referencing latest [major](./The%20CI%20environment%20-%20Publishing#how-it-works) | :white_check_mark: | :white_check_mark: | :white_check_mark: |

# Template-orchestration

The _template-orchestrated_ approach means using a _main_ or so-called _master template_ for deploying resources in Azure. This template will only contain nested deployments, where the modules - instead of directly embedding their content into the _master template_ - will be referenced by the _master template_.

With this approach, modules need to be stored in an available location, where the Azure Resource Manager (ARM) can access them. This can be achieved by storing the module templates in an accessible location like _local_, _Template Specs_ or the _Bicep Registry_.

In an enterprise environment, the recommended approach is to store these _master templates_ in a private environment, only accessible by enterprise resources. Thus, only trusted authorities can have access to these files.

## To be considered

Once you start building a solution using this library, you may wonder how best to start. Following, you can find some points that can accelerate your experience:

- Use the [VS-Code extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-bicep) for Bicep to enable DSL-native features such as auto-complete. Metadata implemented in the modules are automatically loaded through the extension.
- Use the readme
  - If you don't know how to use an object/array parameter, you can check if the module's ReadMe file specifies any 'Parameter Usage' block for the given parameter ([example](https://github.com/Azure/ResourceModules/blob/main/modules/Microsoft.AnalysisServices/servers/readme.md#parameter-usage-tags)) - or - check the module's `Deployment Examples` ([example](https://github.com/Azure/ResourceModules/blob/main/modules/Microsoft.AnalysisServices/servers/readme.md#deployment-examples)).
  - In general, take note of the `Deployment Examples` specified in each module's ReadMe file, as they provide you with rich & tested examples of how a given module can be deployed ([example](https://github.com/Azure/ResourceModules/blob/main/modules/Microsoft.AnalysisServices/servers/readme.md#deployment-examples)). An easy way to get started is to copy one of the examples and then adjust it to your needs.
- Note the outputs that are returned by each module.
  - If an output you need isn't available, you have 2 choices:
    1. Add the missing output to the module
    1. Reference the deployed resource using the `existing` keyword (Note: You cannot reference the same resource as both a new deployment & `existing`. To make this work, you have to move the `existing` reference into it's own `.bicep` file).

## GitHub Samples

<details>
<summary>Referencing <b>local files</b></summary>

### Local files

The following example shows how you could orchestrate a deployment of multiple resources using local module references. In this example, we will deploy a resource group with a Network Security Group (NSG), and use them in a subsequent VNET deployment.

```bicep
targetScope = 'subscription'

// ================ //
// Input Parameters //
// ================ //

// RG parameters
@description('Optional. The name of the resource group to deploy')
param resourceGroupName string = 'validation-rg'

@description('Optional. The location to deploy into')
param location string = deployment().location

// =========== //
// Deployments //
// =========== //

// Resource Group
module rg 'modules/Microsoft.Resources/resourceGroups/deploy.bicep' = {
  name: 'registry-rg'
  params: {
    name: resourceGroupName
    location: location
  }
}

// Network Security Group
module nsg 'modules/Microsoft.Network/networkSecurityGroups/deploy.bicep' = {
  name: 'registry-nsg'
  scope: resourceGroup(resourceGroupName)
  params: {
    name: 'defaultNsg'
  }
  dependsOn: [
    rg
  ]
}

// Virtual Network
module vnet 'modules/Microsoft.Network/virtualNetworks/deploy.bicep' = {
  name: 'registry-vnet'
  scope: resourceGroup(resourceGroupName)
  params: {
    name: 'defaultVNET'
    addressPrefixes: [
      '10.0.0.0/16'
    ]
    subnets: [
      {
        name: 'PrimarySubnet'
        addressPrefix: '10.0.0.0/24'
        networkSecurityGroupName: nsg.name
      }
      {
        name: 'SecondarySubnet'
        addressPrefix: '10.0.1.0/24'
        networkSecurityGroupName: nsg.name
      }
    ]
  }
}
```

</details>

<details>
<summary>Referencing a <b>Bicep Registry</b></summary>

### Bicep Registry

The following sample shows how you could orchestrate a deployment of multiple resources using modules from a private Bicep Registry. In this example, we will deploy a resource group with a Network Security Group (NSG), and use them in a subsequent VNET deployment.

> **Note**: the preferred method to publish modules to the Bicep registry is to leverage the [CI environment](./The%20CI%20environment) provided in this repository. However, this option may not be applicable to all scenarios (ref e.g., the [Consume library](./Getting%20started%20-%20Scenario%201%20Consume%20library) section). As an alternative, the same [`Publish-ModuleToPrivateBicepRegistry.ps1`](https://github.com/Azure/ResourceModules/blob/main/utilities/pipelines/resourcePublish/Publish-ModuleToPrivateBicepRegistry.ps1) script leveraged by the publishing step of the CI environment pipeline can also be run locally.

```bicep
targetScope = 'subscription'

// ================ //
// Input Parameters //
// ================ //

// RG parameters
@description('Optional. The name of the resource group to deploy')
param resourceGroupName string = 'validation-rg'

@description('Optional. The location to deploy into')
param location string = deployment().location

// =========== //
// Deployments //
// =========== //

// Resource Group
module rg 'br/modules:microsoft.resources.resourcegroups:1.0.0' = {
  name: 'registry-rg'
  params: {
    name: resourceGroupName
    location: location
  }
}

// Network Security Group
module nsg 'br/modules:microsoft.network.networksecuritygroups:1.0.0' = {
  name: 'registry-nsg'
  scope: resourceGroup(resourceGroupName)
  params: {
    name: 'defaultNsg'
  }
  dependsOn: [
    rg
  ]
}

// Virtual Network
module vnet 'br/modules:microsoft.network.virtualnetworks:1.0.0' = {
  name: 'registry-vnet'
  scope: resourceGroup(resourceGroupName)
  params: {
    name: 'defaultVNET'
    addressPrefixes: [
      '10.0.0.0/16'
    ]
    subnets: [
      {
        name: 'PrimarySubnet'
        addressPrefix: '10.0.0.0/24'
        networkSecurityGroupName: nsg.name
      }
      {
        name: 'SecondarySubnet'
        addressPrefix: '10.0.1.0/24'
        networkSecurityGroupName: nsg.name
      }
    ]
  }
}
```

The example assumes you are using a [`bicepconfig.json`](https://docs.microsoft.com/en-us/azure/azure-resource-manager/bicep/bicep-config) configuration file like:

```json
{
    "moduleAliases": {
        "br": {
            "modules": {
                "registry": "<registryName>.azurecr.io",
                "modulePath": "bicep/modules"
            }
        }
    }
}
```

</details>

<details>
<summary>Referencing <b>Template-Specs</b></summary>

### Template Specs

The following example shows how you could orchestrate a deployment of multiple resources using template specs. In this example, we will deploy a resource group with a Network Security Group (NSG), and use them in a subsequent VNET deployment.

> **Note**: the preferred method to publish modules to template-specs is to leverage the [CI environment](./The%20CI%20environment) provided in this repository. However, this option may not be applicable to all scenarios (ref e.g., the [Consume library](./Getting%20started%20-%20Scenario%201%20Consume%20library) section). As an alternative, the same [Publish-ModuleToTemplateSpec.ps1](https://github.com/Azure/ResourceModules/blob/main/utilities/pipelines/resourcePublish/Publish-ModuleToTemplateSpec.ps1) script leveraged by the publishing step of the CI environment pipeline can also be run locally.

```bicep
targetScope = 'subscription'

// ================ //
// Input Parameters //
// ================ //

// RG parameters
@description('Optional. The name of the resource group to deploy')
param resourceGroupName string = 'validation-rg'

@description('Optional. The location to deploy into')
param location string = deployment().location

// =========== //
// Deployments //
// =========== //

// Resource Group
module rg 'ts/modules:microsoft.resources.resourcegroups:1.0.0' = {
  name: 'registry-rg'
  params: {
    name: resourceGroupName
    location: location
  }
}

// Network Security Group
module nsg 'ts/modules:microsoft.network.networksecuritygroups:1.0.0' = {
  name: 'registry-nsg'
  scope: resourceGroup(resourceGroupName)
  params: {
    name: 'defaultNsg'
  }
  dependsOn: [
    rg
  ]
}

// Virtual Network
module vnet 'ts/modules:microsoft.network.virtualnetworks:1.0.0' = {
  name: 'registry-vnet'
  scope: resourceGroup(resourceGroupName)
  params: {
    name: 'defaultVNET'
    addressPrefixes: [
      '10.0.0.0/16'
    ]
    subnets: [
      {
        name: 'PrimarySubnet'
        addressPrefix: '10.0.0.0/24'
        networkSecurityGroupName: nsg.name
      }
      {
        name: 'SecondarySubnet'
        addressPrefix: '10.0.1.0/24'
        networkSecurityGroupName: nsg.name
      }
    ]
  }
}
```

The example assumes you are using a [`bicepconfig.json`](https://docs.microsoft.com/en-us/azure/azure-resource-manager/bicep/bicep-config) configuration file like:

```json
{
    "moduleAliases": {
        "ts": {
            "modules": {
                "subscription": "<<subscriptionId>>",
                "resourceGroup": "artifacts-rg"
            }
        }
    }
}
```

</details>
<p>

## Azure DevOps Samples

Below, you can find samples which can be used to orchestrate deployments in Azure DevOps. Unlike the above mentioned GitHub sample all samples are using a modified and standalone version of `pipeline.deploy.yml` which can be found with working examples [here](https://github.com/segraef/Platform/blob/main/.azuredevops/pipelineTemplates/jobs.solution.deploy.yml) (`/.azuredevops/pipelineTemplates/jobs.solution.deploy.yml`) which is capable of consuming Modules via any publishing option you prefer.

> Note: The following samples show Azure Pipelines stored in GitHub and use a GitHub service connection endpoint and hence get triggered externally. This is out of pure convenience and can also be stored on Azure Repos directly and be triggered in the same way.

<details>
<summary><b>Reference Samples</b></summary>

All DevOps samples can be found here as a reference: [Litware/Platform](https://github.com/segraef/Platform/).

</details>

<details>
<summary><b>Repository structure</b></summary>

   <img src="./media/SolutionCreation/ADOSolutionTestFolderStructure.png" alt="Repository Structure" height="400">

> <b>Note:</b> This repository structure mimics a Platform deployment aligned to a resource group structure like in [AzOps](https://github.com/Azure/AzOps#output). For the following samples the resource group `prfx-conn-ae-network-hub-rg` is used.

</details>

<details>
<summary>Referencing <b>Azure Artifacts</b></summary>

### Azure Artifacts

1. The _public_ **Azure/ResourceModules** repository is being fetched and could potentially be used to leverage utilities but is not necessary since only Azure Artifacts and parameter files from this local repository are used.
2. This sample is creating a Resource Group, an NSG, a Route Table and a Virtual Network.
   1. Job: **WhatIf**
      1. Verify deployment.
   2. Job: **Deploy multi-repo solution**
     1. Checkout 'Azure/ResourceModules' repo in a nested folder `ResourceModules`
     2. Checkout 'Litware/Platform' repository containing the parameter files in a nested folder - `Platform`
     3. Download specific Modules Artifacts.
     4. Deploy resources in target Azure subscription using downloaded Artifacts.

```YAML
name: 'prfx-conn-ae-network-hub-rg (Artifacts)'

pr: none

trigger:
  batch: true
  branches:
    include:
      - main
  paths:
    include:
      - root (b3b845c6-2a30-6f4c-62d3-a8b417cb9173)/prfx-connectivity-ae (3e51c849-d082-4b01-9385-455f253a5729)/prfx-conn-ae-network-hub-rg/*

variables:
  - template: /settings.yml
  - template: pipeline.variables.yml
  - template: /.azuredevops/pipelineTemplates/jobs.artifacts.variables.yml

stages:
  - stage:
    displayName: WhatIf
    jobs:
      - template: /.azuredevops/pipelineTemplates/jobs.solution.deploy.yml
        parameters:
          jobName: bicep
          displayName: 'Bicep Deployment'
          moduleName: $(RGModuleName)
          moduleTestFilePath: '$(resourceGroupName)/parameters.json'
          whatif: true

  - stage:
    displayName: Deploy
    jobs:
      - template: /.azuredevops/pipelineTemplates/jobs.solution.deploy.yml
        parameters:
          jobName: bicep
          displayName: 'Bicep Deployment'
          moduleName: $(RGModuleName)
          moduleTestFilePath: '$(resourceGroupName)/parameters.json'
```

</details>

<details>
<summary>Referencing a <b>Bicep Registry</b></summary>

### Bicep Registry

1. The _public_ **Azure/ResourceModules** repository is being fetched and could potentially be used to leverage utilities but is not necessary since only Bicep Modules from the remote Bicep registry and parameter files from this local repository are used.
2. This sample is creating a Resource Group, an NSG, a Route Table and a Virtual Network.
   1. Job: **WhatIf**
      1. Verify deployment.
   2. Job: **Deploy multi-repo solution**
     1. Checkout 'Azure/ResourceModules' repo in a nested folder `ResourceModules`
     2. Checkout 'Litware/Platform' repository containing the parameter files in a nested folder - `Platform`
     3. Deploy resources in target Azure subscription using remote Bicep Registry Modules.


```YAML
name: 'prfx-conn-ae-network-hub-rg (Bicep Registry)'

pr: none

trigger:
  batch: true
  branches:
    include:
      - main
  paths:
    include:
      - root (b3b845c6-2a30-6f4c-62d3-a8b417cb9173)/prfx-connectivity-ae (3e51c849-d082-4b01-9385-455f253a5729)/prfx-conn-ae-network-hub-rg/*

variables:
  - template: /settings.yml
  - template: pipeline.variables.yml
  - template: /.azuredevops/pipelineTemplates/jobs.artifacts.variables.yml

stages:
  - stage:
    displayName: WhatIf
    jobs:
      - template: /.azuredevops/pipelineTemplates/jobs.solution.deploy.yml
        parameters:
          jobName: bicep
          displayName: 'Bicep Deployment'
          modulePath: '$(resourceGroupName)/deploy.br.bicep'
          moduleTestFilePath: '$(resourceGroupName)/parameters.json'
          whatif: true

  - stage:
    displayName: Deploy
    jobs:
      - template: /.azuredevops/pipelineTemplates/jobs.solution.deploy.yml
        parameters:
          jobName: bicep
          displayName: 'Bicep Deployment'
          modulePath: '$(resourceGroupName)/deploy.br.bicep'
          moduleTestFilePath: '$(resourceGroupName)/parameters.json'
```

</details>

<details>
<summary>Referencing <b>Template Specs</b></summary>

### Template Specs

1. The _public_ **Azure/ResourceModules** repository is being fetched and could potentially be used to leverage utilities but is not necessary since only Template Specs and parameter files from this local repository are used.
2. This sample is creating a Resource Group, an NSG, a Route Table and a Virtual Network.
   1. Job: **WhatIf**
      1. Verify deployment.
   2. Job: **Deploy multi-repo solution**
     1. Checkout 'Azure/ResourceModules' repo in a nested folder `ResourceModules`
     2. Checkout 'Litware/Platform' repository containing the parameter files in a nested folder - `Platform`
     3. Deploy resources in target Azure subscription using Template Specs.


```YAML
name: 'prfx-conn-ae-network-hub-rg (Template Spec)'

pr: none

trigger:
  batch: true
  branches:
    include:
      - main
  paths:
    include:
      - root (b3b845c6-2a30-6f4c-62d3-a8b417cb9173)/prfx-connectivity-ae (3e51c849-d082-4b01-9385-455f253a5729)/prfx-conn-ae-network-hub-rg/*

variables:
  - template: /settings.yml
  - template: pipeline.variables.yml

stages:
  - stage:
    displayName: WhatIf
    jobs:
      - template: /.azuredevops/pipelineTemplates/jobs.solution.deploy.yml
        parameters:
          jobName: bicep
          displayName: 'Bicep Deployment'
          modulePath: '$(resourceGroupName)/deploy.ts.bicep'
          moduleTestFilePath: '$(resourceGroupName)/parameters.json'
          whatif: true

  - stage:
    displayName: Deploy
    jobs:
      - template: /.azuredevops/pipelineTemplates/jobs.solution.deploy.yml
        parameters:
          jobName: bicep
          displayName: 'Bicep Deployment'
          modulePath: '$(resourceGroupName)/deploy.ts.bicep'
          moduleTestFilePath: '$(resourceGroupName)/parameters.json'

```

</details>


# Pipeline-orchestration

The modules provided in this repo can be orchestrated to create more complex infrastructures, and as such, reusable solutions or products. To deploy resources, the pipeline-orchestration approach leverages the modules & pipeline templates of the 'ResourceModules' repository. Each pipeline job deploys one instance of a resource and the order of resources deployed in a multi-module solution is controlled by specifying dependencies in the pipeline itself.

## GitHub Sample

<details>
<summary><b>Multi-repository approach</b></summary>

### Multi-repository

1. Below, you can find an example which makes use of multiple repositories to orchestrate the deployment (also known as a _multi-repository_ approach) in GitHub.
1. It fetches the _public_ **Azure/ResourceModules** repo for consuming bicep modules and uses the parameter files present in the _private_ **Contoso/MultiRepoTest** repo for deploying infrastructure.
1. This example is creating a Resource group, an NSG and a VNet -
    1. Job: **Deploy multi-repo solution**
        1. Checkout 'Azure/ResourceModules' repo at root of the agent
        1. Set environment variables for the agent
        1. Checkout 'contoso/MultiRepoTest' repo containing the parameter files in a nested folder - "MultiRepoTestParentFolder"
        1. Deploy resource group in target Azure subscription
        1. Deploy network security group
        1. Deploy virtual network A

### Repository structure

   <img src="./media/SolutionCreation/MultiRepoTestFolderStructure.png" alt="Repository Structure" height="300">


```YAML
name: 'Multi-Repo solution deployment'

on:
  push:
    branches:
      - main
    paths:
      - 'network-hub-rg/Parameters/**'
      - '.github/workflows/network-hub.yml'

env:
  AZURE_CREDENTIALS: ${{ secrets.AZURE_CREDENTIALS }}
  removeDeployment: false
  variablesPath: 'settings.yml'

jobs:
  job_deploy_multi_repo_solution:
    runs-on: ubuntu-20.04
    name: 'Deploy multi-repo solution'
    steps:
      - name: 'Checkout ResourceModules repo at the root location'
        uses: actions/checkout@v3
        with:
          repository: 'Azure/ResourceModules'
          fetch-depth: 0

     - name: Set environment variables
        uses: ./.github/actions/templates/setEnvironmentVariables
        with:
          variablesPath: ${{ env.variablesPath }}

      - name: 'Checkout MultiRepoTest repo in a nested MultiRepoTestParentFolder'
        uses: actions/checkout@v3
        with:
          repository: 'contoso/MultiRepoTest'
          fetch-depth: 0
          path: 'MultiRepoTestParentFolder'

      - name: 'Deploy resource group'
        uses: ./.github/actions/templates/validateModuleDeployment
        with:
          templateFilePath: './modules/Microsoft.Resources/resourceGroups/deploy.bicep'
          parameterFilePath: './MultiRepoTestParentFolder/network-hub-rg/Parameters/ResourceGroup/parameters.json'
          location: '${{ env.defaultLocation }}'
          resourceGroupName: '${{ env.resourceGroupName }}'
          subscriptionId: '${{ secrets.ARM_SUBSCRIPTION_ID }}'
          managementGroupId: '${{ secrets.ARM_MGMTGROUP_ID }}'
          removeDeployment: $(removeDeployment)

      - name: 'Deploy network security group'
        uses: ./.github/actions/templates/validateModuleDeployment
        with:
          templateFilePath: './modules/Microsoft.Network/networkSecurityGroups/deploy.bicep'
          parameterFilePath: './MultiRepoTestParentFolder/network-hub-rg/Parameters/NetworkSecurityGroups/parameters.json'
          location: '${{ env.defaultLocation }}'
          resourceGroupName: '${{ env.resourceGroupName }}'
          subscriptionId: '${{ secrets.ARM_SUBSCRIPTION_ID }}'
          managementGroupId: '${{ secrets.ARM_MGMTGROUP_ID }}'
          removeDeployment: $(removeDeployment)

      - name: 'Deploy virtual network A'
        uses: ./.github/actions/templates/validateModuleDeployment
        with:
          templateFilePath: './modules/Microsoft.Network/virtualNetworks/deploy.bicep'
          parameterFilePath: './MultiRepoTestParentFolder/network-hub-rg/Parameters/VirtualNetwork/vnet-A.parameters.json'
          location: '${{ env.defaultLocation }}'
          resourceGroupName: '${{ env.resourceGroupName }}'
          subscriptionId: '${{ secrets.ARM_SUBSCRIPTION_ID }}'
          managementGroupId: '${{ secrets.ARM_MGMTGROUP_ID }}'
          removeDeployment: $(removeDeployment)
```

> 1. 'Azure/ResourceModules' repo has been checked out at the root location intentionally because GitHub Actions expect the underlying utility scripts and variables at a specific location.
> 1. 'contoso/MultiRepoTest' repo has been checked out in a nested folder, called  "MultiRepoTestParentFolder", to distinguish it from the folders of the other repo in the agent, but can also be downloaded at the root location if desired.

</details>

## Azure DevOps Sample

<details>
<summary><b>Multi-repository approach</b></summary>

### Multi-repository

1. Fetching the _public_ **Azure/ResourceModules** repository for consuming Modules and using the parameter files present in the _private_ **Litware/Platform** repo for deploying infrastructure.
2. This sample is creating a Resource Group, an NSG, a Route Table and a Virtual Network -
    1. Job: **Deploy multi-repo solution**
        1. Checkout 'Azure/ResourceModules' repo in a nested folder `ResourceModules`
        2. Checkout 'Litware/Platform' repository containing the parameter files in a nested folder - `Platform`
        3. Deploy resources in target Azure subscription

### Repository structure

   <img src="./media/SolutionCreation/ADOSolutionTestFolderStructure.png" alt="Repository Structure" height="400">

> <b>Note:</b> This repository structure mimics a Platform deployment aligned to a resource group structure like in [AzOps](https://github.com/Azure/AzOps#output). For the following samples the resource group `prfx-conn-ae-network-hub-rg` is used.

```YAML
name: 'prfx-conn-ae-network-hub-rg'

pr: none

trigger:
  batch: true
  branches:
    include:
      - main
  paths:
    include:
      - root (b3b845c6-2a30-6f4c-62d3-a8b417cb9173)/prfx-connectivity-ae (3e51c849-d082-4b01-9385-455f253a5729)/prfx-conn-ae-network-hub-rg/*

variables:
  - template: /settings.yml
  - template: pipeline.variables.yml

resources:
  repositories:
  - repository: modules
    name: $(modulesRepository)
    ref: $(ref)
    endpoint: segraef
    type: github

stages:
  - stage:
    displayName: Deploy
    jobs:
      - template: /.azuredevops/pipelineTemplates/jobs.solution.deploy.yml
        parameters:
          jobName: resourceGroups
          displayName: 'Resource Group'
          modulePath: '/modules/Microsoft.Resources/resourceGroups/deploy.bicep'
          moduleTestFilePath: '$(resourceGroupName)/parameters.json'
          checkoutRepositories:
            - modules
      - template: /.azuredevops/pipelineTemplates/jobs.solution.deploy.yml
        parameters:
          jobName: networkSecurityGroups
          displayName: 'Network Security Groups'
          modulePath: '/modules/Microsoft.Network/networkSecurityGroups/deploy.bicep'
          moduleTestFilePath: '$(resourceGroupName)/networkSecurityGroups/parameters.json'
          checkoutRepositories:
            - modules
      - template: /.azuredevops/pipelineTemplates/jobs.solution.deploy.yml
        parameters:
          jobName: routeTables
          displayName: 'Route Tables'
          modulePath: '/modules/Microsoft.Network/routeTables/deploy.bicep'
          moduleTestFilePath: '$(resourceGroupName)/routeTables/parameters.json'
          checkoutRepositories:
            - modules
      - template: /.azuredevops/pipelineTemplates/jobs.solution.deploy.yml
        parameters:
          jobName: virtualNetworks
          displayName: 'Virtual Networks'
          modulePath: '/modules/Microsoft.Network/virtualNetworks/deploy.bicep'
          moduleTestFilePath: '$(resourceGroupName)/virtualNetworks/parameters.json'
          checkoutRepositories:
            - modules
          dependsOn:
            - networkSecurityGroups
            - routeTables
```

> 1. `Azure/ResourceModules` repo has been checked out in a nested folder called `ResourceModules` (unlike in the above mentioned GitHub sample workflow and due to restrictions in order to support all publishing options in ADO.)
> 1. `Litware/Platform` repo has been checked out in a nested folder, called `Platform`, to distinguish it from the folders of the other repo in the agent, and in order to support multiple repositories.

</details>
