---
layout: post
title: "Prototypal Inheritance Using PowerShell (Part 4): Static Properties"
date: 2012-08-29 15:15
published: true
categories: powershell patterns
---
*This post is part of a four part series on prototypal inheritance using PowerShell.*
- [Part 1][] Introduction
- [Part 2][] ScriptProperties
- [Part 3][] Mixins
- [Part 4][] Static Properties

If you didn't read [Part 1][] and [Part 2][], I would recommend reading them as I build off of their functionality and theory.

In the previous articles I built a simple platform and API to help create PowerShell objects mimicking prototypal inheritance. Each object created with the [Prototype.ps][] API is essentially classless but instead has a loose specification and the actual 'class' of the object is held in its underlying `PSObject.TypeNames` member. We can leverage these type names while we are extending the underlying object to create proxy properties.

Why proxy properties? When adding a static property, we have a small problem. Due to the lack of control that we have for dispatching, we can't control how missing methods are handled, and thus we must define any methods that we want to be able to call. This also means that it does not make sense to attach static properties after object creation (outside of the object definition) as we cannot delegate the calls dynamically (any existing references would be missing the proxy property).

The Common Language Infrastructure (CLI) defines the Common Type System (CTS) which lays the groundwork for how the type system in .NET works. In the CTS definition, the runtime holds a single instance of a `Type` (e.g., there is one instance of the `Console` `Type` which should not be confused with an instances of the `Console` `class`); by mimicking this model, we can instantiate a shared 'type' object on which we can attach 'static' behavior and properties. Then, we can attach a proxy property on the instances which will call into this single instance.

The first step is to add a line to both the `New-Prototype` and `Update-TypeName` which will create the static instance if it doesn't exist.

``` powershell
function New-Prototype {
  param(
    $baseObject = (new-object object)
  )
  process {
    $prototype = [PSObject]::AsPSObject($baseObject)
    $prototype.PSObject.TypeNames.Insert(0,"Prototype")
    $prototype | Add-StaticInstance
    $prototype
  }
}

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
    $_ | Add-StaticInstance
  }
}
```

To simplify the implementation, I create a global registry that will match the `TypeName` to the shared instance. If the registry does not exist, then we create it. Once we have a registry, we need to get the current `TypeName` of the instance passed in from the pipeline. If the current `TypeName` has not been added to registry, we create a new `object` and wrap it in a `PSObject` which will allow us to extend the object and assign the new object's `TypeName` to that of the current pipe object. Once done, I add the new object into the global registry.

``` powershell
function Add-StaticInstance {
  process {
    # Create the static instance registry to mimic the CTS's single class instance per type
    if($Global:__CTS__ -eq $null) {
      $dictionary = (New-Object -TypeName 'System.Collections.Generic.Dictionary[string,object]' 
                                -ArgumentList @([StringComparer]::OrdinalIgnoreCase))
      $value = [PSObject]::AsPSObject($dictionary)
      $Global:__CTS__ = $value
    }
    $registry = $Global:__CTS__
    $key = $_.PSObject.TypeNames[0]

    # If this 'type' has not been added, create a new psobject and add it
    if(!$registry.ContainsKey($key)) {
      $instance = [PSObject]::AsPSObject((New-Object object))
      $instance.PSObject.TypeNames.Insert(0,$key)
      $registry[$key] = $instance
    }
  }
}
```

We now have an instance which is shared across our PowerShell session thiat is created the first time we instantiate our objects.

To implement `Add-StaticProperty`, I attach the desired property to the shared instance, and add a proxy `ScriptProperty` which calls into the registry, finds the instance that maps to our 'type' and accesses the property needed. As always, there is a small catch.

The `TypeName` is only known during the call to `Add-StaticProperty` and PowerShell doesn't support closures, so we can't capture the `TypeName` at this moment in a `ScriptBlock`. Instead, the `ScriptBlock` must be created from a string and the `TypeName` is captured via string interpolation. At the same time, we cannot let the rest of the proxy `ScriptBlock`s' values be interpolated (this leads to some syntactic messiness).

``` powershell
function Add-StaticProperty {
  param(
    [string]$name, 
    [object]$value = $null,
    [System.Management.Automation.ScopedItemOptions]$options = [System.Management.Automation.ScopedItemOptions]::None,
    [Attribute[]]$attributes = $null
  )
  process {
    $key = $_.PSObject.TypeNames[0]

    $instance = $Global:__CTS__[$key]
    $instance | Add-Property $name $value $options $attributes

    $getterBlock = [ScriptBlock]::Create('$Global:__CTS__["' + "$key" + '"].' + "$name")
    $setterBlock = [ScriptBlock]::Create('param($value) $Global:__CTS__["' + "$key" + '"].' + "$name" + ' = $value')
    $_ | Add-ScriptProperty $name $getterBlock $setterBlock
  }
}
```

Enough background, let's look at an example. Borrowing the example from [Part 1][], I am making the `Message` static so that it can be modified and used by separate instances.

``` powershell
function New-SapiVoice {
  $prototype = New-Prototype
  $prototype | Update-TypeName
  $prototype | Add-Function Say {
    param([string]$message)
    $speaker = new-object -com SAPI.SpVoice
    ($speaker.Speak($message, 1)) | out-null
  }
  $prototype | Add-StaticProperty Message "Hello, World!"
  $prototype
}

$voice = New-SapiVoice
$voice.Say($voice.Message) # says: Hello, World!

# Changing the message on $voice changes it on $voice2 as well
$voice.Message = "The dude abides"

$voice2 = New-SapiVoice
$voice2.Say($voice2.Message) # says: The dude abides
```

Because of the way the shared instances are mapped with the proxy properties, derived objects are able to call static methods on their base implementations. 

``` powershell
function New-Foo {
  $prototype = New-Prototype
  $prototype | Update-TypeName
  $prototype | Add-StaticProperty Name "Baz"
  $prototype
}

function New-Bar {
  $prototype = New-Foo
  $prototype | Update-TypeName
  $prototype
}

$foo = New-Foo
$bar = New-Bar

$foo.Name # outputs Baz
$bar.Name # outputs Baz

$foo.Name = "The Dude"

$foo.Name # outputs The Dude
$bar.Name # outputs The Dude
```


If you want to view the 'inheritance hierarchy' of your prototypal object, you can call `$value.PSObject.TypeNames` and it will give you an ordered collection of types being implemented. You can also do this on any .NET object that you wrap in a `PSObject` instance.

I hope you enjoyed this trip down the rabbit hole.

  [Part 1]: /2012/08/prototypal-inheritance-using-powershell
  [Part 2]: /2012/08/prototypal-inheritance-using-powershell-part-two-scriptproperties
  [Part 3]: /2012/08/prototypal-inheritance-using-powershell-part-three-mixins
  [Part 4]: /2012/08/prototypal-inheritance-using-powershell-part-4-static-properties
  [Prototype.ps]: https://github.com/idavis/prototype.ps