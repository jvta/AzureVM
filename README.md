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
    * Complex DSC
    * Generic DSC
    * BGInfo
    * OMS Agent

The template assumes a pre-existing VNet awaits your VM deployment, and a boot diagnostics storage account to reference.

You will need to review the sample parameter file and provide your own relevant inputs. It is recommended to reference Key Vault secrets for all protected settings like passwords or storage account keys or SAS tokens if using DSC extension option.

The azureDeploy.json file can be copied into a new Template Deployment in the portal along with your accompanying parameter file and it will reference the publicly published linked templates. Alternatively the following script can be used to deploy the master template as is using PowerShell from its current folder location:

```PowerShell
# Deploy a VM from this master template in GitHub
New-AzDeployment -Location <Azure location> `
                      -ResourceGroupName <Name of your Resource Group> `
                      -Name <A descriptive name for this deployment> `
                      -TemplateUri https://raw.githubusercontent.com/jvta/AzureVM/master/azuredeploy.json `
                      -TemplateParameterFile <path to your local azuredeploy.network.parameters.json file> `
                      -Verbose
# Substitute New-AzDeployment with New-AzureRmDeployment if you are running the old AzureRm PowerShell module
# instead of the Az module
```

If you clone or download this solution locally to insert your own edits you will still need to provide a publicly accessible path to your files e.g. a public GitHub repo of your own. Be sure to also update the templateLink variable in your copy of the azuredeploy.json master template to reflect your updated template collection location.

```PowerShell
# Deploy a VM from your local copy of the azuredeploy.json master template
#  Note: you will still need to publish your edited linked templates to a public location
#  and reference this location in your azuredeploy.json file
New-AzDeployment -Location <Azure location> `
                      -ResourceGroupName <Name of your Resource Group> `
                      -Name <A descriptive name for this deployment> `
                      -TemplateFile <path to your local copy of the azuredeploy.json file> `
                      -TemplateParameterFile <path to your local azuredeploy.parameters.json file> `
                      -Verbose
```

## Parameter Files
The solution includes a .gitignore file to prevent unintended upload of any *.param.json files into your public storage

Create a local parameter file for each VM to be built and store it as appropriate within your organisation.

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
All VM deployments require the following parameters at a minimum to deploy a valid VM:
* strVmName
* strAdminUserName
* sstrAdminPasswordOrKey
* strVirtualNetworkName
* strVirtualNetworkRGName
* strSubnetName
* strBootDiagnosticsStorage
Other key parameters like VM size and OS selection do have defaults supplied within the template that will produce a vaiable VM, however you should always review all parameters for relevance to your current build. All remaining parameters can be considered as optional.

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
* strOmsProxyUri (optional if you require it, leave as empty - "" if not)

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
* strArtifactsLocation (e.g. storage account where your DSC blob is stored e.g. "https://mydscacc.blob.core.windows.net" or a perhaps another GitHub location e.g. https://"https://raw.githubusercontent.com/mygithub")
* strDscFunction (your name for a DSC "configuration" block to apply e.g. 'StandardAzureServer')
* strDscScript (your DSC configuration script name. Default is configuration.ps1)
* strDscExtensionArchiveFolder (a folder or container name hosting the configuration file. Will otherwise use a storage blob default of 'windows-powershell-dsc')
* strDscExtensionArchiveFileName (your archive file name. Will otherwise use a default of 'configuration.ps1.zip')

Default variables:
* strExtensionApiVersion (uses a default of '2015-06-15' set in azuredeploy.json)

Example-specific parameters:
* sstrStorageUserName (e.g. provide a storage account name for use with Azure Files share to store SOE installation files)
* sstrStoragePassword (storage account password for above - SAS tokens don't work for this)
* strLocalAccountUserName (for creating an additional local account)
* sstrLocalAccountUserPassword (password for above)
* sstrArtifactsLocationSasToken (SAS token for accessing configuration.ps1.zip file blob)

**Generic DSC**
The Generic DSC extension contains only details for accessing a PowerShell DSC configuration file and will pass no additional parameters.

Common parameters:
* boolUseGenericDSCExtension - set to 'true' to use this extension
* strDscExtensionUpdateTagVersion (Default '1'. Increment and redeploy to force an updated configuration)
* strArtifactsLocation (e.g. storage account where your DSC blob is stored e.g. "https://mydscacc.blob.core.windows.net" or a perhaps another GitHub location e.g. https://"https://raw.githubusercontent.com/mygithub")
* strDscFunction (your name for a DSC "configuration" block to apply e.g. 'StandardAzureServer')
* strDscScript (your DSC configuration script name. Default is configuration.ps1)
* strDscExtensionArchiveFolder (a folder or container name hosting the configuration file. Will otherwise use a storage blob default of 'windows-powershell-dsc')
* strDscExtensionArchiveFileName (your archive file name. Will otherwise use a default of 'configuration.ps1.zip')

Default variables:
* strExtensionApiVersion (uses a default of '2015-06-15' set in azuredeploy.json)

Optional parameter:
* sstrArtifactsLocationSasToken (an Azure SAS token key to access a storage blob if this is where your DSC configuration file is stored. Leave empty to omit e.g. "")
