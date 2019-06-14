---
layout: post
title:  "Powershell Remoting Wrapper"
categories: [powershell]
tags: [powershell]
author: Christopher Berg Schwanstrøm
---

## Introduction

This article will demonstrate how to automatically wrap a function to make it runnable on remote computers.

Lets say we have a function that does something useful, like listing the top N processes with the highest memory consumption. Lets say we want to run this on remote computers.

``` powershell
Function Get-HighestMemoryUsage {
    Param(
        [Parameter()][int]$Top = 5
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
        [Parameter()][int]$Top = 5,
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
Set-Item -Path Function:global:Get-HighestMemoryUsageRemote -Value (New-RemoteFunction (Get-Command Get-HighestMemoryUsage))

# We can by default create this new function with the same name with Remote tacked on at the end
Get-HighestMemoryUsageRemote -ComputerName computername.domain.local
```

## Template Function

We begin with a template function that we can modify parts of. This could just as well just be a string, but if we have it be a valid Powershell function we still get syntax highlighting and other editor features for it.

``` powershell
Function _RemoteTemplate {
    [CmdletBinding()]
    # We need to replace this with the parameters of the wrapped function,
    # in addition to parameters needed for the remoting
    Param()

    Begin {
        # This is a function that will extract bound parameters 
        # from its calling function (this function) and either return
        # the session that was passed in, or create a new one.
        $sessionList = Get-DynamicRemoteSession

        $sessionList | % {
            # Here we need replace {ArgumentList} with the names of the parameters we added
            # to the Param block at the top
            Invoke-Command -Session $_ -ArgumentList {ArgumentList} -ScriptBlock {ScriptBlock}
        }
    }
    End {
        # If we created a session, this function will remove it
        Remove-DynamicRemoteSession -Session $sessionList
    }
}

```

## Helper Functions

Before we create the function itself, this template functions has a few dependencies on other functions. This is so we can avoid making the template huge, and also some of these functions can be useful elsewhere. First we create the function that will get or create a new session.

This function has a cool technique, in that it looks at the function that called it, and uses the parameters that was given to its parent function. When using this function, we have to make sure that these parameters actually exist on the calling function.

``` powershell
Function Get-DynamicRemoteSession {
    [OutputType([Management.Automation.Runspaces.PSSession[]])]
    Param()

    # We get our calling function
    $stack = Get-PSCallStack
    $caller = $stack[1]

    # This is a hashmap of the parameters sent into the calling function
    $callerParams = $caller.InvocationInfo.BoundParameters

    if ($callerParams["Session"]) {
        return $callerParams["Session"]
    } elseif ($callerParams["ComputerName"]) {
        $sessionList = @()
        foreach ($name in $callerParams["ComputerName"]) {
            $cred = @{}
            if ($callerParams["Credential"]) { $cred["Credential"] = $callerParams["Credential"]}
            $sessionList += New-PSSession -ComputerName $name @cred
        }
        return $sessionList
    } else {
        return
    }
}
```

Next we create the function that will clean up the PSSessions if we created them.

Here we do the same thing, look at the calling function. If the calling function received the ComputerName parameter, we know we created the sessions, and we remove them.

``` powershell
Function Remove-DynamicRemoteSession {
    Param(
        [Management.Automation.Runspaces.PSSession[]]$Session
    )

    $stack = Get-PSCallStack
    $caller = $stack[1]

    $callerParams = $caller.InvocationInfo.BoundParameters

    if ($callerParams["ComputerName"] -and $Session) {
        $Session | Remove-PSSession
    }
}
```

## Generator Function

Finally, we can create the function that will construct the remote version of the function we want to wrap.

We use the FunctionInfo object returned by Get-Command. This contains a whole lot of useful structures that represents the function.

In particular, we use the ScriptBlock.Ast (Abstract Syntax Tree) structure to parse out parts of the function. This is a structure that breaks up all the code into tokens so we can manipulate and parse the code.

``` powershell
Function New-RemoteFunction {
    Param(
        [Parameter(Mandatory)][Management.Automation.FunctionInfo]$FunctionInfo
    )
    $fi = $FunctionInfo

    # Keep a list of the names of the parameters, so we can pass them into the Invoke-Command
    $parameterNameList = @()

    # Prepare the remoting parameters to be added to the wrapper function
    $parameterLines = @(
        "[Parameter(Mandatory, ParameterSetName=`"ComputerName`")][string[]]`$ComputerName",
        "[Parameter(ParameterSetName=`"ComputerName`")][PSCredential]`$Credential",
        "[Parameter(Mandatory, ParameterSetName=`"Session`")] `
        [Management.Automation.Runspaces.PSSession[]]`$Session"
    )

    # Loop through the parameters in the AST and get the strings representing
    # the parameter names and definitions.
    $fi.ScriptBlock.Ast.Body.ParamBlock.Parameters | % {
        $parameterLines += $_.Extent.Text
        $parameterNameList += $_.Name.Extent.Text
    }

    $paramText = "Param(`n" + ($parameterLines -join ",`n") + "`n)"
    $argumentList = ($parameterNameList | % { "$_"}) -join ", "

    # Get the text of our template function and replace the placeholders
    $def = (Get-Item Function:\_RemoteTemplate).Definition
    $def = $def.Replace("Param()", $paramText)
    $def = $def.Replace("{ScriptBlock}", "(Get-Command $($FunctionInfo.Name)).ScriptBlock")
    $def = $def.Replace("{ArgumentList}", "@(" + $argumentList + ")")

    # Create the ScriptBlock and return it
    return [ScriptBlock]::Create($def)
}
```
