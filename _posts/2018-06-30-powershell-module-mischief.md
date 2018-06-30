---
layout: post
title: "Powershell Module Mischief: Loki"
date: 2018-06-30 11:05
published: true
categories: powershell monkeypatching
---

> What would it look like if your command line updated itself and was context aware?

Thinking this through, we have a couple requirements
1. When moving into a directory or its children, you should be in the context of that directory.
2. When leaving a directory, the command line shouldn't leak directory specific behavior.
3. Given the mischief we are up to, we'll call this Loki.

To pull this off, we are going to do a couple of things. First, we need to proxy `Set-Location`.

{% highlight powershell %}
$setLocationMetadata =
    New-Object System.Management.Automation.CommandMetadata (Get-Command Set-Location)
[System.Management.Automation.ProxyCommand]::Create($setLocationMetadata)
    > Set-Location.Proxy.ps1
{% endhighlight %}

With this we can replace what happens every time we change directories, but we are going to keep is simple and just add a little hook. The code is abbreviated from `Set-Location.Proxy.ps1`:

{% highlight powershell %}
[CmdletBinding(DefaultParameterSetName='Default' <# more params #>)]
param( <# more params #> )

begin { <# more code #> }

process { <# more code #> }

end {
    try {
        $steppablePipeline.End()
    } catch {
        throw
    }

    Register-LokiFile <# <<======== THIS IS THE HOOK #>
}
<# more docs #>
{% endhighlight %}

Now in our profile, we need to add the `Register-LokiFile` functionality. A `.loki*` file will be the naming convention for custom code to load.

{% highlight powershell %}
# Register a global dictionary to store data.
$Global:__loki = @{}
 
function Register-LokiFile {
  # See if there are any .loki* files in our parent hierarchy
  function Get-LokiFile {
    $currentDir = Get-Location
    $lokiFiles = @(Resolve-Path (Join-Path $currentDir ".loki*")
                     -ErrorAction SilentlyContinue)
    if($lokiFiles.Length -eq 0) { return $null }
    return $lokiFiles[0]
  }
  
  $lokiFile = Get-LokiFile
  if(!$lokiFile) { return }

  # we have a loki file. Unload the current if one is loaded (and different)
  if($Global:__loki.current -and !($Global:__loki.current -eq $lokiFile)) {
    if($PSCmdlet.MyInvocation.BoundParameters["Debug"].IsPresent) {
      Write-Host "Removing loki config file: $($Global:__loki.current)"
    }
    Remove-Module -Name $Global:__loki.current -Force -ErrorAction SilentlyContinue
    $Global:__loki.current = $null
  }

  # set the current loki file
  $Global:__loki.current = "$lokiFile"
  if($PSCmdlet.MyInvocation.BoundParameters["Debug"].IsPresent) {
    Write-Host "Importing loki config file: $lokiFile"
  }

  # create a new, empty module
  $module = New-Module -Name "$lokiFile" -ScriptBlock {}

  # source the loki file into the new module's scope and monkey patch it
  . $module $lokiFile

  # import the module
  Import-Module $module
}

<# Finally set up our proxy function which registers the modules #>
. \Set-Location.Proxy.ps1
{% endhighlight %}

Now, as we move around in the command line, we can have custom functionality weave in and out with us.

The full repo is found at [idavis/loki](https://github.com/idavis/loki).
