---
title: Enabling Predictive-Intellisense in the PowerShell Terminal
description: Make PowerShell suggest previous things you've typed!
date: 2024-03-24 00:00:00 -000
image: assets/img/BlogPosts/MatrixIcon.png
categories: [Productivity, Automation]
tags: [PowerShell, PSReadLine]
---

# Overview

Enabling Predictive-Intellisense allows commands you have previously entered to appear as suggestions whilst you type. Great for when you often find yourself repeating the same one-liners of code again and again.

![PsReadLine Example](assets/img/BlogPosts/PsReadLineHistoryExample.gif)
*Note how just typing one letter shows suggestions from my terminal history*

## Prerequisites
- Windows PowerShell 5.1 or PowerShell 7 and above

## Quick Setup
- In PowerShell, run the below code:
```powershell
Install-Module PsReadLine -AllowClobber -Force
Notepad $Profile 
```
- Notepad should now open (if Notepad warns you that the file doesn't exist, say yes to the prompt which appears to create the file).

- Copy-paste the below code into Notepad: 
```powershell
Set-PSReadLineOption -PredictionViewStyle ListView
```
- Save and close the file in Notepad.
- Restart PowerShell.

That's it! The more you use PowerShell, the more suggestions will start appearing.

## Breaking Down the Above
- In the first PowerShell line, we're making sure we have the latest version of the `PsReadLine` module installed.
    - Without `-AllowClobber -Force` you might receive an error that PsReadLine is already installed but by a different method. This should still ensure the installation is successful.
- In the second line, we're opening your PowerShell "profile" in Notepad. In PowerShell, a profile is simply a PowerShell file which runs everytime you start up the program. 
- The content we paste into the profile is the command which enables the history feature and, because it's now in your profile, it will now load each time you open PowerShell.

## How Does this Work?
Behind the scenes, the PsReadLine module is saving a history of your commands to a text file. You can find the location of this file by entering `(Get-PsReadLineOption).HistorySavePath` into your terminal.

It's worth noting that this text file is in plain-text so can be easily read or modified. PsReadLine knows not to save some commands (such as `ConvertTo-SecureString`) but it's worth reviewing this file on occasion to make sure nothing sensitive has slipped through the net.

## Troubleshooting Errors
- The above is tried and tested on Windows but _should_ work on Linux/Mac, you will just need to edit your profile with your preferred text editor instead.
- If you're seeing lots of red text which includes the line `You cannot run this script on the current system. For more information about running scripts and setting execution policy`, run the below:
```powershell
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope CurrentUser
```
It is important to understand that the ExecutionPolicy is there to prevent you from running code you do not understand! By changing it to `Unrestricted` you are telling PowerShell that you know what you're doing.

## Would you like to Know More?
See [Announcing PsReadLine 2.1 with Predictive Intellisense](https://devblogs.microsoft.com/powershell/announcing-psreadline-2-1-with-predictive-intellisense/).