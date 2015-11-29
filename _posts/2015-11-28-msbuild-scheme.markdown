---
layout:	post
title:	"A Scheme for Projects MS Build"
date:	2015-11-28 15:32:21 +0200
categories:	Development
tags:	[development, build, ms-build]
---

#  Introduction

This is the scheme I've used (with some variations) to set up a build with MS Build on my .NET Projects.

The scheme features:
-  Automatic inclusion of all targets into the build.
-  Management of dependencies on a per-target basis.
-  Adding new projects to your build simply means creating the targets file (almost) no edits.
-  Scripted template for targets.
-  Build arbitrary projects and it's dependencies
-  Option to build only an arbitrary project and not it's dependencies
-  Configuration and Platform propagation into project builds

#  File Tree

Here is a file tree for our project

~~~
|-- root
|   |-- build.proj
|   |-- createTargets.bat
|   |-- SomeProject
|   |   |-- SomeProject.csproj
|   |   |-- SomeProject.targets
|   |   |-- ...
|   |-- SomeLibrary
|   |   |-- SomeLibrary.csproj
|   |   |-- SomeLibrary.targets
|   |   |-- ...
~~~

#  Root File: build.proj

This is the file that binds everything together. It defines the `BuildAll` and `CleanAll` targets and imports everything.

~~~ XML
<?xml version="1.0" encoding="utf-8"?>
<Project
         ToolsVersion="4.0"
         xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

    <!-- Defaults -->
    <PropertyGroup>
        <Platform Configuration=" '$(Configuration)' == '' ">Release</Platform>
        <Platform Condition=" '$(Platform)' == '' ">AnyCPU</Platform>
    </PropertyGroup>
    
    <!-- Import all targets -->
    <Import Project=".\**\*.targets" />
   
    
    <!-- Top Level Build Targets -->
    <Target Name="BuildAll" DependsOnTargets="@(BuildAllDependsOn)">
    </Target>
    
    <Target Name="CleanAll" DependsOnTargets="@(CleanAllDependsOn)">
    </Target>
</Project>
~~~

#  Target files: SomeProject.targets

Your .targets files should contain targets to build individual modules of your system. In these files we define what to build, and what it's dependencies are. These files should live next to your .csproj (or vbproj, etc) files and you can even add them to your solutions so that they are easy to edit. The example below is for a would be `SomeProject.csproj`.

``` xml
<?xml version="1.0" encoding="utf-8"?>
<Project
         ToolsVersion="4.0"
         xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

    <!-- List Dependencies here -->
    <PropertyGroup>
        <SomeProjectDependsOn Condition=" '$(ExplicitTargetsOnly)' != 'true' ">
            SomeLibrary
        </SomeProjectDependsOn>
    </PropertyGroup>

    <!-- Hook in to top level targets here -->
    <ItemGroup>
        <CleanAllDependsOn Include="CleanSomeProject" />
        <BuildAllDependsOn Include="SomeProject"/>
    </ItemGroup>

    <!-- Projects to Build -->
    <ItemGroup>
        <SomeProjectProjectsToBuild Include=".\SomeProject\SomeProject.csproj">
            <Properties>Configuration=$(Configuration);Platform=$(Platform)</Properties>
        </SomeProjectProjectsToBuild>
    </ItemGroup>

    <!-- Build Targets -->
    <Target Name="SomeProject"  DependsOnTargets="$(SomeProjectDependsOn)">
        <MSBuild Projects="@(SomeProjectProjectsToBuild)" />
    </Target>

    <Target Name="CleanSomeProject">
        <MSBuild Projects="@(SomeProjectProjectsToBuild)" Targets="Clean" />
    </Target>

</Project>
```


**Project Name**
The convention of the project file's name carries through (CleanSomeProject, SomeProjectDependsOn, etc) to the target name and should ideally be the assembly name as well

**BuildAll and CleanAll**
Note how the CleanAllDependsOn and BuildAllDependsOn are defined and include the targets. This is how the targets get added to the respective root targets.

**Dependencies**
To set your depencies, you can simply include the targets you want built first in the SomeProjectDependsOn property. As per MSBuild properties, these should be separated with a `;`

**Configuration and Platform**
These are injected into the project based on what is defined at build, and default to your Release|AnyCPU.

**Relative Paths**
The path of the csproj you're building must be relative to `build.proj`

**ExplicitTargetsOnly**
Sometimes you want to look at why only a specific project isn't building and not spam your console with the 50 dependencies. Set this property to true at runtime (`/p:ExplicitTargetsOnly=true`) to ignore dependencies.

#  Running a Build

You can build all by going: `msbuild build.proj /t:buildall`

You can also make this the default target.

#  CreateTargets.bat

I have a script which generates a .targets file for my project based on convention for the project.

From root, simply: `createTargets .\SomeProject\SomeProject.csproj`

Check out my [Github](https://github.com/geoffles/geoffles-msbuild-scheme) for a sample repo.