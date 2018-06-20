---
layout: post
title: "Prototypal Inheritance Using PowerShell"
date: 2012-08-24 10:15
published: true
categories: powershell patterns
---

*This post is part of a four part series on prototypal inheritance using PowerShell.*
- [Part 1][] Introduction
- [Part 2][] ScriptProperties
- [Part 3][] Mixins
- [Part 4][] Static Properties

PowerShell has no concept of a class, so classical inheritance is not an option; however, PowerShell's Adaptive Type System (ATS) gives us great power in implementing prototypal objects. [Prototype-based programming][] relies on delegation to execute late bound features of an object. Ruby is famous for its message delegation capabilities. PowerShell, on the other hand, has some shortcomings.

PowerShell, even the new [Dynamic Language Runtime][] (DRL) based v3, does not support true [dynamic dispatch][] and we cannot get into the internals of `PSObject`'s `IDynamicMetaObjectProvider` implementation. If we could have true dynamic dispatch and take over delegation, we could mimic JavaScript's prototypal inheritance with each object having an actual prototype to which we could delegate messages and support overriding (prototypes with prototypes with prototypes with ...).

PowerShell, through `PSObject`, does support a single level of prototypal inheritance. I created an OSS project called [Prototye.ps][] in order to create a domain specific language for creating PowerShell objects using prototypal inheritance. There are a few simple operations we need in order to build a prototypal object factory:

1.  Wrap the prototype and add type metadata
2.  Update type metadata
3.  Add variables to the prototype
4.  Add functions to the prototype
5.  Attach a `ScriptProperty` to the prototype (In a later post)
6.  Apply Mixins (In a later post)

To create a new prototype object, we want to wrap an existing object in a `PSObject` so that we can add late bound functionality leveraging ATS. We also want to set up the infrastructure needed to support formatting ps1xml files via the `TypeNames` property.

``` powershell
function New-Prototype {
  param($baseObject = (new-object object))
  process {
    $prototype = [PSObject]::AsPSObject($baseObject)
    $prototype.PSObject.TypeNames.Insert(0,"Prototype")
    $prototype
  }
}
```

If a `PSObject` is passed into `New-Prototype`, the call to `[PSObject]::AsPSObject` will return that object rather than rewrap it. This allows use to extend prototypes using the various behaviors and mix them together without unneeded indirection.

Now that we have a handy function to create custom objects, we need the ability to append more type metadata easily when creating custom objects. Once we have these two tasks done, we can start to create our object factories.

``` powershell
function Update-TypeName {
  process {
    $caller = (Get-PSCallStack)[1].Command
    $caller = $caller -replace "new-", [string]::Empty
    $caller = $caller -replace "mixin-", [string]::Empty
    $derivedTypeName = $_.PSObject.TypeNames[0]
    if($derivedTypeName) {
      $derivedTypeName = "$derivedTypeName#{0}" -f $caller
    }
    $_.PSObject.TypeNames.Insert(0,"$derivedTypeName")
  }
}
```

You will notice that the code is actually in a `process {}` block. This is so that `Update-TypeName` will work in the pipeline. `Update-TypeName` assumes that the functions calling it follow the pattern `new-noun` or `mixin-noun` and then appends the calling members 'type' to the underlying `TypenName`. Now that we can create 'typed' prototypal object, let's take a look at a sample.

``` powershell
function New-Sample {
  $prototype = New-Prototype
  $prototype | Update-TypeName
  $prototype # always return the prototype
}

$sample = New-Sample
$sample | get-member

   TypeName: Prototype#Sample
[...]
```

That's a start, and we have something extendable, but not that useful yet. Next we need the ability to attach functions and properties/variables to our prototype.

``` powershell
function Add-Property {
  param(
    [string]$name, 
    [object]$value = $null,
    [System.Management.Automation.ScopedItemOptions]$options = [System.Management.Automation.ScopedItemOptions]::None,
    [Attribute[]]$attributes = $null
  )
  process {
    $variable = new-object System.Management.Automation.PSVariable $name, $value, $options, $attributes
    $property = new-object System.Management.Automation.PSVariableProperty $variable
    $_.psobject.properties.remove($name)
    $_.psobject.properties.add($property)
  }
}

function Add-Function {
  param(
    [string]$name,
    [scriptblock]$value
  )
  process {
    $method = new-object System.Management.Automation.PSScriptMethod "$name", $value
    $_.psobject.methods.remove($name)
    $_.psobject.methods.add($method)
  }
}
```

The implementations will detach any existing property/function/variable and attach the new implementation. You can attach private, readonly, and const variables/properties.

A more complex example might be a wrapper around the Microsoft Speech API (`SAPI`). I am going to define a message, a function to say messages, and update the type metadata so that I have a `Prototype#SapiVoice` object.

``` powershell
function New-SapiVoice {
  $prototype = New-Prototype
  $prototype | Update-TypeName
  $prototype | Add-Function Say {
    param([string]$message)
    $speaker = new-object -com SAPI.SpVoice
    ($speaker.Speak($message, 1)) | out-null
  }
  $prototype | Add-Property Message "Hello, World!"
  $prototype
}

$voice = New-SapiVoice
$voice.Say($voice.Message) # Says "Hello, World1"
$voice | Get-Member -View Extended # Show what we have added to the base object

   TypeName: Prototype#SapiVoice

Name    MemberType   Definition                         
----    ----------   ----------                         
Message NoteProperty System.String Message=Hello, World!
Say     ScriptMethod System.Object Say();
```

Following this pattern we can mix in modules, support multiple inheritance, and create reusable objects without having to pull in inline C# via `Add-Type`. This should look a bit like JavaScript ;)

  [Prototype-based programming]: http://en.wikipedia.org/wiki/Prototype-based_programming
  [Dynamic Language Runtime]: http://en.wikipedia.org/wiki/Dynamic_Language_Runtime
  [dynamic dispatch]: http://en.wikipedia.org/wiki/Dynamic_dispatch
  [Prototye.ps]: https://github.com/idavis/prototype.ps
  [Part 1]: /2012/08/prototypal-inheritance-using-powershell
  [Part 2]: /2012/08/prototypal-inheritance-using-powershell-part-two-scriptproperties
  [Part 3]: /2012/08/prototypal-inheritance-using-powershell-part-three-mixins
  [Part 4]: /2012/08/prototypal-inheritance-using-powershell-part-4-static-properties