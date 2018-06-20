---
layout: post
title: "Rubyesque Regex in PowerShell"
date: 2012-08-21 17:02
published: true
categories: powershell ruby regex
---

Ruby provides a handy mechanism for working with regular expressions. Using the `=~` operator, ruby will load the regex matches into global variables `$1, $2, $3, [...]`. For example:

``` rb
module Jekyll
  class GitHubTag < Liquid::Tag
    include HighlightCode
    include TemplateWrapper
    Pattern = /^(\w+)\s+(\w+)\s+(\w+)\s*(\.?\w+.?\w*)?\s*(http[s]*:\/\/\S+)?\s*(\S[\S\s]*)?$/i
    def initialize(tag_name, text, token)
      [...]
      @highlight = true
      if text =~ /\s*lang:(\w+)/i
        @filetype = $1
        text = text.sub(/lang:\w+/i,'')
      end
      if text =~ Pattern
        @user = $1
        @repo = $2
        @sha = $3
        @caption = $4
        @file = @caption
        @link = $5
        @linkTitle = $6
        [...]
      end

      if @file =~ /\S[\S\s]*\w+\.(\w+)/ && @filetype.nil?
        @filetype = $1
      end
    end
    [...]
  end
end
```

In PowerShell, it can be done a little better (and a little worse). Instead of relying on global variables, the variables can be loaded into the scope of the calling method. Unfortunately, PowerShell neither supports custom operators nor operator overloading. To create a similar usage, I can define a function named `=~` (yes, you can create crazy function names in PowerShell) that will take the source text in from the pipeline, and then the pattern to match as an argument giving us the syntax `$text | =~ $pattern`. The text may not match the string pattern; as such, the function must return `$true/$false` so that we know it is safe to use the automatic variables. We also need to deal with the case in which the variables already exist so that the variables are set rather redeclaring them.

``` powershell
function =~ {
  param([regex]$regex, [switch]$debug, [switch]$caseSensitive)
  process {
    $matches = $null # gets defined automatically when we do the matching
    $mached = $false
    if($caseSensitive) {
      $matched = $_ -cmatch $regex
    } else {
      $matched = $_ -match $regex
    }
    if(!$matched) { return $false }
    foreach( $key in $matches.keys) {
      $name = "$key"
      $value = 0
      $isInt = [int]::TryParse($name, [ref] $value)
      if(!$isInt -or $value -lt 1) { continue }
      
      $value = $matches[$key]
      $variableIsDeclared = @(Get-Variable $name -Scope 1 -EA SilentlyContinue).Length -gt 0
      if($variableIsDeclared) {
        if($debug) { write-host "setting $name to $value" }
        Set-Variable -Name $name -Value $value -Scope 1
      } else {
        if($debug) { write-host "making $name to $value" }
        New-Variable -Name $name -Value $value -Scope 1
      }
    }
    return $true
  }
}
```

As an example, I converted the regex part of my github extension for Octopress to PowerShell. It matches a user, repository, hash, file name, url, and title and populates six automatic variables.

``` powershell
$pattern = "^(\w+)\s+(\w+)\s+(\w+)\s*(\.?\w+.?\w*)?\s*(http[s]*:\/\/\S+)?\s*(\S[\S\s]*)?$"
$text = "idavis octopress aa676f90237c3941cc11f072fd74ab415a0c4022 github_tag.rb https://github.com/idavis/octopress/blob/master/plugins/github_tag.rb Current Version"

if($text | =~ $pattern) {
  Write-Host "Matched!"
  write-host "user = $1"
  write-host "repo = $2"
  write-host "sha = $3"
  write-host "caption = $4"
  write-host "file = $4"
  write-host "link = $5"
  write-host "linkTitle = $6"
}
```

Running the script above, we get the following output.

```
Matched!
user = idavis
repo = octopress
sha = aa676f90237c3941cc11f072fd74ab415a0c4022
caption = github_tag.rb
file = github_tag.rb
link = https://github.com/idavis/octopress/blob/master/plugins/github_tag.rb
linkTitle = Current Version
```

Now we have much easier regex usage in PowerShell inspired by ruby.