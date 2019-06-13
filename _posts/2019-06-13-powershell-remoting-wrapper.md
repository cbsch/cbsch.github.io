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
    # We need to replace this with the parameters of the wrapped function, in addition to parameters needed for the remoting
    Param()

    Begin {
        # This is a function that will extract bound parameters from it's calling function (this function) and either return
        # the session that was passed in, or create a new one.
        $sessionList = Get-DynamicRemoteSession

        # Here we need to replace this bit with the script definition of our wrapped function
        #$scriptBlock = {ScriptBlock}

        $sessionList | % {
            # Here we need replace {ArgumentList} with the names of the parameters we added in to the Param block at the top
            Invoke-Command -Session $_ -ArgumentList {ArgumentList} -ScriptBlock {ScriptBlock}
        }
    }
    End {
        # If we created a session, this function will remove it
        Remove-DynamicRemoteSession -Session $sessionList
    }
}

```

Before we create the function itself, this template functions has a few dependencies on other functions. This is so we can avoid making the template huge, and also some of these functions can be useful elsewhere. First we create the function that will get or create a new session.

This function has a cool technique, in that it looks at the function that called it, and uses the parameters that was given to it's parent function. When using this function, we have to make sure that these parameters actually exist on the calling function.

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

Finally, we can create the function that will construct the remote version of the function we want to wrap.

The object returned by Get-Command is a FunctionInfo object that contains a whole lot of useful structures that represents the function.

In particular, we use the ScriptBlock.Ast (Abstract syntax tree) structure to parse out parts of the function. This is a structure that breaks up all the code into tokens so we can manipulate and parse the code.

``` powershell
Function New-RemoteFunction {
    Param(
        [Parameter(Mandatory)][Management.Automation.FunctionInfo]$FunctionInfo,
        [Parameter()][string]$Name
    )
    $fi = $FunctionInfo

    $parameterNameList = @()
    $parameterLines = @(
        "[Parameter(Mandatory, ParameterSetName=`"ComputerName`")][string[]]`$ComputerName",
        "[Parameter(ParameterSetName=`"ComputerName`")][PSCredential]`$Credential",
        "[Parameter(Mandatory, ParameterSetName=`"Session`")][Management.Automation.Runspaces.PSSession[]]`$Session"
    )
    $fi.ScriptBlock.Ast.Body.ParamBlock.Parameters | % {
        $parameterLines += $_.Extent.Text
        $parameterNameList += $_.Name.Extent.Text
    }

    $paramText = "Param(`n" + ($parameterLines -join ",`n") + "`n)"
    #$paramText = $fi.ScriptBlock.Ast.FindAll({$args[0] -is [System.Management.Automation.Language.ParamBlockAst]}, $true).Extent.Text

    $argumentList = ($parameterNameList | % { "$_"}) -join ", "

    $templateDefinition = (Get-Item Function:\_RemoteTemplate).Definition

    $def = $templateDefinition.Replace("Param()", $paramText)
    $def = $def.Replace("{ArgumentList}", "@(" + $argumentList + ")")
    $def = $def.Replace("{ScriptBlock}", "(Get-Command $($FunctionInfo.Name)).ScriptBlock")

    $def | Out-Host

    $newScript = [ScriptBlock]::Create($def)

    if ($Name) {
        Set-Item -Path Function:global:"$Name" -Value $newScript
    } else {
        Set-Item -Path Function:global:"$($FunctionInfo.Name)Remote" -Value $newScript
    }
}
```
