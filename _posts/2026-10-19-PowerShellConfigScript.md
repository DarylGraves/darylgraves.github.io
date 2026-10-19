---
title: PowerShell Lightweight Config Script
date: 2026-10-19 00:00:00 -000
image: assets/img/BlogPosts/MatrixIcon.png
categories: [Automation, Productivity]
description: A small set of PowerShell functions I wrote for quickly storing and retrieving variables from the registry
tags: [PowerShell]
---

## Summary
*(It seems my goal to write one blog post a month failed! All is not lost though, I'm still aiming for 12!)*

Over the years, I've had to write a number of scripts which need to store a setting or piece of data and bring it back in subsequent runs. On a Windows machine, the best way of doing this is to store them in registry under the user's specific hive (`HKCU:\`).

Everytime I need to do do this, I forget the exact PowerShell syntax. I know I use `New-Item` to create a folder but in the registry you have folders and then key-value pairs and things can be nested and it all gets a bit confusing with `New-ItemProperty` so I always have to do a bit of trial and error.

On Friday, I realised this is something that keeps happening so I took the time to create 3x basic functions. A mini-module if you will. I've calledit "Config".

## How It Works
### Creating a Config
1. Create a hashtable of the default values you want to store:
```powershell
$defaults = @{
    favouriteFood = "Pizza"
    favouriteDrink = "Water"
}
```
2. Call `Initialize-Config` and store the result in a variable. When you are calling teh function you need to pass in the name of the configuration you're creating/retrieving and pass in the hashtable.
```powershell
$Config = Initialize-Config -ConfigName "Test" -Defaults $defaults
``` 

If it doesn't exist `Initialize-Config` will create a folder `HKCU:\Software\PowerShell\$ConfigName` and go through the hash table. If any value in the hash table doesn't exist in the registry yet, it will create it using the default value defined in the hash table.

The function will then return the values in the registry.

### Using the Config
To use variables retrieved by config you can simply view the properties of the variable the data was outputted to. In the above example, we stored the results in the `$Config` variable:
```powershell
$Config.Test.favouriteFood # Pizza
$Config.Test.favouriteDrink # Water
```

### Changing Variables
To change Variables simply use `Set-ConfigVariable`. This requires the config name, a name of the variable you're creating/editing and the value you wish to store. It will then update the variable in the registry but also return it back to you.
```powershell
$Config = Set-ConfigVariable -ConfigName "Test" -VariableName "favouriteFood" -Value "Chocolate"
```

## The Code
Simply copy this into your script:
```powershell
function Initialize-Config {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory = $true)]
        [string]$ConfigName,
        [Parameter(Mandatory = $true)]
        [hashtable]$Defaults
    )

    Write-Verbose "Config: Initialize-Config function starting."

    $regPath = "HKCU:\Software\PowerShell\$ConfigName"
    Write-Verbose "Config: regPath = $regPath"

    $configVars = @{}
    if (-Not(Test-Path -Path $regPath)) {
        Write-Verbose "Config: First time run - Creating Registry Folder"
        New-Item -Path $regPath -Force | Out-Null
    }

    Write-Verbose "Config: Loading Variables"
    foreach ($keyName in $Defaults.Keys) {
        if (-Not(Get-ItemProperty -Path $regPath -Name $keyName -ErrorAction SilentlyContinue)) {
            Write-Verbose "Config: Variable $keyName missing so setting to default ($($Defaults[$keyName]))"
            Set-ItemProperty -Path $regPath -Name $keyName -Value $Defaults[$keyName] -Force | Out-Null
            $configVars.Add($keyName, $Defaults[$keyName] )
        }
    }

    return Get-Config -ConfigName $ConfigName
}

function Set-ConfigVariable {
    [CmdletBinding()]
    param(
        [string]$ConfigName,
        [string]$VariableName,
        [string]$Value
    )

    Write-Verbose "Config: Set-ConfigVariable starting"

    $regPath = "HKCU:\Software\PowerShell\$ConfigName"
    Write-Verbose "Config: regPath = $regPath"

    Set-ItemProperty -Path $regPath -Name $VariableName -Value $Value -Force | Out-Null
    Get-Config -ConfigName $ConfigName
}

function Get-Config {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string]$ConfigName
    )

    Write-Verbose "Config: Get-Config starting"

    $regPath = "HKCU:\Software\PowerShell\$ConfigName"
    Write-Verbose "Config: regPath = $regPath"

    $configVars = @{}
    $properties = (Get-ItemProperty -Path $regPath).PSObject.Properties |
    Where-Object Name -NotMatch '^PS'

    foreach ($property in $properties) {
        $configVars[$property.Name] = $property.Value
    }

    return [PSCustomObject]@{
        $ConfigName = [PSCustomObject]$configVars
    }
}
```

## Limitations
- As the variables are stored in the registry they must be flat objects. Mainly strings.
- Linux doesn't have a registry so in the future I will probably add a text file alternative that stores the data in `~/.config/PowerShell`

## Small Note
After writing this function and getting Gemini to help me tweak it to make it better, Gemini did point out that something much more comprehensive exists in PSFramework but I have no regrets. This was fun to make and can be used for smaller projects when you don't want to worry about downloading additional modules.