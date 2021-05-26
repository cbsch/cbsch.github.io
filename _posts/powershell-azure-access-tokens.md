---
title:  "Powershell basic data structures"
categories: [powershell, azure]
tags: [powershell, azure]
author: Christopher Berg Schwanstrøm
---

# Aquiring access tokens for various Azure services

Various Azure services can be connected to via an already established AzContext.

This is a a general example for creating a graph token:

```powershell
Import-Module Az
$tenantId = "<tenantId>"
$service = "https://graph.microsoft.com"

$context = Get-AzContext
$graphToken = [Microsoft.Azure.Commands.Common.Authentication.AzureSession]::Instance.AuthenticationFactory.Authenticate(
    $context.Account,
    $context.Environment,
    $TenantId,
    $null,
    [Microsoft.Azure.Commands.Common.Authentication.ShowDialog]::Never,
    $null,
    $service
).AccessToken
```

### Other URLs:
AzureAD: `https://graph.windows.net`
PartnerCenter: `https://api.partnercenter.microsoft.com`


## Practical usage

```Powershell
Function Get-AzureAccessToken {
    Param(
        [Parameter(Mandatory)][string]$TenantId,
        [Parameter(Mandatory)][string]$Uri,
    )
    $context = Get-AzContext
    return [Microsoft.Azure.Commands.Common.Authentication.AzureSession]::Instance.AuthenticationFactory.Authenticate(
        $context.Account,
        $context.Environment,
        $TenantId,
        $null,
        [Microsoft.Azure.Commands.Common.Authentication.ShowDialog]::Never,
        $null,
        $Uri
    ).AccessToken
}
```

With this function we can connect to various Azure services

#### AzureAD

```powershell
$context = Get-AzContext
$tenantId = "<tenantId>"
$aadToken = Get-AzureAccessToken -TenantId $tenantId -Uri "https://graph.windows.net"
$graphToken = Get-AzureAccessToken -TenantId $tenantId -Uri "https://graph.microsoft.com"
Connect-AzureAD -AadAccessToken $aadToken -AccountId $context.Account.Id -TenantId $tenantId -MsAccessToken $graphToken
```

#### PartnerCenter

```powershell
Import-Module PartnerCenter
$token = Get-AzureAccessToken -TenantId "<tenantId>" -Uri "https://api.partnercenter.microsoft.com"
Connect-PartnerCenter -AccessToken $token
```