---
title: Authentication in Azure PowerShell Function Apps 
date: 2026-01-01 00:00:00 -000
image: assets/img/BlogPosts/AzureFunctionAppIcon.png
categories: [Dev]
description: How to retrieve authenticated users when the process differs between development and production.
tags: [PowerShell, Dev, Api, Function Apps]
---

# Summary
When a Function App is configured to require authentication in Azure, Azure injects headers into the requests which detail the authenticated user's information. Whilst this is definitely helpful, we don't have access to these headers when writing the code locally and so if you need to do anything with the authenticated user's information then you need to write different code to cover both your local and deployed environments.

To resolve this, I created a function called `Get-UserFromHeaders` which abstracts the process for me. This function retrieves and returns the authenticated user's email address regardless of the environment, pivoting the approach based on the storage account connection string.

**Full disclaimer:** Whilst I knew how I wanted the function to work, the logic to extract email addresses from a JWT was written by AI.

# How it Works
The structure of the function is simple:

- If `$Env:AzureWebJobStorage -eq "UseDevelopmentStorage=true"` then we know we're using the Azurite Storage Emulator. This is *usually* only used in local environments so in this case we don't have the headers and must manually extract the email address from a JSON Web Token (JWT) which is added by our front-end.
- If the above is not true, then we're using real storage in Azure. This means we're probably also running the function in Azure and so Azure will have automatically added a header called `x-ms-client-principal-name` for us.

# The Code
Add the below function to `profile.ps1` so it is a global function and can be accessed by all other Azure Functions in the project:

```powershell
function Get-UserFromHeaders {
    <#
    .SYNOPSIS
    Gets the user email address based on the HTTP Headers.
    .NOTES
    We have a two-pronged approach depending on whether this is using Development Storage or running in Azure. When running locally this will extract data from the JWT to get user details but when Authentication is enabled on the Function in Azure itself, it will automatically append a new header so we can just return this for less computational overhead.
    #>
    param(
        [HashTable]$Headers
    )

    Write-Information "Get-UserFromHeaders called."

    if ($Env:AzureWebJobsStorage -eq "UseDevelopmentStorage=true") {
        # Running in Dev so extract value from JWT
        Write-Information "Running locally so extracting JWT"

        $authHeader = $Headers["authorization"]

        if ($authHeader -and $authHeader.StartsWith("Bearer ")) {
            $token = $authHeader.Substring(7)
            
            # Decode the JWT (header.payload.signature)
            $payload = $token.Split(".")[1]
            $padded = $payload.Replace('-', '+').Replace('_', '/')
            switch ($padded.Length % 4) {
                2 { $padded += '==' }
                3 { $padded += '=' }
            }
            $bytes = [Convert]::FromBase64String($padded)
            $json = [System.Text.Encoding]::UTF8.GetString($bytes)
            $claims = $json | ConvertFrom-Json

            Write-Information "Returning $($claims.unique_name)"
            return $claims.unique_name
        }
        else {
            return $null
        }
    }

    # Running in Prod so Azure created the header for us
    $toReturn = $Headers["x-ms-client-principal-name"]
    Write-Information "Returning $toReturn"
    return $toReturn
}
```
# Usage
In Azure Functions which are triggered by HTTP requests, we can call our global function like so:
```powershell
param($Request, $TriggerMetadata)

$User = Get-UserFromHeaders -Headers $Request.Headers
Write-Debug "Create Request called by $User"
```

Here is a second more detailed example which assumes `$Env:Approvers` is an array of email addresses:

```powershell
param($Request, $TriggerMetadata)

$User = Get-UserFromHeaders -Headers $Request.Headers
Write-Debug "Create Request called by $User"

if($User -notin $Env:Approvers)
{
    Push-OutputBinding -Name Response -Value ([HttpResponseContext]@{
            StatusCode = [HttpStatusCode]::BadRequest
            Body       = "User is not an approver so cannot approve requests"
        }
    )
}

# If we get this far then the user is able to approve requests
```
# Notes
- The key thing here is we're pivoting based on our connection string. I use the Azurite Storage Emulator but if your local development is still using Azure storage, you may need to change the `if` statement being checked.
- This code has been written and tested on Entra Authentication only so may require tweaking for other Identity Providers - They may not use the `unique_Name` claim.