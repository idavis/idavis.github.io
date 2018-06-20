---
layout: post
title: "Creating xargs for PowerShell"
date: 2012-08-20 21:43
published: true
categories: powershell cli gems
---
While working in Ubuntu, I don't really have a hard time dealing with gems and ruby thanks to rvm. On windows, however, it is another story. Many times I have needed to reset my installed gems by uninstalling them all. Ken Nordquist has a great article on how to [remove all ruby gems][] on linux. Being primarily a windows developer, I do most of my CLI work in PowerShell. 

PowerShell has a built-in `xargs` functionality, but it doesn't work with non-powershell commands like rake.

``` powershell
$gems = gem list

#make a dir for each gem
$gems | mkdir -path { $_ }

#remove each directory we just created
$gems | rmdir -Include { $_ }
```

We pipe the lines from `$gems` into invocations of `mkdir/rmdir`, binding the `path/include` parameter to the `$_` expression on each invocation. You have to use this bulky syntax and it must be bound to parameters. For example, `$gems | mkdir { $_ }`, `$gems | mkdir -path $_`, and `$gems | mkdir $_` will not work. To start, let's look at a simple, straightforward, implementation to remove all gems.

``` powershell
# get a list of our gems in "name (version)" format
gem list
#[...]
jekyll (0.11.2)
json (1.7.5)
#[...]

# With this list, we can now take the left side splitting on a space character to get the gem names
# The split call with no parameters will automatically spit on spaces.
gem list | % { $_.split()[0] }

# We can then take those gem names as pipeline input into a foreach-object block
gem list | % { $_.split()[0] } | % { gem uninstall -aIx $_ }
```

This leaves something to be desired; too many times in PowerShell we have to do `% { xxx yyy $_ }` instead of `xargs xxx yyy`. This may seem small, but it is very annoying to deal with the additional syntax `% { $_ }`. Adding a (simplified) `xargs` implementation to PowerShell is relatively easy with a filter.

``` powershell
filter xargs { invoke-expression "$args $_" }
```

The `xargs` filter passes all parameters into an expression string and concatenates the pipe value. The problem with this approach is that the we are following the paradigm of passing strings through the pipe instead of objects. To use objects in our pipeline, we need to execute the command itself with its arguments and concatenating the pipe object.

``` powershell
filter xargs { & $args[0] ($args[1..$args.length] + $_) }
```

Now with a proper `xargs` implementation, we can rewrite the original command using our new `xargs` filter.

``` powershell
gem list | % { $_.split()[0] } | xargs gem uninstall -aIx
```

 [remove all ruby gems]: http://geekystuff.net/2009/01/14/remove-all-ruby-gems/