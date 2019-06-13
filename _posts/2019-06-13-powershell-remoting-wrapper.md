---
layout: post
title:  "Powershell Remoting Wrapper"
---

# Powershell Remoting Wrapper

``` powershell
Function New-RemoteFunction {
    Param(
        [Parameter(Mandatory)][Management.Automation.FunctionInfo]$FunctionInfo
    )
}
```
