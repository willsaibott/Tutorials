# Publishing and Importing Native C++ Packages into Local Feeds using NuGet

### Add a *PackageName*.nuspec file:

```xml
<?xml version="1.0" encoding="utf-8"?>
<package xmlns="http://schemas.microsoft.com/packaging/2010/07/nuspec.xsd">
    <metadata>
        <id>PackageName</id>
        <version>0.0.0.1</version>
        <description>Package description</description>
        <tags>native</tags>
        <authors>Author</authors>
        <!-- Optional Fields -->
        <icon>images/icon.png</icon>
        <projectUrl></projectUrl>
        <repository type="git" url="" branch="master" commit=""/>
        <!-- Visual Studio requires it for native C++ packages -->
        <dependencies>
            <group targetFramework="native0.0" />
        </dependencies>

    </metadata>
     
    <files>
        <!-- Apparently there must be a "native" folder, so VS can understand these are C++ project files, if no native folder is given, the restore is rejected -->
        <file src="src\*.h"          target="native\lib"     />
        <!-- <file src="src\*.cpp"          target="native\lib"     /> -->
        <file src="bin\*.lib"          target="native\bin"     />
        <file src="docs\*"               target="docs\"    />
        <file src="images\icon.png"      target="images\"    />
        <!-- PackageName.targets must be in build directory on target and match the package name -->
        <file src="PackageName.targets"      target="build\"    />
    </files>
</package>
```

* Override PackageName with your package name
* The targets file must be placed in the **build** directory in the *target* machine and it's name match the **metadata.id** field value. Ex:
    ```xml
    <metadata>
        <id>Lib.Advanced</id>
        <version>0.0.0.1</version>
        ...

    </metadata>
     
    <files>        
        <file src="Lib.Advanced.targets"      target="build\"    />
    </files>
    ```
* The binaries, headers and implementation files, must be placed in a path that contains "native" in the target. Visual Studio has a routine that rejects the restore of the package if the native directory doesn't have files in.
* Do not forget the **metadata.dependencies** field with "\<group targetFramework="native0.0" /\>" as value. Visual studio will reject the package if you do.
* Pay attention to the comments

### Add a *PackageName*.target file

This files will be imported by visual studio, and contain the properties and includes the package will have:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003" ToolsVersion="15.0">
  <PropertyGroup>
    <LibraryType Condition="'$(Configuration)'=='Debug'">mtd</LibraryType>
    <LibraryType Condition="'$(Configuration)'=='Release'">mt</LibraryType>
  </PropertyGroup>
  <PackageLibrariesPath Include="$(MSBuildThisFileDirectory)\native\bin\$(platform)\*.lib" />
  <PropertyGroup>
    <!-- Expland the items to a property -->
    <PackageLibraries>@(PackageLibrariesPath)</PackageLibraries>
    </PropertyGroup>
    <ItemDefinitionGroup>
    <ClCompile>   
        <!-- This is the most important: -->
        <AdditionalIncludeDirectories>$(MSBuildThisFileDirectory)..\native\lib;%(AdditionalIncludeDirectories)</AdditionalIncludeDirectories>
    </ClCompile>
    <Link>
      <AdditionalDependencies>$(PackageLibraries);%(AdditionalDependencies) 
      </AdditionalDependencies>
    </Link>
  </ItemDefinitionGroup>
</Project>
```

*MSBuildThisFileDirectory* variable is the directory of the *PackageName*.targets that will be in $(SolutionDir)packages/*PackageName.Version*/build/*PackageName*.targets.

* You must pass the include directory path for your packages in this file (relative to *MSBuildThisFileDirectory*):

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003" ToolsVersion="15.0">
<PropertyGroup>
    <ClCompile>
        <AdditionalIncludeDirectories>$(MSBuildThisFileDirectory)..\native\lib%(AdditionalIncludeDirectories)</AdditionalIncludeDirectories>
    </ClCompile>
</PropertyGroup>
</Project>
```

* If you have *.lib, or *.dll binaries generated, you must provide the paths and include them as link dependencies:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003" ToolsVersion="15.0">
  <PackageLibrariesPath Include="$(MSBuildThisFileDirectory)\native\bin\$(platform)\*.lib" />
  <PropertyGroup>
    <!-- Expland the items to a property -->
     <PackageLibraries>@(PackageLibrariesPath)</PackageLibraries>
  </PropertyGroup>
    <ItemDefinitionGroup>
    ...
    <Link>
      <AdditionalDependencies>$(PackageLibraries);%(AdditionalDependencies)</AdditionalDependencies>
    </Link>
    ...
```


##### Create The Local Nuget Feed.

###### In TFS:

1. Open Project>Build & Realase>Packages
2. Install the Project Management Plugin if itsn't installed yet.
3. Add a new Feed
4. Give a name to your Local NuGet Feed

###### In Azure DevOps:




##### Command to add a NuGet Local Feed reference to your machine:

1. Generate a PAT (Personal Access Token) in TFS or Azure DevOps, copy it and save it in a save place.
2. Type:

```bash
$>  nuget sources add -Name <SourceName> -Source <SourceURL> -username <UserName> -password <PAT>
```

##### Command to pack:

```bash
$>  NuGet.exe pack PackageName.nuspec -OutputDirectory "Output Directory Path"
```

Example:

```bash
$>  NuGet.exe pack lib.http.nuspec -OutputDirectory export\
```

##### Command to publish:

```bash
$>  NuGet.exe push -Source *"Nuget Feed Name"* -ApiKey *API Key* *PackageName*.nupkg
```

Example:

```bash
$> NuGet.exe push -Source "LocalNugetFeed" -ApiKey VSTS export\\lib.http.nupkg
```

##### Import in Visual Studio

1. Open Tools->NuGet Package Manager->Package Manager Settings->Package Sources
    * Add the Local News Feed if not yet.
    * click in OK
2. Open Tools->NuGet Package Manager->Manage NuGet Packages for Solution...
3. Browse your package and install it in the desired project.
4. Test if you can include files of the package in a header or implementation file of your project.

You may need to clean the NuGet Package cash, if you can't see the most recent version of your package:

1. Open Tools->NuGet Package Manager->Package Manager Settings->General
2. click in "Clear All NuGet Cache(s)"


##### Install using Nuget CLI

```bash
$> Install-Package -Source "Nuget Feed Name" PackageName -version *Version*
```

Example:

```cmd
$> Install-Package -Source "LocalNugetFeed" Lib.Http -version 0.0.0.9
```