# AzureVM

## Universal Azure VM Deployment Collection

This project is designed to host a series of linked deployment templates that can be referenced manually or by automated provisioning systems to produce a broad range of commonly requested Azure IaaS VM provision options such as:

* Marketplace Image Selection (Windows and Linux/appliance)
* VM Extensions
* Data Disk additions
* AV Set inclusion
* Tag data
* Recovery Services Vault and policy selection
* Load Balancer membership
* Private IP - static or dynamic
* Public IP - static or dynamic
* NSG association
* Accelerated Networking option
* IP Forwarding option

The template assumes a pre-existing VNet awaits your VM deployment, and a boot diagnostics storage account to reference.

You will need to review the sample parameter file and provide your own relevant inputs. It is recommended to reference Key Vault secrets for all protected settings like passwords or storage account keys or SAS tokens if using DSC extension option.

The following script can be used to deploy the master template as is from the current location:

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

You can also copy the azuredeploy.json template into a new Template Deployment resource within the portal and supply parameters manually or by selecting your preconfigured parameter file.

If you clone or download this solution locally to insert your own edits you will still need to provide a publicly accessible path to your files e.g. a public GitHub repo of your own. Be sure to also update the templateLink variable in your copy of the azuredeploy.json master template to reflect your updated template collection location.

```PowerShell
# Deploy a VM from your local copy of the azuredeploy.json master template
# Note: you will still need to publish the linked templates to a public location
New-AzDeployment -Location <Azure location> `
                      -ResourceGroupName <Name of your Resource Group> `
                      -Name <A descriptive name for this deployment> `
                      -TemplateFile <path to your local copy of the azuredeploy.json file> `
                      -TemplateParameterFile <path to your local azuredeploy.parameters.json file> `
                      -Verbose
```

## Parameter Files
The solution includes a .gitignore file to prevent unintended upload of any *.param.json files into your public storage

Create a local parameter file for each VM to be built and store it as appropriate within your organisation. It is recommended to abstract all sensitive data out of parameter files and store as Secrets in Azure Key Vault.

For example, provisionVM.json creates a VM that requires two sensitive inputs by default: adminUsername and adminPasswordOrKey. These should be stored in a Key Vault as distinct Secrets and referenced within a parameter file as follows:

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

## VM Extensions
ProvisionVM.json is also set to optionally permit a domain join or PowerShell DSC extension. Simply set parameters for useDomainJoinExtension or useDSCExtension to "No" if you do not wish to use these. Alternatively, review the parameters required by each extension below and edit your copy of the templates to suit.

### Domain Join
The domain join extension requires you specify:
* domainToJoin (specify FQDN of AD domain e.g. company.local)
* ouPath (Canonical format e.g. OU=Servers,DC=domain,DC=local)
* domainJoinAccount
* domainJoinAccountPassword
Server will restart upon successful domain join and has dependency on VM provision completing

### PowerShell DSC
To use the PowerShell DSC extension you will need to write your own DSC config file and publish it to a storage account blob then adjust any settings here to provide the appropriate inputs - these parameters will vary from what you create. DSC content creation is not covered here.

For further reference start with:
* https://docs.microsoft.com/en-us/powershell/dsc/configurations/configurations

For DSC configuration publication to Azure blob storage:
* https://docs.microsoft.com/en-us/powershell/module/azurerm.compute/publish-azurermvmdscconfiguration?view=azurermps-6.13.0

The PowerShell DSC extension requires you specify several configuration related settings common to all configurations, as well as protected settings that will be specific to your particular PowerShell DSC configuration file and what it does.

Common parameters:
* dscExtensionUpdateTagVersion (Default '1'. Increment and redeploy to force an updated configuration)
* _artifactsLocation (storage account where your DSC blob is stored e.g. "https://mydscacc.blob.core.windows.net")
* dscFunction (your name for a DSC "configuration" block to apply e.g. 'StandardAzureServer')

Default variables:
* DscExtensionArchiveFolder (uses the default of 'windows-powershell-dsc')
* DscExtensionArchiveFileName (uses the default of 'configuration.ps1.zip')
* extensionApiVersion (uses a default of '2015-06-15' set in azuredeploy.json)

Example-specific parameters:
* storageUserName (e.g. provide a storage account name for use with Azure Files share to store SOE installation files)
* storagePassword (storage account password for above - SAS tokens don't work for this)
* localAccountUserName (for creating an additional local account)
* localAccountUserPassword (password for above)
* _artifactsLocationSasToken (SAS token for accessing configuration.ps1.zip file blob)
