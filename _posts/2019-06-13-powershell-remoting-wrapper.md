---
layout: post
title:  "Powershell Remoting Wrapper"
categories: [powershell]
tags: [powershell]

---

# Powershell Remoting Wrapper

``` powershell
Function New-RemoteFunction {
    Param(
        [Parameter(Mandatory)][Management.Automation.FunctionInfo]$FunctionInfo
    )
}
```
