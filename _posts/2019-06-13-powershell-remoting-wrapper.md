---
layout: post
title:  "Powershell Remoting Wrapper"
categories: [powershell]
tags: [powershell]
---

This article will demonstrate how to automatically wrap a function to make it runnable in a session.

Lets say we have a function that does something useful, like list 


``` powershell
Function New-RemoteFunction {
    Param(
        [Parameter(Mandatory)][Management.Automation.FunctionInfo]$FunctionInfo
    )
}
```
