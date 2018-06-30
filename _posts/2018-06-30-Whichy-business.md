---
layout: post
title: "(Which)y Business"
date: 2018-06-30 6:25
published: true
categories: powershell
---
Inspired by the `which` command, I sought out to solve a few problems:

1. Determine the source of a command
    ```
    > which git
    C:\Program Files\Git\cmd\git.exe
    ```
2. Resolve the source of aliases (including nested aliases)
    ```
    > Set-Alias a b
    > Set-Alias b c
    > Set-Alias c d
    > Set-Alias d cd
    > which a

    Alias:  b
    Alias:  c
    Alias:  d
    Alias:  cd
    Alias:  Set-Location

    Set-Location [[-Path] <string>] [-PassThru] [-UseTransaction] [<CommonParameters>]
    Set-Location -LiteralPath <string> [-PassThru] [-UseTransaction] [<CommonParameters>]
    Set-Location [-PassThru] [-StackName <string>] [-UseTransaction] [<CommonParameters>]
    ```
3. Provide detailed information for commands
    ```
    > which Set-Location -v
    VERBOSE:

    Set-Location [[-Path] <string>] [-PassThru] [-UseTransaction] [<CommonParameters>]
    Set-Location -LiteralPath <string> [-PassThru] [-UseTransaction] [<CommonParameters>]
    Set-Location [-PassThru] [-StackName <string>] [-UseTransaction] [<CommonParameters>]
    Implemented by Microsoft.PowerShell.Commands.SetLocationCommand
    Defined in
    C:\WINDOWS\Microsoft.Net\assembly\GAC_MSIL\Microsoft.PowerShell.Commands.Management\
      v4.0_3.0.0.0__31bf3856ad364e35\Microsoft.PowerShell.Commands.Management.dll
    ```
4. Supply detailed information about the implementation of commands which I can edit.
    ```
    > which bundler
    C:\Ruby24-x64\bin\bundler.bat

    > which bundler -v
    C:\Ruby24-x64\bin\bundler.bat
    VERBOSE:
    @ECHO OFF
    IF NOT "%~f0" == "~f0" GOTO :WinNT
    @"C:\Ruby24-x64\bin\ruby.exe" "C:/Ruby24-x64/bin/bundler" %1 %2 %3 %4 %5 %6 %7 %8 %9
    GOTO :EOF
    :WinNT
    @"C:\Ruby24-x64\bin\ruby.exe" "%~dpn0" %*
    ```
5. For powershell functions, tell me exactly where they are implemented (file, lines, modules).
    ```
    >which which -v
    VERBOSE:
    In C:\Users\<user>\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1 line(s) 29-102:

    function which { ... }
    ```
6. For editable scripts, allow them to be openend in a text editor
    ```
    > which which -editor vim
    # opens my profile in Vim to line 29
    ```

The implementation is recursive and relatively simple. Current version is on [gist.github.com/idavis](https://gist.github.com/idavis/07a7b98b67b9e5dce0795d5dc1c7d2f7). This version supports Vim and VS Code.

{% highlight powershell %}
function which {
  [CmdletBinding()]
  param([string]$name, [Parameter(Mandatory=$false)][ValidateSet('code','vim')][string]$editor = "")
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
      $content = $block.StartPosition.Content
      $start = $block.StartPosition.StartLine
      $end = $block.StartPosition.EndLine
      
      if($block.Module) {
        Write-Verbose "`nIn $($block.Module.Path)`nIn $($block.File) line(s) $($start)-$($end):`n`n$($content)"
      } else {
        Write-Verbose "`nIn $($block.File) line(s) $($start)-$($end):`n`n$($content)"
      }
      switch ($editor) {
        "code"   { & code --goto "`"$($block.File)`":$($start):$($block.StartPosition.StartColumn)" }
        "vim"    { & vim "+$($block.StartPosition.StartLine)" "$($block.File)" }
        Default  { }
      }
    }
    "Application" {
      $extension = [System.IO.Path]::GetExtension($cmd.Definition)
      Write-Host $cmd.Definition
      function Open([string]$fileName) {
        switch ($editor) {
          "code"   { & code --goto "$fileName" }
          "vim"    { & vim "$fileName" }
          Default  { }
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
