---
title:  "Powershell basic data structures"
categories: [powershell]
tags: [powershell]
author: Christopher Berg Schwanstrøm
---

# Basic data structures

The most basic ways in Powershell (and in all other languages) to organize data is with lists or dictionaries.


## List / Array

Declaring a list is done using a special syntax: `@()`. So a list from 1 to 5 would be declared like this: `@(1,2,3,4,5)`.

There are also a shortcut to declaring sequences. The same can be achieved with `1..5`. This also works for letters: `"a".."e"`

Accessing individual elements of a list can be done like this:
```powershell
$list = 1..5
$list[2]
# Outputs 3
```

## Dictionary

A dictionary can be declared like this: `@{}`

Declaring a dictionary with values:

```powershell
$dict = @{
    Name = "Test"
    Value = 123
}
```

Accessing dictionary elements are commonly done one of two ways:
```powershell
$dict["Name"]
# Outputs "Test"
$dict.Name
# Outputs "Test"
```

## Complex objects and other common formats

Declaring a complex object in Powershell
```powershell
$object = @{
    list = @(1,2,3,4,5)
    objects = @(@{
        name = "Test1"
        value = 1
    }, @{
        name = "Test2"
        value = 2
    })
}
```

JSON equivalent
```json
{
  "list": [1, 2, 3, 4, 5],
  "objects": [{
    "name": "Test1",
    "value": 1
  }, {
    "name": "Test2",
    "value": 2
  }]
}
```

YAML equivalent
```yaml
list:
- 1
- 2
- 3
- 4
- 5
objects:
- name: Test1
  value: 1
- name: Test2
  value: 2
```