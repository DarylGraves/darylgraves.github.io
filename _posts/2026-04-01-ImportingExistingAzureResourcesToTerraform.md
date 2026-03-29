---
title: Importing Existing Azure Resource Groups into Terraform
date: 2026-04-01 00:00:00 -000
image: assets/img/BlogPosts/TerraformIcon.png
categories: [IaC]
description: How to use aztfexport to add existing resource groups into your existing repository
tags: [Terraform, Azure]
---

## Summary
Below is a quick guide on how to grab an existing resource group and add them to an existing state file. I found other guides felt a bit too complicated.

## Prerequisites
- `aztfexport` installed
- `Az CLI` installed, signed in and configured to the target subscription

We're not going to cover these steps in this document as it varies between different OS.

## Steps
*Note: For this example we're using "TEMPRG" as the name of the Resource Group being copied so change it everywhere you see it to the Resource Group you're importing!*

1) In the terminal navigate into the root or your repository,

2) Run: `aztfexport rg -o "TEMPRG" -hcl-only "TEMPRG"` - The `-o "TEMPRG"` is the output folder, `-hcl-only` prevents it from creating statefiles and other bits we don't need and then the final `"TEMPRG"` is the resource group in Azure itself.

3) An interactive screen will then appear where you can browse the resources and check for errors. Do take a few minutes to review this because somethings won't import and in one situation I had to manually re-enable SAS Keys on a Storage Account through the portal because I could import it successfully. Once you have reviewed this, you can push `w` to export. This will then create the subfolder with a collection of your files.

4) Navigate into the `terraform.tf` file which was created in the subfolder and compare the `hashicorp/azurerm` version against the `terraform.tf` file in your main folder - If the subfolder differs you may need to update the main one and run `terraform init -upgrade` to change the version to ensure your Terraform project supports all the parameters being used.
 
5) Delete the `/TEMPRG/provider.tf` and `/TEMPRG/terraform.tf` files as these are not required when importing to an existing repository (we already have the provider)

6) In your `main.tf` file, add a module reference to the folder which contains the code that was generated.

```terraform
module "TEMPRG" {
  source = "./TEMPRG"
}
```

7) To register the module, run `terraform init`

8) The last step is to run a command which loops through the JSON file of resources that was generated (`/TEMPRG/atfexportResourceMapping.json`) and adds them to the State file. This is the command using the "Fish" shell in Linux, at the bottom of this post are a few other variations:

``` shell
set mapfile "/PATH_TO_YOUR_REPO/TEMPRG/aztfexportResourceMapping.json"
set module_name "TEMPRG"

for line in (jq -r --arg mod "$module_name" 'to_entries[] | "module.\($mod).\(.value.resource_type).\(.value.resource_name) \(.value.resource_id)"' $mapfile)
    set parts (string split -m1 ' ' $line)
    set addr $parts[1]
    set id $parts[2]
    echo "Importing $addr"
    terraform import -input=false -lock-timeout=30s "$addr" "$id" < /dev/null
end
```

9) Finally run `terraform plan` and verify that nothing is going to change.

## Conclusion
That's it! My code is now in Terraform and I can add/remove things and know that Terraform will control it.

I'm not sure why but I found the aztfexport tutorials all seemed to be a bit hard to understand so hopefully this gives you a good example of how to use it. If I'm honest, I'm not sure why I have to do manually steps to parse JSON and loop over all the resources to add them into a tfstate file - I thought there would be a command which simply merges two state files - but it doesn't seem worth spending more time on as I've reached my end goal.

The rest of this post are some other variations on step 8 for other terminals. These haven't been tested but will hopefully save you an AI Prompt!

## Other Terminals (Untested - Converted with AI)
### Bash
``` bash
#!/bin/bash
MAPFILE="/PATH_TO_YOUR_REPO/TEMPRG/aztfexportResourceMapping.json"
MODULE_NAME="TEMPRG"

# Use jq to format "address resource_id" pairs
jq -r '. | to_entries[] | "module.'$MODULE_NAME'.\(.value.resource_type).\(.value.resource_name) \(.value.resource_id)"' "$MAPFILE" | while read -r addr id; do
    echo "Importing $addr..."
    terraform import -input=false -lock-timeout=30s "$addr" "$id" < /dev/null
done
```

### PowerShell
``` powershell
$mapFile = "/PATH_TO_YOUR_REPO/TEMPRG/aztfexportResourceMapping.json"
$moduleName = "TEMPRG"

$mappings = Get-Content $mapFile | ConvertFrom-Json

foreach ($key in $mappings.PSObject.Properties.Name) {
    $resource = $mappings.$key
    $addr = "module.$moduleName.$($resource.resource_type).$($resource.resource_name)"
    $id = $resource.resource_id
    
    Write-Host "Importing $addr..."
    terraform import -input=false -lock-timeout=30s "$addr" "$id"
}
```

