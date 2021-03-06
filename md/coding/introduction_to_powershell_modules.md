# Introduction to Powershell Modules

Let's say we write a simple function, and then want to use it throughout a package, or just anywhere on our local machine. How do we make such a function available in a way that's simple and sustainable? 

Answer: you make modules. 

How do we then do the same for the modules? I.e.: automatically loading in another module, when loading in a module that requires it? Answer: module manifests.

In this tutorial I'll describe how to go from running a function to make it accessible, to creating modules that import other modules.

## How you might do it without modules
Take the function below. Note that it's very simple. 

> When you're planning on starting a big project, it's always useful to start with basic wrapper functions for often used actions. Now, if we want to change the way we log later on, instead of refactoring all our scripts, we only need to change the wrapper function below.

``` powershell
# .\writelog.ps1

Function Write-Log()
{
	Param($Message, $LogFilePath)
	
	$message >> $LogFilePath
}
```
Taking that it's a wrapper function that basically extends on Write-Host, and will be used everywhere in our codebase, we'll need to find a way to make it accesible everywhere.

If we want to have it automatically available in another script, and not have to separately run writelog.ps1 beforehand,
we could resort to dot sourcing:

``` powershell
# .\anotherScript.ps1
. writelog.ps1
Write-Log -Message "Hi" -LogFilePath "C:/log.txt"
```
This method is really useful for breaking up a large script into several parts. All the scripts will then be in the same folder and will always be at the same location relative to eachother. If you want to create a finer structure later on though, like separating the functions and config files into separate folders, you'll need to update the filepath in all the scripts where dot sourcing has been used. Not very flexible.

It would also be useful to have one logging solution being available anytime we run code, even if it's just in the CLI. Maybe even have it possible to easily export this code to another machine and make it available globally there too.


## A simple module
There really isn't much to making this happen. First, we change the filename extension of our previous file from .ps1 to .psm1.

Then, we put this module file into a folder that is listed in `$PSModulePath`. This would by default be 
`$home\Documents\WindowsPowerShell\Modules\` (to make this accessible for only yourself), and `$pshome\Modules` to make the module accessible for everyone on the machine. 

Note that you can always add extra folders to the $PSModulePath. 

The .psm1 file has to be in a folder of the same name. That folder has to be directly in one of the filepaths listed in $PSModulePath.

In this example the path will be:

```
C:\Users\<username>\Documents\WindowsPowerShell\Modules\WriteLog\WriteLog.psm1
```

Now we can use the Write-Log function in any script by doing the following:

``` powershell
# .\aBetterSolution.ps1
Import-Module WriteLog
Write-Log -Message "Hi" -LogFilePath "C:/log.txt"
```
## Expanding on this
You can now add extra functions to WriteLog.psm1. Any function in there will be loaded when the module is loaded. This way, it's very easy to expand your logging options by just adding extra functions to the module that you've already created.

Installing the module on another computer is as simple as just copying the folder over to that computer's $PSModulePath.

Note that any module in the $PSModulePath will be automatically loaded into your CLI, you don't need to run Import-Module. 

A script should always include the `Import-Module <ModuleName>` line though, to make it clear that this module is required for the script to run, even if the module is loaded by default on your powershell profile (as it might not be on others!).

## A Root Module (aka Module Manifest)
Instead of putting every kind of custom function in one module, you might want to split your module up into different packages.

For example, you might want to put all your logging functionality in one package, and have a general package that does a lot
of custom things use that module by default. These packages can be called submodules, or better known as NestedModules.

So, let's say we create a new module, called __InfraManager__, that brings a lot of custom functions with it, like connecting to storage solutions, creating VM's, etc. We'll also rename WriteLog into __Logger__ to make it more clear that this module is a general logger, not just a way to dynamically load the Write-Log function.  

Let's assume the following: The Logger module will have to be loaded whenever InfraManager is loaded, 
as InfraManager uses functions from Logger throughout its own functions. One way to do this is to simply add `Import-Module Logger` at the top of InfraManager.psm1. 

This will only work for fully fledged
modules though, i.e. modules that are installed directly on the $PSModulePath. In my case, I want to split up InfraManager into a lot
of different submodules, but I don't necessarily want to install them as full modules yet. 

I want to be able to install just 
InfraManager, and have the code segmented into submodules that will be loaded in with it (as it's a structural part of the larger module as a whole). We can then split off the submodules into their own fully fledged modules
when they are mature enough (and have a usecase outside of InfraManager, like a universal logging tool would).

We created WriteLog as a standalone module, in the next example we'll install the same module (renamed to Logger) as a submodule of InfraManager (the root module).

Let's build a quick InfraManager.psm1 file:

``` powershell
# $home\Documents\WindowsPowerShell\Modules\InfraManager\Inframanager.psm1

Function Test-Logging()
{
	Write-Log "testest" -LogFilePath "C:/log.txt"
}
```
We place InfraManager.psm1 in a folder, called InfraManager, and place this folder in a $PSModulePath folder, i.e.: `$home\Documents\WindowsPowerShell\Modules\` 

In the InfraManager folder, we create a folder called Modules, in which we'll place the WriteLog module folder (again, renamed to Logger), We thus get:

```
-- $home\Documents\WindowsPowerShell\Modules\
-- -- \InfraManager\
-- -- -- \Modules\
-- -- -- -- \Logger\
-- -- -- -- -- \Logger.psm1
-- -- -- \InfraManager.psm1
```

### Creating the module manifest
Run the following command in powershell:

``` powershell
New-ModuleManifest -Path "$home\Documents\WindowsPowerShell\Modules\InfraManager\InfraManager.psd1"
```

This will create a full manifest for you. You can look here for information on the extra options in that file:
[How to write a powershell manifest (docs.microsoft)](https://docs.microsoft.com/en-us/powershell/developer/module/how-to-write-a-powershell-module-manifest).

For now, we will be concerned mostly with RequiredModules, NestedModules, and FunctionsToExport.

The comments in the file are pretty self explanatory. If we want to preload modules that are not (directly) in the $PSModulePath
when we load the InfraManager module we can add the following to the NestedModules list:
``` powershell
NestedModules     = @(  'Modules\Logger\Logger.psm1' )
```
Notice that this path is relative to $PSScriptRoot.

Now, there is a choice to be made. You can specify the FunctionsToExport variable. Microsoft docs state that:

> FunctionsToExport specifies the functions that the module exports (wildcard characters are permitted) to the caller's session state. By default, all functions are exported. You can use this key to restrict the functions that are exported by the module.

This is thus a way to have internal functions (that only serve to help the cmdlets that the user uses) not show up after importing the module. If you want this, then you should specify FunctionsToExport.

For me, FunctionsToExport looked like this in my freshly generated .psd1:

``` powershell
FunctionsToExport = @()
```

This lead to some headscratching as no functions were exported, while I didn't change the default setting. What the documentation mean to say is that:

``` powershell
#FunctionsToExport = @()
```
Will lead to all functions being exported. If you leave the default setting in the .psd1 file, no functions will be exported as FunctionsToExport is defined as an empty list.


#### Exporting submodules as proper modules
If we later want to change Logger to a full fledged module, we can copy the Logger folder to be directy under `C:\Users\<username>\Documents\WindowsPowerShell\Modules\`, 
and remove the item from NestedModules, and change RequiredModules:

``` powershell
RequiredModules = @('Logger')
```

### Conclusion: Why use manifests in our case
- You can use NestedModules. This allows you to separate out code into modules without:
	* Polluting the module path with half baked modules
	* Having to install a lot of modules for one solution, just copy over one folder
- Having two modules requiring eachother without going into an Import-Module loop.

Let me explain the last point a bit, because I haven't talked about it yet: Let's say WriteLog uses 
functions in InfraManager (and we don't care because both will always be installed on _our_ machines), and InfraManager uses
functions from WriteLog. You might want to add `Import-Module <the other one>` to each module, but this will result in an import
loop. You can solve this by adding:

``` powershell
If (! (Get-Module <the other module>))
{ Import-Module <the other module> }
```

It was still kind of buggy for me, but I'm sure you can make the above code work in some form or another.
Point is: manifest files are a very clean way to document all the info on your module in one location. If you're going through
the trouble of building a Root module, you might as well make the 30 minutes investment of building a manifest file.						

