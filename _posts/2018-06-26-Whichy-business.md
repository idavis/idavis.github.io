---
layout: post
title: "(Which)y Business"
date: 2018-06-26 6:25
published: false
categories: powershell
---

{% highlight powershell %}
function which {
  [CmdletBinding()]
  param([string]$name, [Parameter(Mandatory=$false)][ValidateSet('code','npp','vim')][string]$editor = "")
  $commands = @(get-command $name -ErrorAction SilentlyContinue)

  if($commands.Length -eq 0) {
    Write-Error "Unknown command or alias $name."
  }

  if($commands.Length -gt 1) {
    Write-Error "Ambiguous command or alias $name. Found:"
    $commands | % { Write-Error "`t$($_)" }
  }
  function Write-NonVerbose {
    if(!$verbose) {
      Write-Host $cmd.Definition
    }
  }
  $cmd = $commands[0]
  $verbose = $PSBoundParameters['Verbose'] -eq $true
  switch($cmd.CommandType) {
    "Alias" {
      Write-Host "$($cmd.CommandType):`t$($cmd.Definition)`n"
      which "$($cmd.Definition)" -Verbose:$verbose
    }
    "Cmdlet" {
      Write-NonVerbose
      Write-Verbose "`n$($cmd.Definition)`nImplemented by $($cmd.ImplementingType)`nDefined in $($cmd.DLL)"
    }
    "Function" {
      Write-NonVerbose
      $block = $cmd.ScriptBlock
      if($block.Module) {
        Write-Verbose "`nIn $($block.Module.Path)`nIn $($block.File) line(s) $($block.StartPosition.StartLine)-$($block.StartPosition.EndLine):`n`n$($block.StartPosition.Content)"
      } else {
        Write-Verbose "`nIn $($block.File) line(s) $($block.StartPosition.StartLine)-$($block.StartPosition.EndLine):`n`n$($block.StartPosition.Content)"
      }
      switch ($editor) {
        "code" { & code --goto "`"$($block.File)`":$($block.StartPosition.StartLine):$($block.StartPosition.StartColumn)" }
        "npp"    { & npp "$($block.File)" -n"$($block.StartPosition.StartLine)" }
        "vim"    { & vim "+$($block.StartPosition.StartLine)" "$($block.File)" }
        Default {}
      }
    }
    "Application" {
      $extension = [System.IO.Path]::GetExtension($cmd.Definition)
      Write-Host $cmd.Definition
      function Open([string]$fileName) {
        switch ($editor) {
          "code" { & code --goto "$fileName" }
          "npp"    { & npp "$fileName" }
          "vim"    { & vim "$fileName" }
          Default {}
        }
      }
      switch($extension) {
        ".cmd" {
          Write-Verbose ("`n" + (Get-Content $cmd.Definition -Raw))
          Open $cmd.Definition
        }
        ".bat" {
          Write-Verbose ("`n" + (Get-Content $cmd.Definition -Raw))
          Open $cmd.Definition
        }
        default {
          # Don't know what to do for other extensions.
        }
      }
    }
    default {
      Write-Host $cmd.Definition
    }
  }
}

{% endhighlight %}
