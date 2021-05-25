---
title:  "Powershell Pipelines"
categories: [powershell]
tags: [powershell]
author: Christopher Berg Schwanstrøm
---

# Powershell Pipelines

Powershell pipelines are in essence a replacement for loops. They are also frequently used to manipulate single objects.

## Example

#### Foreach
```powershell
$list = 1..5
$result = @()
foreach ($element in $list) {
    $result += $element * 2
}
```

#### Pipeline
```powershell
$list = 1..5
$result = $list | % { $_ * 2 }
```

> `%` is an alias for the cmdlet `ForEach-Object`. This alias, along with `?` for `Where-Object` are two special aliases for two of the most common cmdlets.

In both these cases the value of `$result` will be `@(2, 4, 6, 8, 10)`

> `@()` is the Powershell syntax for a list, or array. The other basic data structure is a dictionary, which is declared like this: `@{}`

### Pipeline execution

Every element on the left side of the pipeline will be processed by the expression on the right side of the pipeline

`<input> | <process>`

Pipelines can also be chained, so that the output of each pipe is sent to the next

`<input> | <process-first> | <process-second>`

An important thing to note is that all the pipelines will execute at the same time. Each element will be passed through the entire pipeline chain before the next element is processed. That means that this expression will be equivalent to a single loop, and not two loops

```powershell
1..5 | % {
    Write-Host "Pipeline 1 processing $_"
    $_ # Pass the object to the next pipe
} | % {
    Write-Host "Pipeline 2 processing $_"
}
```

One might think that this will be equivalent to this:
```powershell
$list = 1..5
foreach ($obj in $list) {
    Write-Host "Pipeline 1 processing $obj"
}
foreach ($obj in $list) {
    Write-Host "Pipeline 2 processing $obj"
}
```

But it will be equivalent to this:
```powershell
foreach ($obj in 1..5) {
    Write-Host "Pipeline 1 processing $obj"
    Write-Host "Pipeline 2 processing $obj"
}
```

This applies to most cmdlets, but there may be exceptions. Some cmdlets will by their nature need to process the entire input before they can output a result. An example of this is the `Group-Object` cmdlet.

```powershell
1..3 + 1..5 |
    % { Write-Host "Pipeline 1 processing $_"; $_ } |
    % { Write-Host "Pipeline 2 processing $_"; $_ } |
    Group-Object |
    # Group-Object will create new objects with the properties Name, Count and Group
    % { Write-Host "Pipeline 3 processing $($_.Name)"}
```
