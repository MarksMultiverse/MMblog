+++
title = 'Automate Azure Image Builder in Bicep with custom builder resource group'
date = 2024-02-16T10:00:46+02:00
draft = false
tags = ["Azure", "Automation", "Bicep", "AVD", "AIB"]
categories = ["Bicep"]
+++

Of course we want to secure and keep our Azure tenant tidy with the help of Azure Policies. And without any hesitation I preach to automate everything! But what if these two conflict with one another? What if there is an Azure Policy in place that demands a naming standard or tags for resource groups that causes a Azure Image Builder deployment to fail? \
With Azure Image Builder (AIB) we can automate the process of building images for use in an Azure Virtual Desktop environment for instance. AIB automatically creates a temporary resource group to store temporary resources which it needs to build the image (storage account, vnet ,vm, disk, etc.). When the build is complete Azure deletes most of these resources. This build resource group is given a random name that starts with IT_. When you have policies in place that enforce a certain naming convention of require certain tags on a resource group the AIB build will fail. There is a way to make sure that the resource group makes use of the right naming convention and tags.

So this blog covers two aspects:
1. Automate AIB in Bicep
2. Customize the build resource group for AIB

## Steps

Steps involved:
- Create resource groups
- Create user assigned managed identity
- Create custom role
- Assign the custom role to the managed identity
- Create Azure Compute Gallery
- Create Gallery Image
- Create Image Template
- Build Image Template

## Overview

In this example all resource groups must start with `RG-` and require the tags `Project` and `Responsible`. \
We would like to see the following resources in our Azure subscription:


## Bicep

The complete Bicep files can be found in my GitHub repository <a href="https://github.com/MarksMultiverse/AIB-bicep">here</a>
The automation contains five Bicep files:
- aib-main.bicep
- aib-rgs.bicep
- aib-role.bicep
- aib-roletemp.bicep
- aib-image.bicep

This deployment results in the following resources in Azure:
![Azure resources](/BICEP-automate-AIB/resources.jpg)

### The main Bicep file

This is the main bicep file that will be deployed to Azure. Please be aware that you have to change the `subscriptionID` value with tour Azure subscription ID.

```bicep
targetScope = 'subscription'
param location string = 'westeurope'
param subscriptionID string = '<YOUR_SUBSCRIPTION_ID>'
param baseTime string = utcNow()

param tags object = {
  Project: 'Automate-AIB-Deployment'
  Responsible: 'Mark Multiverse'
}
param RGnameAVDimage string = 'RG-image'
param RGnameAIB string = 'RG-temp'
param azureImageBuilderName string = 'myImageTemplate'
param runOutputName string = 'Win11test'
param galleryName string = 'myGallery'
param galleryImageName string = 'myGalleryImage'

// Create resource groups
module resourcegroups 'aib-rgs.bicep' = {
  name: 'resources-groups-deployment'
  params: {
    RGnameAIB: RGnameAIB
    RGnameAVDimage: RGnameAVDimage 
    location: location
    tags: tags
  }
}

// Create UAMI, custom role and assign them (on image resource group)
module role 'aib-role.bicep' = {
  scope: resourceGroup(RGnameAVDimage)
  name: 'ID-and-role-deployment'
  params: {
    location: location
    baseTime: baseTime
  }
}

// Assign UAMI and costum role on temporary resource group
module temprole 'aib-roletemp.bicep' = {
  scope: resourceGroup(RGnameAIB)
  name: 'Temporary-AIB-deployment'
  params: {
    subscriptionID: subscriptionID
    RGnameAIB: RGnameAIB
    RGnameAVDimage:RGnameAVDimage
    idName: role.outputs.idName
    baseTime: baseTime
  }
}

// Create Azure Compute Gallery, image and build the image
module image 'aib-image.bicep' = {
  scope: resourceGroup(RGnameAVDimage)
  name: 'image-deployment'
  params: {
    location: location
    RGnameAIB: RGnameAIB
    galleryName: galleryName
    subscriptionID: subscriptionID
    galleryImageName: galleryImageName
    RGnameAVDimage: RGnameAVDimage
    azureImageBuilderName: azureImageBuilderName
    runOutputName: runOutputName
    idNameid: role.outputs.idNameID
  }
}
```
You see that there are four modules invoked by the file. With the `tags` and `RGnameAIB` parameter we can conform to the Azure Policy that defines the name of the resource group and the mandatory tags.

### Creating resource groups

The creation of the resource groups is done in the `aib-rgs.bicep` file.

```bicep
targetScope = 'subscription'

param location string
// Placing the tags on the resource groups
param tags object 
// Naming the resource groups
param RGnameAVDimage string
param RGnameAIB string

resource RGAVDimage 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name: RGnameAVDimage
  location: location
  tags: tags
}

resource RGAVDimagebuild 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name: RGnameAIB
  location: location
  tags: tags
}
```

### Creating the Manages Identity and Custom Role


We use the same managed identity to access the two resource geroups and build the image.
![Managed Identity flow](/BICEP-automate-AIB/flow.jpg)
Within the `aib-role.bicep` file we create the User Assigned Managed Identity and the Custom Role.

```bicep
param location string
param baseTime string

var idAIBName = 'AIB${baseTime}'
var roleDefName = 'Azure Image Builder Def ${baseTime}'


// Create a user assigned identity
resource aibId 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-07-31-preview' = {
  name: idAIBName
  location: location
}

output idName string = aibId.name
output idNameID string = aibId.id

// Create a custom role
resource roleDef 'Microsoft.Authorization/roleDefinitions@2022-05-01-preview' = {
  name: guid(resourceGroup().id, 'bicep')
  properties: {
    roleName: roleDefName
    description: 'Image Builder access to create resources for the image build'
    type: 'customRole'
    permissions: [
      {
        actions: [
          'Microsoft.Compute/galleries/read'
          'Microsoft.Compute/galleries/images/read'
          'Microsoft.Compute/galleries/images/versions/read'
          'Microsoft.Compute/galleries/images/versions/write'
          'Microsoft.Compute/images/write'
          'Microsoft.Compute/images/read'
          'Microsoft.Compute/images/delete'

          'Microsoft.VirtualMachineImages/imageTemplates/run/action'
        ]
        notActions: []
      }
    ]
    assignableScopes: [
      resourceGroup().id
    ]
  }
}

resource roleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, aibId.name)
  properties: {
    principalId: aibId.properties.principalId
    roleDefinitionId: roleDef.id
    principalType: 'ServicePrincipal'
  }
}
```

The name value of the three resources created here all have variables in them. I prefer this to keep track of my image deployment.
The `'Microsoft.VirtualMachineImages/imageTemplates/run/action'` action is not required for the actions done in the `RG-image` resource group. But is is required for the build of the image in the `RG-temp` and I want to use the same Managed Identity for this.

### Set the Custom Role on the build resource group
In the `aib-roletemp.bicep` file we assign the combination of the Managed Identity and Custom Role to the build resource group (`RG-temp`).

```bicep
param baseTime string
param subscriptionID string
param RGnameAIB string
param idName string
param RGnameAVDimage string

resource aibId 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-07-31-preview' existing = {
  name: idName
  scope: resourceGroup(RGnameAVDimage)
}

resource roleAssignmentAIBrg 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, aibId.name, baseTime)
  properties: {
    principalId: aibId.properties.principalId
    roleDefinitionId: '/subscriptions/${subscriptionID}/providers/Microsoft.Authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c'
    principalType: 'ServicePrincipal'
    scope: '/subscriptions/${subscriptionID}/resourcegroups/${RGnameAIB}'
  }
}
```

### Create and build the image

```bicep
param location string
param subscriptionID string
param galleryName string
param azureImageBuilderName string
param galleryImageName string
param runOutputName string
param RGnameAIB string
param RGnameAVDimage string
param idNameid string


resource acg 'Microsoft.Compute/galleries@2022-08-03' = {
  name: galleryName
  location: location
  properties: {
    description: 'mygallery'
  }
}

resource ign 'Microsoft.Compute/galleries/images@2022-08-03' = {
  name: galleryImageName
  location: location
  parent: acg
  properties: {
    identifier: {
      offer: 'windows-11'
      publisher: 'microsoftwindowsdesktop'
      sku: 'win11-23h2-avd'
    }
    osState: 'Generalized' 
    osType: 'Windows'
    hyperVGeneration: 'V2'
  }
}

resource azureImageBuilder 'Microsoft.VirtualMachineImages/imageTemplates@2022-02-14' = {
  name: azureImageBuilderName
  location: location
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: json('{"${idNameid}":{}')
  }
  properties: {
    buildTimeoutInMinutes: 60
    distribute: [
      {
        type: 'SharedImage'
        galleryImageId: ign.id
        runOutputName: runOutputName
        replicationRegions: [
          location
        ]
      }
    ]
    source: {
      type: 'PlatformImage'
      publisher: 'microsoftwindowsdesktop'
      offer:'windows-11'
      sku: 'win11-23h2-avd'
      version: 'latest'
    }
    stagingResourceGroup: '/subscriptions/${subscriptionID}/resourceGroups/${RGnameAIB}'
    vmProfile: {
      vmSize: 'Standard_D2s_v3'
      osDiskSizeGB: 127
    }
    customize: [
      {
        type: 'PowerShell'
        name: 'GetAZCopy'
        inline: [
          'New-Item -Type Directory -Path c:\\ -Name temp'
          'invoke-webrequest -uri https://aka.ms/downloadazcopy-v10-windows -OutFile c:\\temp\\azcopy.zip'
          'Expand-Archive c:\\temp\\azcopy.zip c:\\temp'
          'copy-item C:\\temp\\azcopy_windows_amd64_*\\azcopy.exe\\ -Destination c:\\temp'
        ]
      }
    ]
  }
}

resource buildimage 'Microsoft.Resources/deploymentScripts@2023-08-01' = {
  name: 'buildimage'
  location: location
  kind: 'AzureCLI'
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: json('{"${idNameid}":{}')
  }
  properties: {
    azCliVersion: '2.52.0' 
    retentionInterval: 'P1D'
    environmentVariables: [
      {
        name: 'azureImageBuilderName'
        value: azureImageBuilderName
      }
      {
        name: 'RGnameAVDimage'
        value: RGnameAVDimage
      }
    ]
    scriptContent: '''
      az login --identity
      az image builder run -n $azureImageBuilderName -g $RGnameAVDimage --no-wait
    '''
  }
  dependsOn: [
    azureImageBuilder
  ]
}
```
The ectual build of the image is done in Azure CLI. I hardcoded most of the image because it makes it easier to read. In production I would make more use of parameters. \
To show how customizations are done I added AZCopy to the image. The storage account which was created automatically contains the log file of the build. This is very usefull when you add more customizations than AZcopy. <br>
Feel free to mess around with various customizations. That makes it all the more fun!