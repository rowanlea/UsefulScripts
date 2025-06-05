# Exporting Azure Resources To Terraform
<details>
<summary>Fixing Terraform: a personal note about fixing my install</summary>

Running `terraform --version` showed it was installed, but told me it was out of date, and to update.

The link it gave me to update wasn't very helpful, so I went to winget:
`winget install --id=Hashicorp.Terraform  -e`

Running `terraform --version` gave me the same error as before, and I couldn't see it in my PATH variables.

Running `where.exe terraform` told me where it was located (note: this is for Windows). I had previously installed it with chocolatey.

Chocolatey blocked me from uninstalling it via commands, so I just went to the folder "where.exe" listed, and deleted the terraform.exe folder.

**Take away:** to avoid this kind of problem I'd highly recommend using just 1 method on your OS to manage all packages. Since I had considerably more outside of Chocolatey I decided to continue using winget. Also I prefer the terminal responses, Chocolatey gets a bit too messy for my liking.
</details>

## Installing aztfexport
The Microsoft Learn page takes you to the artifacts page of the GitHub repository, but honestly it's much easier if you just follow the instructions [inside the repo](https://github.com/Azure/aztfexport) for your operating system.

## Log in via your terminal
You'll also need the Azure CLI installed locally for this.
`az login`

## Show which subscription you're currently using
`az account show --query id --output tsv`

## List all subscriptions you have access to
`az account list --all --output table`

## Switch to the subscription where your resources lie
`az account set --subscription <YourSubscriptionId>`

## Show which resource groups you have access to, to double check
`az group list`

## Export your resource group
`aztfexport resource-group <resource group name>` (note: resource group name is case sensitive), then press "W" to "acquire" the listed resources locally in Terraform format. It says "import", but it means "export".

You can also export an individual resource, by calling `aztfexport resource <resource ID>`. To get the resource ID, go to the resource, then go to Settings->Properties, and it's listed there. Note that the entry includes a whole sub-path including the subscription and resource group location - it needs this whole things as the ID.

## Verifying it worked
`terraform init --upgrade`
This command downloads the Azure provider required to manage your Azure resources. Upgrade ensures the provider plugins are the highest version they can be, that still meets your specific configuration.

`terraform plan -out main.tfplan`

If the terminal outputs No changes needed, then congratulations!

**Note:** from my experience it's unlikely to report this. Azure Terraform generation is so flaky that it will always report changes.

## Deploying and removing
You can deploy with `terraform apply main.tfplan` after you've done the plan.
And remove infra with `terraform destroy`, but only do this if your environment is ephemeral.

**Deploying to a new Resource Group**
You need to ensure you change the resource group name if moving to a new resource group. This needs to be changed in multipe places in the file. In the long run you should pass this as a varaible.
You need to move/rename the terraform.tfstate file in the repo you've cloned to, otherwise the resource IDs will conflict in Terraform's view. In the long run they should be hosted somewhere, and you would directly access only the state file you need.
If you're deploying any resources that use a unique hostname, they need to be given a new one.

You can't just "write" everything straight at a resource group, you need to import to your state file first.
`terraform import azurerm_resource_group.<local-name-for-resource> /subscriptions/<your subscription ID>/resourceGroups/<your resource group>`

The "local-name-for-resource" needs to match the name of the root resource in your Terraform file.
i.e. 'resource "azurerm_resource_group" "res-0" {' would have the name "res-0".


## Skipped Resources
When you run the tool there will be a file named aztfexportSkippedResources.txt in your folder.
You'll need to go through and identify if any of these do need to be manually Terraformed, or if there's anything you need to do about scripting things that don't work with Terraform.

## Fixing Errors
When I first ran this I got a load of errors the tool generates out of the box.

<details>
<summary>List of errors you could encounter</summary>

### alpha numeric characters and hyphens only are allowed in "name"
Run this script to remove all _ from name values
TODO
- Remove prepending _ if it exists
- Replace underscores in names
- Replace each name via string replace across the whole file, in case the thing you renamed is referenced anywhere

### Missing required argument
So far I've only received this when it generated "azurerm_storage_account_queue_properties" for my storage account.

Perhaps it's because I'm not actually using any storage accounts that the defaults didn't get filled in. If you've set your queue up I would hope that under the hood Azure has set up the things it's missing here.

The solution is to add the defaults in that are missing. I tried removing the queue properties resource, but when I run `terraform plan` in a later step, it told me that the config actually had to be set.

TODO - app that fixes missing values:
- define a list of parameters and defaults somewhere, I can add to whenever I find missing default values
- look for each value, depending on resource type, and fill in the default if it's missing


### expected site_config.0.ip_restriction_default_action to be one of ["Allow" "Deny"]
If you haven't set any deny list for resource access up this should default to Allow. Perhaps this is too complex for the script.

Run this to default all to Allow or Deny:
TODO - script that lets you pass --allow or --deny in
- Finds all instances of ip_restriction_default_action where there's no value and set it to Allow or Deny
- Finds all instances of scm_ip_restriction_default_action where there's no value and set it to Allow or Deny

### Check depends_on for comments
You might get a message like "# One of azurerm_storage_account.res-7,azurerm_storage_account_queue_properties.res-11 (can't auto-resolve as their ids are identical)" inside the depends_on field, rather than a resource ID.

Running the following can help identify this:
TODO
- Look for all depends_on that start with a "#"
- List the resource IDs and names for each occurence of this

This likely needs to be fixed on a case by case basis, depending on what the dependency is.
For my example above I'm going to go with the first one, as it's the storage account itself.

### Fun with Azure Container Registry
When you generate the Terraform for an ACR, it generates a load of azurerm_container_registry_scope_map.
These were the resources that had the wrong name above.
If you don't rename them, `terraform plan` fails
So I changed the name, then ran `terraform plan`, which told me it was going to rename them (by deleting and re-creating them), that's fine.
`terraform apply` then gives me an error for each of these that they can't be deleted because they're protected.
My next thought is that if I remove these config map resources from the Terraform file they'll be ignored and handled in some default way.
Running `terraform plan` then tells me it's going to delete them completely...
I'll have to accept this for now, and observe what it does when I create an Azure Container Registry straight from Terraform, hopefully it will just sort the default out itself.

### Hostname binding resource already exists
From what I can establish, when you create an App Service resource, it sets up the hostname for you, so if you get any "azurerm_app_service_custom_hostname_binding" resources generated, they should be safe to delete.

</details>
