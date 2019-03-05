# AzureVM

## Universal Azure VM Deployment Collection

This project is designed to host a series of linked deployment templates that can be referenced manually or by automated provisioning systems to produce a broad range of commonly requested Azure IaaS VM provision options such as:

* Marketplace Image Selection (Windows and Linux/appliance)
* Data Disk additions
* AV Set inclusion
* Tag data
* Recovery Services Vault and policy selection
* Load Balancer membership
* Private IP - static or dynamic
* Public IP - static or dynamic
* NSG association
* Accelerated Networking option
* Hybrid Use Benefit option
* VM Extensions
    * Domain Join
    * BGInfo
    * OMS Agent
    * Auto Shutdown
    * Complex DSC
    * Generic DSC

**Pre-requisites**
* It is assumed a pre-existing VNet and subnet await your VM deployment
* A boot diagnostics storage account to reference

**NOTE:** All disks configured by the template will be Managed Disks only - no support for Unmanaged disks in Storage Accounts.

All you will need to get started is to review the sample parameter file and provide your own relevant inputs. It is recommended to reference Key Vault secrets for all protected settings like passwords, storage account keys or SAS tokens if using the DSC extension options.

## Deployment

The azureDeploy.json file can be copied or uploaded into a new Template Deployment resource in the Azure Portal along with your accompanying parameter file and it will reference the publicly published linked templates. Alternatively the following lines can be used to deploy the master template as is using PowerShell (with the Az or AzureRm modele installed) from its current folder location:

```PowerShell
# Deploy a VM from this original published template in GitHub
New-AzDeployment -Location <Azure location> `
                      -ResourceGroupName <Name of your Resource Group> `
                      -Name <A descriptive name for this deployment> `
                      -TemplateUri https://raw.githubusercontent.com/jvta/AzureVM/master/azuredeploy.json `
                      -TemplateParameterFile <path to your local azuredeploy.network.parameters.json file> `
                      -Verbose
# Substitute New-AzDeployment with New-AzureRmDeployment if you are running the old AzureRm PowerShell module
# instead of the Az module
```

If you clone or download this solution locally to customise it further you will still need to provide a publicly accessible path to your linked template files e.g. a public GitHub repo of your own. Be sure to also update the templateLink variable in your copy of the azuredeploy.json master template to reflect your updated template collection location.

```PowerShell
# Deploy a VM from a local copy of the azuredeploy.json parent template
#  Note: you will still need to publish your edited linked templates to a public location
#  and reference this location in your azuredeploy.json file variables section
New-AzDeployment -Location <Azure location> `
                      -ResourceGroupName <Name of your Resource Group> `
                      -Name <A descriptive name for this deployment> `
                      -TemplateFile <path to your local copy of the azuredeploy.json file> `
                      -TemplateParameterFile <path to your local azuredeploy.parameters.json file> `
                      -Verbose
```

Additionally, if you clone the repo in GitHub or a similar pulbic repository you can provide a branch parameter (strBranch) to assist in selecting development or master branches during testing. Leave parameter set as "master" if unsure.

## Parameter File
Should you clone this solution it includes a .gitignore file to prevent unintended upload of any *.param.json files into your public repository

Create a locally saved parameter file for each VM to be built then store it securely as appropriate within your organisation.

It is recommended to abstract all sensitive data out of parameter files and store as Secrets in Azure Key Vault. For example, the provisionVM templates (either with or without data disks) create a VM that requires two sensitive inputs by default: adminUsername and adminPasswordOrKey. These should be stored in a Key Vault as distinct Secrets and referenced within the parameter file as follows:

```JSON
{
    "adminUsername": {
        "reference": {
            "keyVault": {
                "id": "/subscriptions/********-****-****-****-************/resourceGroups/testrg/providers/Microsoft.KeyVault/vaults/testKeyVault"
            },
            "secretName": "vmWindowsLocalAdminUsername"
        }
    },
    "adminPasswordOrKey": {
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
* strVmName
* strAdminUserName
* sstrAdminPasswordOrKey
* strVirtualNetworkName
* strVirtualNetworkRGName
* strSubnetName
* strBootDiagnosticsStorage

Other key parameters, even VM size and OS selection do have defaults supplied within the template that will produce a vaiable VM, however you should always review all parameters for relevance to your current build. All remaining parameters can be considered as optional.

### Image Selection

The template supplies three default options for image selection, and a custom option. Set strPlatform to either 'Windows' or 'Linux' to pass parameters specific to each platform, then set strImage as one of the following options:
* WindowsServer (default - currently pub:MicrosoftWindowsServer; offer:WindowsServer; sku:2016-Datacenter; version: latest)
* Windows10     (currently pub:MicrosoftWindowsDesktop; offer:Windows-10; sku:rs5-pro; version: latest)
* Linux         (currently pub:RedHat; offer:RHEL; sku:7.4; version: latest)
* Custom        (provide required strPub, strOffer, strSku and strVersion separately)

If your requirement does not match the default options provide the required Publisher, Offer, SKU and Version details to the following parameters:
* strPub
* strOffer
* strSku
* strVersion

### Data Disks

Data Disks must be entered in an array format within the parameter file as per the example below for two different disk types:

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

**Note** The array object must not be passed in as empty or null as the script logic cannot compute the length of an empty array and will result in an error. For this reason ALWAYS leave at least some default data for a single disk in the array object as below:

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

If you do not need any data disks simply set the Boolean parameter boolAddDataDisks to false. The dummy data disk inputs will not result in a data disk provision if the Boolean is set to false.

### Tags
6 tags are provided in the parameter file which are converted to an array in the azuredeploy file variables. Ideally tags should be provided or not based on inputs to the parameter file and passed accordingly to the linked templates, however it was not feasible to provide an array object as input as with objDataDisks as above and convert this into an array object the tags field could consume. Subsequently, either use the individual named tags provided here or clone your own copy of this project to provide more suitable tags for your environment.

Included tags are:
* CostCentre     (provide your preferred cost centre or department code for tracking or recharge purposes)
* Application    (a descriptive name of the application or purpose of this resource collection e.g. AD)
* ProvisionedBy  (the person or account used to provision this resource)
* ProvisionDate  (the date or time of provision - in your preferred format)
* ITOwner        (who is responsible for the resource)
* Environment    (e.g. Test, Prod)

## VM Extensions
The VM provision templates also permit optional VM extensions such as BGInfo, AD domain join or a PowerShell DSC extension. Simply set Boolean parameters e.g. boolUseDomainJoinExtension or boolUseGenericDSCExtension to 'false' if you do not wish to use these. Alternatively, review the parameters required by each extension below and edit your copy of the templates to suit.

### Domain Join
The domain join extension requires you specify:
* strDomainToJoin (specify FQDN of AD domain e.g. company.local)
* strOuPath (Canonical format e.g. OU=Servers,DC=domain,DC=local)
* strDomainJoinAccount
* sstrDomainJoinAccountPassword

Server will restart upon successful domain join and has dependency on VM provision completing

### BGInfo
Simpy set the following parameter to true if you require this extension:
* boolUseBGInfoExtension

### OMS Agent
To attach the OMS (Log Analytics) agent to your VM supply the following parameters:
* boolUseOMSExtension - set to 'true' to use this extension
* strOmsWorkspaceId (your unique workspace ID)
* sstrOmsWorkspaceKey (store this in KeyVault if possible)
* strOmsProxyUri (optional if you require it, leave as empty e.g. "" if not)

### Auto Shutdown
To apply the Auto Shutdown extension to your VM supply the following parameters:
* boolUseAutoShutdownExtension - set to 'true' to use this extension
* strShutdownStart (Supply at time in 24 hour clock notation e.g. 1900 for 7PM. Timezone will be taken from the VM timezone parameter)

### PowerShell DSC
To use either of the PowerShell DSC extensions you will need to write your own DSC config file and publish it to a storage account blob or other publicly accessible URL location then adjust any settings here to provide the appropriate inputs - the parameters for the Customised DSC extension may vary from what you create. Specific DSC content creation is not covered here.

For further reference start with:
* https://docs.microsoft.com/en-us/powershell/dsc/configurations/configurations

For DSC configuration publication to Azure blob storage:
* https://docs.microsoft.com/en-us/powershell/module/azurerm.compute/publish-azurermvmdscconfiguration?view=azurermps-6.13.0

**Customised DSC**

The Customised PowerShell DSC extension requires you specify several configuration related settings common to all configurations, as well as protected settings that will be specific to your particular PowerShell DSC configuration file and what it does.

Common parameters:
* boolUseCustomisedDSCExtension - set to 'true' to use this extension
* strDscExtensionUpdateTagVersion (Default '1'. Increment and redeploy to force an updated configuration)
* strArtifactsLocation (e.g. storage account where your DSC blob is stored e.g. "https://mydscacc.blob.core.windows.net" or a perhaps another GitHub location e.g. "https://raw.githubusercontent.com/mygithub")
* strDscFunction (your name for a DSC "configuration" block to apply e.g. 'StandardAzureServer')
* strDscScript (your DSC configuration script name. Default is configuration.ps1)
* strDscExtensionArchiveFolder (a folder or container name hosting the configuration file. Will otherwise use the storage container default named 'windows-powershell-dsc')
* strDscExtensionArchiveFileName (your archive file name. Will otherwise use a default of 'configuration.ps1.zip')

Default variables:
* strExtensionApiVersion (uses a default of '2015-06-15' set in azuredeploy.json)

Example-specific parameters:
* sstrStorageUserName (e.g. provide a storage account name for use with Azure Files share to store SOE installation files)
* sstrStoragePassword (primary or secondary storage account password for above - SAS tokens don't work for this)
* strLocalAccountUserName (for creating an additional local account)
* sstrLocalAccountUserPassword (password for above)
* sstrArtifactsLocationSasToken (SAS token for accessing configuration.ps1.zip file blob)

**Generic DSC**

The Generic DSC extension contains only details for accessing a PowerShell DSC configuration file and will pass no additional parameters.

Common parameters:
* boolUseGenericDSCExtension - set to 'true' to use this extension
* strDscExtensionUpdateTagVersion (Default '1'. Increment and redeploy to force an updated configuration)
* strArtifactsLocation (e.g. storage account where your DSC blob is stored e.g. "https://mydscacc.blob.core.windows.net" or perhaps another GitHub location e.g. "https://raw.githubusercontent.com/mygithub")
* strDscFunction (your name for a DSC "configuration" block to apply e.g. 'StandardAzureServer')
* strDscScript (your DSC configuration script name. Default is configuration.ps1)
* strDscExtensionArchiveFolder (a folder or container name hosting the configuration file. Will otherwise use the storage container default named 'windows-powershell-dsc')
* strDscExtensionArchiveFileName (your archive file name. Will otherwise use a default of 'configuration.ps1.zip')

Default variables:
* strExtensionApiVersion (uses a default of '2015-06-15' set in azuredeploy.json)

Optional parameter:
* sstrArtifactsLocationSasToken (an Azure SAS token key to access a storage blob if this is where your DSC configuration file is stored. Leave empty to omit e.g. "")
