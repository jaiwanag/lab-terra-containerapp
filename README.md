# Prerequisites:

* Personal Azure Subscription
* Personal GitHub account

# Install Software:

### Install VSCode
In PowerShell past the following code:
`Winget install -e --id Microsoft.VisualStudioCode --source winget`

### Install Az CLI
In PowerShell past the following code:
`winget install -e --id Microsoft.AzureCLI --source winget`

### Install Terraform and Configure
In PowerShell past the following code:
```powershell
$url = "https://releases.hashicorp.com/terraform/1.3.9/terraform_1.3.9_windows_386.zip"
$outputPath = "$env:TEMP\terraform.zip"
$destinationPath = "C:\admin\terraform"
 
# Create the Terraform folder if it doesn't already exist
if (-not (Test-Path $destinationPath)) {
    New-Item -ItemType Directory -Path $destinationPath | Out-Null
}
 
# Add the Terraform folder to the Windows PATH if it's not already there
if (-not ($env:Path -split ";" | Select-String -SimpleMatch $destinationPath)) {
    $env:Path += ";$destinationPath"
    [Environment]::SetEnvironmentVariable("Path", $env:Path, [System.EnvironmentVariableTarget]::Machine)
}

# Download and extract Terraform
Invoke-WebRequest -Uri $url -OutFile $outputPath
Expand-Archive -Path $outputPath -DestinationPath $destinationPath
Remove-Item $outputPath
 
# Display a message to confirm that Terraform is installed and in the PATH
Write-Host "Terraform has been installed to $destinationPath and added to the Windows PATH."
```

### Install Git for Windows
In PowerShell paste the following:
`winget install -e --id Git.Git --source winget`

### Configure Git
```bash
# add user name
git config --global user.name "Your Name"

# add email address
git config --global user.email "email@google.com"

# verify
git config –global –list
```

### Configure VS Code
1. Enable Git  
	a. Open Visual Studio Code  
 	b. Go to File menu, then Preferences, and then Settings  
	c. In the search box, type “Git: Enabled”, next tick the Check Box to Enable Git  
2. Set Path to Git  
  	a. Open Visual Studio Code 
  	b. Go to File menu, then Preferences, and then Settings  
  	c. In the search box, type "git.path" and enter the following line in the "Settings" editor and save:  
	d. "git.path": "C:\\Program Files\\Git\\bin\\git.exe"  
3. Add Extensions    
  * Terraform (HashiCorp    
  * Azure Terraform (Microsoft)  
  * Gitignore (CodeZombie)   
  * GitHub Pull Request and Issues   

### Setup Terraform Project

Container App - https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/container_app
 
### Create Terraform Project  
1. Create a folder called **_lab-terra-containerapp_** in **_c:\\admin\\labs_** 
2. Create the following files in the folder:   
  * gitignore (terraform)  
  * main.tf  (Paste code from Terraform site https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/container_app)

Replace Resource Names:  
 * Resource Group:  lab-rg  
 * Log Analytics Workspace:  lab-law   
 * Container App Environment:  lab-cae   
 * Container App:  lab-ca   

3. Create a **_providers.tf_** file in **_c:\\admin\\labs\\lab-terra-containerapp_**
4. Add the following code to the file
```yaml
terraform {
  required_providers {
    azurerm = {
      source = "hashicorp/azurerm"
      version = "3.43.0"
    }
  }
}
 
provider "azurerm" {
  features {}
}
```

### Add to Source Control
1. Commit to Git
  a. Visual Studio Code > Source Code > Initialize Repository
  b. Type commit message: first commit 
2. Commit to GitHub
  a. Visual Studio Code > Source Code > Publish Branch

### Terraform initialize, plan, and deploy
1. Initialize directory, pull down providers
`terraform init`
2. Format code per HCL canonical standard
`terraform fmt`
3. Preview Terraform deployment
`terraform plan`
4. Apply the changes with 256 simultaneous resource operations
`terraform apply -auto-approve -parallelism=256`

In the Azure portal validate that the resources were created

### Login to the Azure portal and verify that the following resources are created:

* Resource Group:  **_lab-rg_**
* Log Analytics Workspace:  **_lab-law_**
* Container App Environment:  **_lab-cae_**
* Container App:  **_lab-ca_**

### Test the Application Url for the Container App
* Azure Portal > Container App > **_lab-ca_** > overview > Application Url

### View Terraform state
* Show the state file in a human-readable format.
`Terraform show`
* Lists out all the resources that are tracked in the current state file.
`Terraform state list`
* Show the specified resource in the state file.
`Terraform state show <resourcename> `

### Add output.tf
1. Create a **_output.tf_**  file in **_c:\\admin\\labs\\lab-terra-containerapp_**
2. Add the following code to the file
```yaml
output "resource_group_name" {
  description = "Resource group name where the container app is deployed to"
  value       = azurerm_resouse_group.example.name
}
```
3. Commit to source control

### Add variables.tf
1. Create a **_variables.tf_**  file in **_c:\\admin\\labs\\lab-terra-containerapps_**
2. Add the following code to the file
```yaml
variable "image" {
  description = "Container image to be deployed to the container app"
  type        = string
  default     = "mcr.microsoft.com/azuredocs/containerapps-helloworld:latest"
}
```

3. Added the variable for the image in the main.tf file
```yaml
template {
  container {
    name   = "examplecontainerapp"
    image  = var.image
    cpu    = 0.25
    memory = "0.5Gi"
  }
}
```

4.  Commit to source control

### Add terraform.tfvars
1. Create a **_terraform.tfvars_** file in **_c:\\admin\\labs\\lab-terra-containerapp_**
2. Add the following code to the file
`Image = "nginx:latest"`
3. Commit to source control

## Add a Remote Backend for the Terraform State

#### Create Azure resources
1. In the Azure Portal  
  * Create resource group (infra-rg)  
  *  Create storage account (infrasa)   
2. Create Azure AD service principal
```powershell
az ad sp create-for-rbac --name "s-DevOPS-lab" --role contributor `
  --scopes /subscriptions/{subscription-id}/resourceGroups/infra-rg `
  --sdk-auth
# NOTE:  to get the subscription id:   
az account show 
```

Save the output
```json
{
    "clientId": "<GUID>",
    "clientSecret": "<GUID>",
    "subscriptionId": "<GUID>",
    "tenantId": "<GUID>",
}
```

3. Update the terraform providers.tf file to apply the remote backend
_(Note:  it's not best practice to add the client_id, subscription_id, and tenant_id in the providers.tf file)_
```yaml
# Authenticating using Azure AD Authentication of a service principal

backend "azurerm" {
  storage_account_name = "vseinfrasa"
  container_name = "tfstate"
  key = "lab-terra-containerapp.tfstate"
  use_azuread_auth = true
  client_id = "<GUID>" 
  subscription_id = "<GUID>" 
  tenant_id = "<GUID>" 
}
```

4.	Commit to source control

## Use GitHub Actions to deploy Terraform

1. Create GitHub secret (use information from the saved service principal output)
2. Create GitHub workflow
## Home work:

Add tags for each resource:
* Resource group
* Log Analytics
* Container App
* Container App Environment

Tag values
* Environment = VSE
* IaC = terraform
* Owner = "your name"
* project = lab-terra-containerapp

## Future work:

1. Branch the project
2. Make updates and merge
3. Create a custom docker container and store in GitHub
4. GitHub actions to deploy the container to the Container App

### Quick Configs
PowerShell command to update path in the registry
```powershell
Set-ItemProperty -Path 'Registry::HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Session Manager\Environment' `
  -Name PATH `
  -Value  (((Get-ItemProperty -Path 'Registry::HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Session Manager\Environment' -Name PATH).path) + ";c:\admin\terraform" )
```
## References:

Install Terraform on Windows with Bash:  https://learn.microsoft.com/en-us/azure/developer/terraform/get-started-windows-bash?tabs=bash
Use GitHub Actions to connect to Azure:  https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Cwindows#use-the-azure-login-action-with-a-service-principal-secret
