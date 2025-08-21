---
title: A Script to Purge Container Registries
date: 2025-8-1 00:00:00 -000
image: assets/img/BlogPosts/AzureContainerRegistry.png
categories: [Dev]
description: Are you reusing tags in your ACR? The old ones are still taking up storage!
tags: [Docker, Dev, Containers, Azure]
---

# Problem
I recently found out the hard way that if you deploy more than one docker image with the same tag into an Azure Container Registry (ACR), it doesn't actually delete the previous version! Instead, it just removes the tag from the old version but the manifest still exists and so your ACR storage will continue to grow over time.

There is a [Microsoft ACR-CLI Tool](https://github.com/Azure/acr-cli) which you can use to purge untagged images but be warned - at the time of writing, running `acr purge --untagged` can delete *everything*! Luckily I ran `--dry-run` first... For more info, see the GitHub issue I raised [here](https://github.com/Azure/acr-cli/issues/480).

# Solution
Here is a script I wrote to purge my Container Registry of old, untagged manifests. I only spent an hour or two on it so feel free to improve on it, but it did the job.

``` powershell
#-------------------------------------------------------------------------------
# Variables
#-------------------------------------------------------------------------------
$Subscription = "SubName"
$AcrName = "AcrName"
$TestRepo = "TestRepoName" # If debug = true, only this repo is targeted

$SafeMode = $true # When true, you are prompted to accept before deletion
$Debug = $false

#-------------------------------------------------------------------------------
# Script
#-------------------------------------------------------------------------------
Set-AzContext -Subscription $Subscription | Out-Null
$Repositories = Get-AzContainerRegistryRepository -RegistryName $AcrName

if ($Debug) {
    # Filter to one repository for testing
    $Repositories = $Repositories | Where-Object { $_ -eq $TestRepo } 
}

foreach ($repo in $Repositories) {
    Write-Host "-------- Currently on $repo --------" -ForegroundColor Magenta

    # Get images without tags
    $untaggedImages = ((Get-AzContainerRegistryManifest -RegistryName $AcrName -RepositoryName $Repo).ManifestsAttributes  | Where-Object { $_.Tags.Count -eq 0 }).Digest
    $taggedImages = ((Get-AzContainerRegistryManifest -RegistryName $AcrName -RepositoryName $Repo).ManifestsAttributes  | Where-Object { $_.Tags.Count -ne 0 }).Digest

    # Display results
    Write-Host "Image to Keep:" -ForegroundColor Yellow
    $taggedImages | Sort-Object $_ | Foreach-Object { Write-Host "`t$_" }

    Write-Host "Images to Delete:" -ForegroundColor Yellow
    if ($untaggedImages.count -eq 0) {
        Write-Host "`tNone! Skipping..."
        continue
    }
    $untaggedImages | Sort-Object $_ | Foreach-Object { Write-Host "`t$_" }

    # Prompt for user to continue
    if ($SafeMode) {
        $Answer = Read-Host "Ok to continue? (y/n)"
        
        if ($Answer -ne "y") {
            Write-Host "Skipping..."
            continue
        }
    }

    # Remove
    foreach ($image in $untaggedImages) {
        Remove-AzContainerRegistryManifest -RegistryName $AcrName -RepositoryName $repo -Manifest $image | Out-Null
    }
}
```

# Summary
Use this at your own risk, I won't accept responsibility if you take down production! All I can say is it worked for me.

Also, it's probably up for debate whether I should be overwriting images with the same tags anyway but that's a topic for another day...

# Note
I just finished writing this and noticed on GitHub that the Acr CLI is about to get a new parameter, likely called `--untagged-only`. This can be seen in this pull request [here](https://github.com/Azure/acr-cli/pull/489).

So this whole post might soon be irrelevant, but since I spent 30 minutes writing it, I’m publishing it anyway so I don't skip another month! 