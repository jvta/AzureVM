# AzureVM

## Universal Azure VM Deployment Collection

This project is designed to host a series of linked deployment templates that can be referenced manually or by automated provisioning systems to produce a broad range of commonly requested Azure IaaS VM provision options such as:

* Marketplace Image Selection (Windows and Linux/appliance)
* Data Disk selection from 0 to N disks and properties
* AV Set creation or reference
* Tag data
* Recovery Services Vault and policy selection
* Load Balancer membership
* Private IP - static or dynamic
* Public IP - static or dynamic
* NSG association
* Accelerated Networking option
* Hybrid Use Benefit option
* Boot Diagnostics Storage Account creation or reference
* VM Extensions
    * Domain Join
    * BGInfo
    * OMS Agent
    * Auto Shutdown
    * Anti Malware
    * Network Watcher
    * Azure Automation DSC
    * Complex DSC
    * Generic DSC

### Pre-requisites

The following resources are presumed to already exist prior to your deployment:
* VNet and subnet
* Storage account for boot diagnostics to reference
* Load Balancer - if required
* NSG - if required
* RSV - if required
* AD Domain - if using Domain Join extension
* DSC configurations - if using DSC extension
* Log Analytics Workspace - if using OMS extension
* Key Vault (optional - recommended repository for all sensitive input parameters)

>**NOTE:**
>
>All disks configured by the template will be Managed Disks only - Unmanaged disks are not supported

All you will need to get started is to review the sample parameter file and provide your own relevant inputs. It is recommended to reference Key Vault secrets for all protected settings like local usernames and passwords (or domain usernames/passwords, storage account keys and SAS tokens if using some of the extension options).

## Deployment

The azureDeploy.json file can be copied or uploaded into a new Template Deployment resource in the Azure Portal along with your accompanying parameter file and it will reference the publicly published linked templates. You can even use the Deploy to Azure button below to launch the template deployment now:

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fjvta%2FAzureVM%2Fmaster%2Fazuredeploy.json" target="_blank">
    <img src="http://azuredeploy.net/deploybutton.png"/>
</a>


Alternatively the following lines can be used to deploy the master template as is using PowerShell or [Cloud Shell](https://shell.azure.com/) (with the Az or AzureRm module installed) from its current folder location:

```PowerShell
# Deploy a VM from this original published template in GitHub
New-AzResourceGroupDeployment -ResourceGroupName <Name of your Resource Group> `
                    -Name <A descriptive name for this deployment> `
                    -TemplateUri https://raw.githubusercontent.com/jvta/AzureVM/master/azuredeploy.json `
                    -TemplateParameterFile <path to your local azuredeploy.network.parameters.json file> `
                    -Verbose
# Substitute New-AzDeployment with New-AzureRmDeployment if you are running the old AzureRm PowerShell module
```

If you clone or download this solution locally to customise it further you will still need to provide a publicly accessible path to your linked template files e.g. a public GitHub repo of your own. Be sure to also update the templateLink variable in your copy of the azuredeploy.json master template to reflect your updated template collection location.

```PowerShell
# Deploy a VM from a local copy of the azuredeploy.json parent template
New-AzResourceGroupDeployment -ResourceGroupName <Name of your Resource Group> `
                      -Name <A descriptive name for this deployment> `
                      -TemplateFile <path to your local copy of the azuredeploy.json file> `
                      -TemplateParameterFile <path to your local azuredeploy.parameters.json file> `
                      -Verbose
```

Additionally, if you clone the repo in GitHub or a similar pulbic repository you can provide a branch parameter (**strBranch**) to assist in selecting development or master branches during testing. Leave parameter set as "master" if unsure.

Should you clone this solution it includes a .gitignore file to prevent unintended upload of any *.param.json files into your public repository

## Parameter File

Create a locally saved parameter file for each VM to be built then store it securely as appropriate within your organisation. Ensure you review all parameters and what they do before attempting your first build. Descriptions for each parameter are in the azuredeploy.json file.

### Sensitive Data

It is recommended to abstract all sensitive data out of parameter files and store as Secrets in Azure Key Vault. For example, the provisionVM templates (either with or without data disks) create a VM that requires two sensitive inputs by default: **strAdminUsername** and **sstrAdminPasswordOrKey**. These should be stored in a Key Vault as distinct Secrets and referenced within the parameter file as follows:

```JSON
{
    "strAdminUsername": {
        "reference": {
            "keyVault": {
                "id": "/subscriptions/********-****-****-****-************/resourceGroups/testrg/providers/Microsoft.KeyVault/vaults/testKeyVault"
            },
            "secretName": "vmWindowsLocalAdminUsername"
        }
    },
    "sstrAdminPasswordOrKey": {
        "reference": {
            "keyVault": {
                "id": "/subscriptions/********-****-****-****-************/resourceGroups/testrg/providers/Microsoft.KeyVault/vaults/testKeyVault"
            },
            "secretName": "vmWindowsLocalAdminPassword"
        }
    }
}
```

### Minimum Mandatory Parameter Requirements
All VM deployments require the following parameters at an absolute minimum to deploy a valid VM:
* **strVmName**
* **strAdminUserName**
* **sstrAdminPasswordOrKey**
* **strVirtualNetworkName**
* **strVirtualNetworkRGName**
* **strSubnetName**

Other key parameters, even VM size and OS selection do have defaults supplied within the template that will produce a vaiable VM, however you should always review ALL parameters for relevance to your current build. All remaining parameters can be considered as optional.

Parameters are named with prefixes according to input type required, e.g. str = string; int = integer; bool = Boolean; obj = object. Ensure input matches expected data types. Most parameters should be named descriptively such that it is apparent what they signify. E.g. **boolAvSetRequired** indicates whether to attach the VM to an AV Set, whereas **boolAvSetCreateNew** indicates whether to use a new or existing AV Set. Follow the parameter through calls from parent to child templates if unsure of function or format required.

### Image Selection

The template supplies three default options for image selection, and a custom option. Firstly set **strOsPlatform** to either 'Windows' or 'Linux' to pass parameters specific to each platform, then set **strImage** as one of the following four options:
* WindowsServer (Default. Currently pub=MicrosoftWindowsServer; offer=WindowsServer; sku=2016-Datacenter; version=latest)
* Windows10     (Currently pub=MicrosoftWindowsDesktop; offer=Windows-10; sku=rs5-pro; version=latest)
* Linux         (Currently pub=RedHat; offer=RHEL; sku=7.4; version=latest)
* Custom        (Provide required strPublisher, strOffer, strSku and strVersion separately)

If your requirement does not match the default WindowsServer, Windows10 or Linux options provide the required Publisher, Offer, SKU and Version details to the following parameters:
* **strPublisher**
* **strOffer**
* **strSku**
* **strVersion**

### Networking

In addition to supplying Subnet Name, Virtual Network Name and associated VNet Resource Group name, also set Booleans for whether Public IP, NSG, Load Balancer or Accelerated Networking are required. Provide details as required for NSG or ALB names, their Resource Groups (even if it is the same as the RG for your VM) and Backend Pool for ALB. Set whether IP addresses should be Static or Dynamic. You can also select SKU type for the Public IP.

### Other Compute Options

In addition to VM Name and Size you can also set:
* OS disk parameters for size in GB and storage type
* Availability Set requirement, AV Set name and whether to create new or reference existing
* The Timezone to set on a Windows VM (defaults to AUS Eastern Standard Time)
* Whether to enable Hybrid Use Benefit licencing for Windows VM
* Whether to use Boot Diagnostics; to create a new or use existing storage account; and the account type to create if required
* Whether to attach the VM to a Recovery Services Vault; its name, Resource Group and Backup Policy to use

### Data Disks

Data Disks must be entered in an array format within the parameter file as per the example below for two disks of different types:

```JSON
"objDataDisks": {
    "value": {
        "disks": [
            {
                "Cache": "None",
                "Size": 1024,
                "StorageAccountType": "Standard_LRS"
            },
                        {
                "Cache": "ReadWrite",
                "Size": 128,
                "StorageAccountType": "Premium_LRS"
            }
        ]
    }
}
```

>**NOTE:**
>
>The array object must not be passed in as empty or null as the script logic cannot compute the length of an empty array and will result in an error. For this reason ALWAYS leave at least some default data for a single disk in the array object as below:

```JSON
"objDataDisks": {
    "value": {
        "disks": [
            {
                "Cache": "None",
                "Size": 1024,
                "StorageAccountType": "Standard_LRS"
            }
        ]
    }
}
```

If you do not need any data disks simply set the Boolean parameter **boolAddDataDisks** to false. The dummy data disk inputs will not result in a data disk provision providing  the Boolean is set to false. You can also safely just delete all of **objDataDisks** from your parameter file as some dummy data will also be provided by default through the azuredeploy.json file thus avoiding this error.

### Tags
Provide your preferred tags to the strTags parameter as a string using backslash characters to escape each double quote character as in example below:
 ```JSON
"strTags": {
    "value" : "{\"Application\":\"Test\",\"CostCentre\":\"MyDept\",\"Environment\":\"Test\"}"
}
```
These tags will be appended to each resource created.

## VM Extensions
The VM provision templates also permit optional VM extensions such as BGInfo, AD domain join or a PowerShell DSC extension. Simply set Boolean parameters e.g. **boolUseDomainJoinExtension** or **boolUseGenericDSCExtension** to 'false' if you do not wish to use these. Alternatively, review the parameters required by each extension below and edit your copy of the templates to suit.

### Domain Join
The domain join extension requires you specify:
* **boolUseDomainJoinExtension** - set to 'true' to use this extension
* **strDomainToJoin** (specify FQDN of AD domain e.g. company.local)
* **strOuPath** (Canonical format e.g. OU=Servers,DC=domain,DC=local)
* **strDomainJoinAccount**
* **sstrDomainJoinAccountPassword**

Server will restart upon successful domain join and has dependency on VM provision completing

### BGInfo
Simply set the following parameter to true if you require this extension:
* **boolUseBGInfoExtension**

### OMS Agent
To attach the OMS (Log Analytics) agent to your VM supply the following parameters:
* **boolUseOMSExtension** - set to 'true' to use this extension
* **strOmsWorkspaceId** (your unique workspace ID)
* **sstrOmsWorkspaceKey** (store this in KeyVault if possible)
* **strOmsProxyUri** (optional if you require it)

### Auto Shutdown
To apply the Auto Shutdown extension to your VM supply the following parameters:
* **boolUseAutoShutdownExtension** - set to 'true' to use this extension
* **strShutdownStart** (Supply at time in 24 hour clock notation e.g. 1900 for 7PM. Timezone will be taken from the VM timezone parameter)

### Anti Malware
To attach the Anti Malware agent to your VM supply the following parameters:
* **boolUseAntiMalwareExtension** - set to 'true' to use this extension
* **strAntiMalwarePathExclusions** (The file or folder paths to be excluded from Anti Malware scans. Use a semi-colon separated list. Wildcards not advised)
* **strAntiMalwareFileExtensionExclusions** (The file extensions to be excluded from Anti Malware scans. Use a semi-colon separated list. Wildcards not advised)
* **strAntiMalwareProcessExclusions** (The processes to be excluded from Anti Malware scans. Use a semi-colon separated list. Wildcards not supported)
* **strAntiMalwareRealtimeProtectionEnabled** (Whether this server should use Anti Malware Real Time Protection)
* **strAntiMalwareScheduledScanEnabled** (Whether this server should conduct Anti Malware Scans)
* **strAntiMalwareScanType** (The type of Anti Malware Scan to perform - choose from Quick or Full)
* **intAntiMalwareScanDay** (The day on which to run the Anti Malware Scheduled Scan. Sunday=1; Monday=2; Tuesday=3; Wednesday=4; Thursday=5; Friday=6; Saturday=7. Default is Sunday)
* **intAntiMalwareScanTime** (The time at which the Anti Malware Scheduled Scan should be run in minutes from midnight. E.g. 60 = 1AM; 360 = 6AM)

### Network Watcher
To attach the Network Watcher extension to your VM supply the following parameter:
* **boolUseNetworkWatcherExtension** - set to 'true' to use this extension

### PowerShell DSC
To use any of the PowerShell DSC extensions you will need to write your own DSC config file and publish it to an Azure Automation Account, storage account blob or other publicly accessible URL location then adjust any settings here to provide the appropriate inputs - the parameters for the Customised DSC extension may vary from what you create. Specific DSC content creation is not covered here.

For further reference start with:
* https://docs.microsoft.com/en-us/powershell/dsc/configurations/configurations

For DSC configuration publication to Azure blob storage:
* https://docs.microsoft.com/en-us/powershell/module/azurerm.compute/publish-azurermvmdscconfiguration?view=azurermps-6.13.0

**Azure Automation State Configuration**

The Azure Automation State Configuration extension references an Azure Automation account and registers the server as a new managed node in State Configuration. Obtain the Primary Access Key and URL for the Automation Account and supply these parameters to the extension.

Common parameters:
* **boolUseAzureAutomationDSCExtension** - set to 'true' to use this extension
* **automationAccountRegistrationKey** - (obtain from Automation Account, Account settings. Will be long complex key)
* **automationAccountRegistrationUrl** - (obtain from Automation Account, Account settings. e.g. https://AE-agentservice-prod-1.azure-automation.net/accounts/<random-GUID>)
* **strNodeConfiguration** (your name for the Azure Automation Node Configuration to apply e.g. 'StandardAzureServer.localhost'. NOTE: it is the name of the Node Configuration, not the Configuration. The name will always be the Configuration followed by a period . and either localhost or a specific computer name)

**Customised DSC**

The Customised PowerShell DSC extension requires you specify several configuration related settings common to all configurations, as well as protected settings that will be specific to your particular PowerShell DSC configuration file and what it does.

Common parameters:
* **boolUseCustomisedDSCExtension** - set to 'true' to use this extension
* **strDscExtensionUpdateTagVersion** (Default '1'. Increment and redeploy to force an updated configuration)
* **strArtifactsLocation** (e.g. storage account where your DSC blob is stored e.g. "https://mydscacc.blob.core.windows.net" or a perhaps another GitHub location e.g. "https://raw.githubusercontent.com/mygithub")
* **strDscFunction** (your name for a DSC "configuration" block to apply e.g. 'StandardAzureServer')
* **strDscScript** (your DSC configuration script name. Default is configuration.ps1)
* **strDscExtensionArchiveFolder** (a folder or container name hosting the configuration file. Will otherwise use the storage container default named 'windows-powershell-dsc')
* **strDscExtensionArchiveFileName** (your archive file name. Will otherwise use a default of 'configuration.ps1.zip')

Default variables:
* **strExtensionApiVersion** (uses a default of '2015-06-15' set in azuredeploy.json)

Example-specific parameters:
* **sstrStorageUserName** (e.g. provide a storage account name for use with Azure Files share to store SOE installation files)
* **sstrStoragePassword** (primary or secondary storage account password for above - SAS tokens don't work for this)
* **strLocalAccountUserName** (for creating an additional local account)
* **sstrLocalAccountUserPassword** (password for above)
* **sstrArtifactsLocationSasToken** (SAS token for accessing configuration.ps1.zip file blob)

**Generic DSC**

The Generic DSC extension contains only details for accessing a PowerShell DSC configuration file and will pass no additional parameters.

Common parameters:
* **boolUseGenericDSCExtension** - set to 'true' to use this extension
* **strDscExtensionUpdateTagVersion** (Default '1'. Increment and redeploy to force an updated configuration)
* **strArtifactsLocation** (e.g. storage account where your DSC blob is stored e.g. "https://mydscacc.blob.core.windows.net" or perhaps another GitHub location e.g. "https://raw.githubusercontent.com/mygithub")
* **strDscFunction** (your name for a DSC "configuration" block to apply e.g. 'StandardAzureServer')
* **strDscScript** (your DSC configuration script name. Default is configuration.ps1)
* **strDscExtensionArchiveFolder** (a folder or container name hosting the configuration file. Will otherwise use the storage container default named 'windows-powershell-dsc')
* **strDscExtensionArchiveFileName** (your archive file name. Will otherwise use a default of 'configuration.ps1.zip')

Default variables:
* **strExtensionApiVersion** (uses a default of '2015-06-15' set in azuredeploy.json)

Optional parameter:
* **sstrArtifactsLocationSasToken** (an Azure SAS token key to access a storage blob if this is where your DSC configuration file is stored)