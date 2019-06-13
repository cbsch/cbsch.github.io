---
layout: post
title:  "Powershell Remoting Wrapper"
categories: [powershell]
tags: [powershell]
---

This article will demonstrate how to automatically wrap a function to make it runnable on remote computers.

Lets say we have a function that does something useful, like listing the top N processes with the highest memory consumption. Lets say we want to run this on remote computers.

``` powershell
Function Get-HighestMemoryUsage {
    Param(
        [Parameter(Mandatory)][int]$Top = 5
    )

    Get-WmiObject Win32_PerfRawData_PerfProc_Process |
        Sort-Object WorkingSetPrivate -Descending |
        Select-Object -First $Top |
        Format-Table Name, WorkingSetPrivate, IDProcess -AutoSize
}
```

In order to do this, we could extend the function with parameters taking in either a PSSession or ComputerName and other parameters needed to run the actual code in an Invoke-Command, like this.

``` powershell
Function Get-HighestMemoryUsage {
    Param(
        [Parameter(Mandatory)][int]$Top = 5,
        [Parameter(Mandatory)][string]$ComputerName
    )

    Invoke-Command -ComputerName $ComputerName -ArgumentList @($Top) -ScriptBlock {
        Param(
            [Parameter(Mandatory)][int]$Top
        )
        Get-WmiObject Win32_PerfRawData_PerfProc_Process |
            Sort-Object WorkingSetPrivate -Descending |
            Select-Object -First $Top |
            Format-Table Name, WorkingSetPrivate, IDProcess -AutoSize
    }
}
```

Already with this simple function that only takes one parameter, this quickly bloats up with plumbing that makes the code quite noisy. And here we only support the ComputerName parameter for Invoke-Command. Maybe we wanted to specify Credentials, Authentication, or optionally a session. For more advanced functions with lots of parameters, supporting these extra things for the remoting, this quickly obfuscates the intention of the function.

``` powershell
Function New-RemoteFunction {
    Param(
        [Parameter(Mandatory)][Management.Automation.FunctionInfo]$FunctionInfo
    )
}
```
