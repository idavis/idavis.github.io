---
layout: post
title: "Ad-hoc Polymorphism (Function Overloading) in PowerShell"
date: 2012-08-23 16:59
published: true
categories: powershell
---
Languages like ruby and PowerShell don't actually support ad-hoc polymorphism.

[Ad-hoc polymorphism (Wikipedia)](http://en.wikipedia.org/wiki/Ad-hoc_polymorphism)
> [...] ad-hoc polymorphism is a kind of polymorphism in which polymorphic functions can be applied to arguments of different types, because a polymorphic function can denote a > number of distinct and potentially heterogeneous implementations depending on the type of argument(s) to which it is applied.


Ruby relies on message passing instead of method invocation; a message is sent with a set of arguments and the receiver decides how to respond. PowerShell uses functions in a `Session` keyed on name so that you can only ever have a single function declared in your `Runspace` with a given name. PowerShell does not have support for creating types directly, but instead favors [monkey patching][] using PowerShell's Adaptive Type System (ATS) to add variables, properties, methods, and `ScriptBlocks` to a `PSObject` instance. The same single name rules apply for these instances (You can add additional behavior to other objects, but internally they are wrapped by a `PSObject` instance). In both languages, whenever you define a function/message with a name that is already in use, you are replacing that earlier implementation.

It would be nice PowerShell supported ad-hoc polymorphism as we could do this:

``` powershell
function Get-Distance {
  param(
    [Parameter(Position=0,Mandatory=$true)]
    [psobject]$origin,
    [Parameter(Position=1,Mandatory=$true)]
    [psobject]$target,
  )
  return [Math]::Sqrt( [Math]::Pow(($origin.x - $target.x),2) + [Math]::Pow(($origin.y - $target.y),2))
}

function Get-Distance {
  param(
    [Parameter(Position=0,Mandatory=$true)]
    [psobject]$origin,
    [Parameter(Position=1,Mandatory=$true)]
    [int]$x,
    [Parameter(Position=2,Mandatory=$true)]
    [int]$y
  )
  return [Math]::Sqrt( [Math]::Pow(($origin.x - $x),2) + [Math]::Pow(($origin.y - $y),2))
}
```

To handle this limitation in PowerShell, we can leverage parameter sets to simulate method overloading. Each parameter set can be thought of as an overload (parameters can also be shared across all sets as shown below). There are a number of rules and features to parameter sets which can be found in §17.3.7 of the PowerShell v2 specification that I won't bore you with today.

Let's create a function that computes the distance between two points. It takes a source point (`$origin`) shared between all parameter sets; it also has two sets of mandatory parameters that contains either a second point (`$target`) or two coordinates (`$x`,`$y` values). When the function is called, PowerShell has a built-in object and variable (`$PsCmdlet.ParameterSetName`) that we can query to figure out which set of parameters were used. Using a `switch` block on that variable will determine which code needs to be executed.

```powershell
$point = new-object psobject
$point | Add-Member NoteProperty -Name X -Value 4
$point | Add-Member NoteProperty -Name Y -Value 0

$point2 = new-object psobject
$point2 | Add-Member NoteProperty -Name X -Value 0
$point2 | Add-Member NoteProperty -Name Y -Value 0


function Get-Distance {
  param(
    [Parameter(Position=0,Mandatory=$true)]
    [psobject]$origin,
    [Parameter(Position=1,Mandatory=$true,ParameterSetName="point")]
    [psobject]$target,
    [Parameter(Position=1,Mandatory=$true,ParameterSetName="xy")]
    [int]$x,
    [Parameter(Position=2,Mandatory=$true,ParameterSetName="xy")]
    [int]$y
  )
  switch ($PsCmdlet.ParameterSetName)
  {
    "point" {
      return [Math]::Sqrt( [Math]::Pow(($origin.x - $target.x),2) + [Math]::Pow(($origin.y - $target.y),2))
    }
    "xy" {
      return [Math]::Sqrt( [Math]::Pow(($origin.x - $x),2) + [Math]::Pow(($origin.y - $y),2))
    }
  }
}

Get-Distance $point $point2
Get-Distance $point 0 0
```

This effectively acts like the two methods in the first example. Though we cannot have ad-hoc polymorphism in PowerShell, parameter sets let us simulate the same functionality and flexibility.

  [monkey patching]: http://en.wikipedia.org/wiki/Monkey_patch