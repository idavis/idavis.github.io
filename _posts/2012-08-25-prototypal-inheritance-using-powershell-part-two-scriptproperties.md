---
layout: post
title: "Prototypal Inheritance Using PowerShell (Part 2): ScriptProperties"
date: 2012-08-25 06:39
published: true
categories: powershell patterns
---

*This post is part of a four part series on prototypal inheritance using PowerShell.*
- [Part 1][] Introduction
- [Part 2][] ScriptProperties
- [Part 3][] Mixins
- [Part 4][] Static Properties

If you didn't read [Part 1][], I would recommend reading it as I build off of its functionality and theory.

Adding functions and variables to a PowerShell object is very useful, but `ScriptProperty` objects give us some unique features (and gotchas) as well. We can attach them just as we did with other prototype helpers:

``` powershell
function Add-ScriptProperty {
  param(
    [string]$name, 
    [scriptblock]$getter,
    [scriptblock]$setter = $null
  )
  process {
    $property = new-object System.Management.Automation.PSScriptProperty "$name", $getter, $setter
    $_.psobject.properties.remove($name)
    $_.psobject.properties.add($property)
  }
}
```

This kind of property comes in very handy for creating proxy properties, composite properties, and syntax helpers:

``` powershell
function New-EnvironmentModifier {
  $prototype = New-Prototype
  $prototype | Add-Function SetUserVariable {
                 param([string]$name, [string]$value)
                 [Environment]::SetEnvironmentVariable($name, $value, "User")
               }
  $prototype | Add-Function GetUserVariable {
                 param([string]$name)
                 [Environment]::GetEnvironmentVariable($name,"User")
               }
  $prototype | Add-ScriptProperty BuildNumber {
                                      $this.GetUserVariable("BuildNumber")
                                    } {
                                      param([String]$value)
                                      $this.SetUserVariable("BuildNumber", $value)
                                    }
  $prototype
}

$modifier = New-EnvironmentModifier
$modifier.BuildNumber = 42
$modifier.BuildNumber
42
```

You MUST remember that `ScriptBlock`s are not closures and you cannot capture variables with them! You can get into situations in which it appears that your code is working, but it is in fact accessing a variable from your current scope. The easiest mistake to make is using your `$prototype` variable in a new function or `ScriptProperty`.

``` powershell
function New-ClosureSample {
  param($target)
  $prototype = New-Object psobject
  $prototype | Add-ScriptProperty Value { $target }
  $prototype
}

$sample = New-ClosureSample 37
# If the variable was captured, we should see 37, but instead we get $null
$sample.Value -eq $null
True
$target = 42
$sample.Value
42
```

Once `$target` is defined, the `Value` block has data to work with. Remember to always use $this to access members of your object within a `ScriptProperty`. If you need to capture an argument, use a regular property (then optionally refer to it from the `ScriptProperty`)

``` powershell
function New-ClosureSample {
  param($target)
  $prototype = New-Object psobject
  $prototype | Add-Property Captured $target
  $prototype | Add-ScriptProperty Value { $this.Captured }
  $prototype
}

$sample = New-ClosureSample 37
$sample.Value 
37
```

In a `ScriptProperty` block you have to use the `$this` automatic variable. From the about_Automatic_Variables help topic:

`$This`
> In a script block that defines a script property or script method, the $This variable refers to the object that is being extended.

In the case of our prototype, `$this` refers to the underlying `PSObject` allowing use to use functions, properties, and variables that we have attached and any others that the object provides.

  [Part 1]: /2012/08/prototypal-inheritance-using-powershell
  [Part 2]: /2012/08/prototypal-inheritance-using-powershell-part-two-scriptproperties
  [Part 3]: /2012/08/prototypal-inheritance-using-powershell-part-three-mixins
  [Part 4]: /2012/08/prototypal-inheritance-using-powershell-part-4-static-properties