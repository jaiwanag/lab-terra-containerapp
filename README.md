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
```
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
git config -–global -–list
```

### Configure VS Code
1. Enable Git  
   a.  Open Visual Studio Code  
   b.  Go to File menu, then Preferences, and then Settings  
   c.  In the search box, type “Git: Enabled”, next tick the Check Box to Enable Git  
2. Set Path to Git  
   a.  Open Visual Studio Code 
   b.  Go to File menu, then Preferences, and then Settings  
   c.  In the search box, type "git.path" and enter the following line in the "Settings" editor:  
      * git.path: **_`C:\\Program Files\\Git\\bin\\git.exe`_**
3. Add Extensions    
   * Terraform (HashiCorp    
   * Azure Terraform (Microsoft)  
   * Gitignore (CodeZombie)   
   * GitHub Pull Request and Issues   
 
### Create The Terraform Project  
1. Create a folder called **_lab-terra-containerapp_** in **_c:\\admin\\labs_** 
2. Create the following files in the folder:   
   * gitignore
     * In VS Code open the command palette by pressing the **_Ctrl+Shift+P_** keys
     * Type "add gitignore" and choose **_terraform_** and press enter
   * main.tf  (Paste code from Terraform site https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/container_app)
   
example main.tf
```
resource "azurerm_resource_group" "example" {
  name     = "example-resources"
  location = "West Europe"
}

resource "azurerm_log_analytics_workspace" "example" {
  name                = "acctest-01"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}

resource "azurerm_container_app_environment" "example" {
  name                       = "Example-Environment"
  location                   = azurerm_resource_group.example.location
  resource_group_name        = azurerm_resource_group.example.name
  log_analytics_workspace_id = azurerm_log_analytics_workspace.example.id
}
resource "azurerm_container_app" "example" {
  name                         = "example-app"
  container_app_environment_id = azurerm_container_app_environment.example.id
  resource_group_name          = azurerm_resource_group.example.name
  revision_mode                = "Single"

  template {
    container {
      name   = "examplecontainerapp"
      image  = "mcr.microsoft.com/azuredocs/containerapps-helloworld:latest"
      cpu    = 0.25
      memory = "0.5Gi"
    }
  }
}
```

   Update the valuse:
   * Do a find and replace of all instances of **_example_** with **_lab_**  
   * Resource Group:  **_lab-rg_**   
   * Log Analytics Workspace:  **_lab-law_**     
   * Container App Environment:  **_lab-cae_**     
   * Container App:  **_lab-ca_**    
   
   Add Ingress to the **_azurerm_container_app_** resource, place under the **_revision_mode_** line:
   ```
   ingress {    
     target_port = 80    
     external_enabled = true    
     traffic_weight {      
       percentage = 100    
     }  
   }
   ```

3. Create a **_providers.tf_** file in **_c:\\admin\\labs\\lab-terra-containerapp_**
4. Add the following code to the file
```
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
   a.  Visual Studio Code > Source Code > Initialize Repository  
   b.  Type commit message: first commit  
2. Commit to GitHub  
   a.  Visual Studio Code > Source Code > Publish Branch  

### Login to Azure
1. Verify that you are logged into Azure and using the correct account:  
`az account show`  
2. To Login to Azure type the following:  
`az login`  

### Terraform initialize, plan, and deploy
1. Initialize directory, pull down providers  
`terraform init`
2. Format code per HCL canonical standard  
`terraform fmt`  
3. Preview Terraform deployment  
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

### View Terraform State
* Show the state file in a human-readable format  
`Terraform show`  
* Lists out all the resources that are tracked in the current state file  
`Terraform state list`  
* Show the specified resource in the state file  
`Terraform state show <resourcename>`  

### Add output.tf
1. Create a **_output.tf_**  file in **_c:\\admin\\labs\\lab-terra-containerapp_**
2. Add the following code to the file
```yaml
output "resource_group_name" {
  description = "Resource group name where the container app is deployed to"
  value       = azurerm_resource_group.example.name
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

#### Create Azure resources:
* Create resource group (infra-rg)  
* Create storage account (infrasa)
  * In the storage account, create a container called **_tfstate_**   
* Create Azure AD service principal
  * Grant the service principal **_Storage Blob Data Owner_** permissions to the storage account

1. Past the following code in a PowerShell terminal to create the resources above
```
$myCode = @"
# Set the region and resource group name
`$region = "westus"
`$resourceGroupName = "infra-rg"
`$spName = "s-DevOps-lab"

# Generate a random 3-digit number and append it to the storage account name
`$randomNumber = Get-Random -Minimum 100 -Maximum 1000
`$storageAccountName = "infrasa`$randomNumber"

# Get the subscription ID
`$subscriptionId = az account show --query id -o tsv

# Create the resource group
az group create --name $resourceGroupName --location $region

# Create the storage account in the resource group
az storage account create --name $storageAccountName --resource-group $resourceGroupName --location $region --sku Standard_LRS

# Create a container in the storage account
az storage container create --name ftstate --account-name $storageAccountName

# Create a service principal with contributor access to the subscription
`$sp = az ad sp create-for-rbac --name $spName --role contributor --scopes /subscriptions/$subscriptionId --sdk-auth -o tsv

# Get the service principal's ID
`$appId = az ad sp show --id $(az ad sp list --display-name $spName --query '[].appId' -o tsv) --query appId -o tsv

# Grant the account Storage Blob Data Owner to the storage account
az role assignment create --assignee $appId --role `"Storage Blob Data Owner`" --scope /subscriptions/$subscriptionId/resourceGroups/$resourceGroupName/providers/Microsoft.Storage/storageAccounts/$storageAccountName

# Output the contents of `$sp
`$sp
"@

Invoke-Expression -Command $myCode
```
2. In the PowerShell terminal type:  
`$sp`
3. Save the output then remove the following lines, we'll need this later to configuring the GitHub secret
> **_Note: this file contains sensitive information_**  
```
  "activeDirectoryEndpointUrl": "",
  "resourceManagerEndpointUrl": "",
  "activeDirectoryGraphResourceId": "",
  "sqlManagementEndpointUrl": "",
  "galleryEndpointUrl": "",
  "managementEndpointUrl": "" 
```
Your saved file should look like this:
```json
{
  "clientId": "xxxxxxxxxx",
  "clientSecret": "xxxxxxxxxx",
  "subscriptionId": "xxxxxxxxxx",
  "tenantId": "xxxxxxxxxx",
}
```

#### Configure Terraform to use the Azure storage account as the backend
1. Create a new file named **_lab-terra-containerapp.tfbackend_** in your Terraform working directory
2. Add the following contents to the file, replacing the placeholders with your actual values:
```yaml
client_id = "<clientid>"
subscription_id = "<subscriptionId>"
tenant_id = "<tenantId>"
```
3. Update the terraform **_providers.tf_** file with the **_backend_** configuration. This tells Terraform to use the azurerm backend, and to load the configuration from the lab-terra-containerapp.tfbackend file. Here's an example of what your backend configuration will look like:
```yaml
terraform {
  required_providers {
    azurerm = {
      source = "hashicorp/azurerm"
      version = "3.43.0"
    }
  }
  backend "azurerm" {
    storage_account_name = "infrasa"
    container_name = "tfstate"
    key = "lab-terra-containerapp.tfstate"
    use_azuread_auth = true
    //Note:  confidential values are stored in the .tfbackend file
  }
}
 
provider "azurerm" {
  features {}
}
```
4. Run **_terraform_** **_init_** in your working directory to initialize the backend and download any required plugins  
`terraform init -backend-config=aca-terraform.vse.tfbackend`
5. Commit to source control

## Use GitHub Actions to deploy Terraform

1. Create GitHub secret (use information from the saved service principal output)
2. Create GitHub workflow
## Home work:

### Add tags for each resource:  
* Resource group  
* Log Analytics  
* Container App  
* Container App Environment  

    #### Tag values:
    * Environment = VSE
    * IaC = terraform
    * Owner = "your name"
    * project = lab-terra-containerapp

## Future work:
* Add authentication to the container app
* Branch the project
* Make updates and merge
* Create a custom docker container and store in Azure Container Regestry
* GitHub actions to deploy the container to the Container App

## References:
Use GitHub Actions to connect to Azure:   
https://github.com/MicrosoftDocs/azure-docs/blob/main/articles/machine-learning/how-to-github-actions-machine-learning.md  
