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

When writing a lot of Powershell for system administration, we probably want to write a whole bunch of functions capable of remoting, and all this plumbing seems rather redundant. So instead of writing a whole bunch of code that all looks the same, we can generate it.

The result we want to end up with is something like this

``` powershell
# We want this function to add a new function to the global scope with remoting capabilities
New-RemoteFunction (Get-Command Get-HighestMemoryUsage)

# We can by default create this new function with the same name with Remote tacked on at the end
Get-HighestMemoryUsageRemote -ComputerName computername.domain.local
```

To achieve this, we begin with a template function that we can modify parts of.

``` powershell
Function _RemoteTemplate {
    [CmdletBinding()]
    # We need to replace this with the parameters of the wrapped function
    {Param}

    # This is a function we will create to dynamically add the remoting specific arguments to our generated functions.
    # It's not strictly neccessary to do it this way, we could just as well add them directly to the Param block.
    DynamicParam { Get-DynamicRemoteParams }

    Begin {
        # This is a function that will extract bound parameters from it's calling function (this function) and either return
        # the session that was passed in, or create a new one.
        $sessionList = Get-DynamicRemoteSession

        # Here we need to replace this bit with the script definition of our wrapped function
        $scriptDefinition = {ScriptDefinition}

        $sessionList | % {
            # Here we need replace {ArgumentList} with the names of the parameters we added in to the Param block at the top
            Invoke-Command -Session $_ -ArgumentList {ArgumentList} -ScriptBlock ([ScriptBlock]::Create($scriptDefinition))
        }
    }
    End {
        # If we created a session, this function will remove it
        Remove-DynamicRemoteSession -Session $sessionList
    }
}

```

``` powershell
Function New-RemoteFunction {
    Param(
        [Parameter(Mandatory)][Management.Automation.FunctionInfo]$FunctionInfo,
        [Parameter()][string]$Name
    )
}
```
