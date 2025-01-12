---
layout: post
title: Install the Centrify Agent For Windows and join the Windows machine to a Zone using a single PowerShell Script


#tags: [books, test]
---
**This blog post show cases remote installation and configuration of software using a single PowerShell script leveraging the WSManCredSSP to get past the second hop hurdle in PowerShell remoting. The post below focuses on installing proprietary software, but the fundermental PowerShell concepts can be used to install any other software/packages onto Windows machines that are joined to an active directory domain.**

Intended Objective for this blog post is to use a single PowerShell script to install the Privileged Elevation agent for Windows and also join the Windows systems to a Centrify zone.

[Full script is here](https://github.com/gracelugandakamya/this-worked-for-me/blob/main/InstallWindowsAgentAndjoinmachinetoaCentrifyZone-github-version.ps1)

**Requirements:**
* Windows machines that are joined to the Active Directory Domain
* The download for the Centrify Privileged Elevation agent(This can be downloaded from the Centrify download center) 

**Explanation of what the script is doing:**
Variables used in the script
```
$source = "C:\Windows\Temp\Centrify Agent for Windows64.exe" This is the location of the downloaded Centrify agent for Windows, in my case this is where the agent is stored on my machine
$computers = Get-content '\\visa2.ocean.net\C$\Windows\Temp\targetmachines2.txt' This is the list of remote machines that I want the Centrify agent for Windows to be installed on.
$destination = "C$\Windows\Temp\" This is where the Centrify Windows agent will be temporarily copied to the remote Windows machines.
$credential = Get-Credential -Credential OCEAN\langley This gets the credentials that will be used in the PowerShell script to perfrom the needed actions on those remote machines.
```
Once our variables are declared, we proceed with the rest of the steps:

This step enables the Enable-WSManCredSSP on the local machine.

```Enable-WSManCredSSP -Role Client -DelegateComputer *.ocean.net -Force```

This next step enables use of the delegated credentials on the remote target servers where we want to install the Centrify agent.

```Invoke-Command -ComputerName $computers -ScriptBlock {Enable-WSManCredSSP -role server -Force} -Credential $credential```


The command below pulls all the variables above and performs the file copy onto the remote machines

```foreach ($computer in $computers) 
{ if (Test-Path -Path \\$computer\$destination) 
{ Copy-Item $source -Destination "\\$computer\$destination" -Recurse -Verbose} ```


**Once the installer executable is copied, proceed to install it silently, the command below installs it silenty on the remote machines**

```Invoke-Command -ComputerName $computers -ScriptBlock {Start-Process 'C:\Windows\Temp\Centrify Agent for Windows64.exe' -ArgumentList "q" -Wait} -Verbose}```


After the agent is installed clean up the agent package off of the remote machines
The command below pulls all the variables above and performs the file removal on the remote target machines

```foreach ($computer in $computers) 
{ if (Test-Path -Path \\$computer\$destination) 
{ Remove-Item -Path "\\$computer\$destination\Centrify Agent for Windows64.exe" -Verbose} ```

This next variable sets a PowerShell session on those target remote machines
```$session = New-PSSession -ComputerName $computers -Credential $credential -Authentication Credssp -Verbose```

This step closes the current PSSession.
```Remove-PSSession -ComputerName $computers  -Verbose```




This step tells the script to wait for 60 seconds as the Centrify agent finishes installing the Agent and laying down libraries.That way our dzjoin command does not fail when it is run.

```Start-Sleep -seconds 600 -Verbose```


This variable gets the credential from the user running the script

```$credential2 = Get-Credential -Credential OCEAN\langley -verbose```

This step creates a new PSSession so that the dzjoin can be run on the remote target machines

```$session2 = New-PSSession -ComputerName $computers -Credential $credential2 -Authentication Credssp -Verbose```




This is to run the dzjoin command on the target remote machines so the machines can join the Centrify zone in active directory.
```Invoke-Command -Session $session2  -ScriptBlock {dzjoin /r yes /z ZoneName} -Verbose }```




