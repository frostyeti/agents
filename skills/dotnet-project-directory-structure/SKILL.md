---
name: dotnet-project-directory-structure
description:
  Best practices for .NET application project structure at a high level including common folder structures and common dotnet build related files like Directory.Build.props and csproj files.
license: MIT
---

# .NET Project Structure Best Practices


## When to Use this Skill

Use cases:

- Creating a new .NET solution from scratch
- Refactoring an existing solution for better organization and maintainability
- Adding new projects to an existing solution and deciding where they fit
- Establishing conventions for a team or organization on how to structure .NET projects
- Creating a new repository or project and using .NET for development


## Common Structures

### Application Layout

```
MyApp/
|   eng/                       # Build scripts, CI/CD configs, and other engineering resources
|   ├──/scripts/                   # Build and maintenance scripts
├── src/
│   ├── MyApp.Domain/           # Core domain logic, entities, interfaces
│   ├── MyApp.Application/      # Use cases, application services, DTOs
│   ├── MyApp.Infrastructure/   # Data access, external services, implementations
│   └── MyApp.Api/              # ASP.NET Core API or presentation layer
├── tests/
│   ├── MyApp.Domain.Tests/
│   ├── MyApp.Application.Tests/
│   └── MyApp.Api.Tests/
├── .gitignore
├── .editorconfig               # store code style settings 
├── castfile                    # Cast task runner configuration
├── Directory.Build.props       # centralized properties.
└── Directory.Packages.props    # Central package management
└── mise.toml                   # Centralized tool version management
├── MyApp.slnx
└── LICENSE                     # License information
└── README.md                   # Project overview and documentation
```

### Small Library Layout

```
eng/
src/
test/
.editorconfig
.gitignore
castfile
Directory.Build.props
Directory.Packages.props
LICENSE
mise.toml
MyLibrary.slnx
README.md
```

### Library Monorepo Layout

This is for multiple related libraries that are developed together and shared repository. web is interchangeable 
with other specific domains such as data, winforms, wpf, etc.

For bcl libraries, they will be released as one version.

For libraries under lib, they will be released separately with their own versioning and will most likely need
separate pipelines.

For web libraries, that will be determined by the README.md or AGENTS.md in the folder. 
If there are multiple web libraries, they should be split into separate folders with their own README.md and AGENTS.md to clarify their purpose and usage.

```
eng/                               # Build scripts, CI/CD configs, and other engineering resources
bcl/                               # Base class libraries
│   ├── Org.LibraryA/              # Library with no dependencies on other libraries
│   │   ├── src/
│   │   │   └── Org.LibraryA.csproj
│   │   ├── tests/
│   │   │   └── Org.LibraryA.Tests.csproj
│   │   └── libraryA.slnx
│   ├── Org.LibraryB/              # Library that depends on Org.LibraryA
│   │   ├── src/
│   │   │   └── Org.LibraryB.csproj
│   │   ├── tests/
│   │   │   └── Org.LibraryB.Tests.csproj
│   │   └── libraryB.slnx
│   ├── bcl.slnx                   # Solution referencing all bcl libraries
│   └── Directory.Build.props      # Shared build props for bcl libraries
lib/                                # Higher level libraries depending on bcl
│   ├── Org.LibraryForHelmet/
│   │   ├── src/
│   │   ├── tests/
│   │   └── libraryForHelmet.slnx
│   ├── Org.LibraryForKeePass/
│   │   ├── src/
│   │   ├── tests/
│   │   └── libraryForKeePass.slnx
│   ├── lib.slnx                   # Solution referencing all lib libraries
│   └── Directory.Build.props
web/                                # Libraries for ASP.NET Core web projects
│   ├── Org.Web.LibraryA/
│   │   ├── src/
│   │   ├── tests/
│   │   └── libraryA.slnx
│   ├── Org.Web.LibraryB/
│   │   ├── src/
│   │   ├── tests/
│   │   └── libraryB.slnx
│   ├── web.slnx
│   └── Directory.Build.props
.editorconfig
.gitignore
all.slnx                            # Optional solution referencing all projects for easy development
castfile
Directory.Build.props                  # Shared build props for all projects
Directory.Packages.props               # Central package management for all projects
mise.toml                             # Centralized tool version management
LICENSE
README.md
```

## Build.Directory.props

Build properties that should be shared across all projects in the solution can be defined in a `Directory.Build.props` file at the root of the solution. This allows for centralized management of common properties such as target frameworks, code analysis settings, and other build configurations.

```xml
<Project>
  <!-- See https://aka.ms/dotnet/msbuild/customize for more details on customizing your build -->


  <PropertyGroup>
    <Company>Company</Company>
    <Copyright>©️ 2010-2025 Company</Copyright>
    <RepositoryUrl>https://github.com/company/repo</RepositoryUrl> 
    <RepositoryType>git</RepositoryType> 
    <Authors>author</Authors>
    <EnableNETAnalyzers>true</EnableNETAnalyzers>
    <AnalysisLevel>7</AnalysisLevel>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <RunAnalyzersDuringLiveAnalysis>true</RunAnalyzersDuringLiveAnalysis>
    <SuppressNETStableSdkPreviewMessage>true</SuppressNETStableSdkPreviewMessage>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <PropertyGroup>
    <!-- Control target framework versions in a centralized manner --> 
    <Fx>net10.0</Fx> 
     
    <!-- target different platforms with conditional properties -->
    <Windows>false</Windows>
    <Linux>false</Linux>
    <MacOs>false</MacOs>

    <RootDir>$(MSBuildThisFileDirectory.TrimEnd("/"))</RootDir>
    <EngDir>$(RootDir)/eng</EngDir>
    <AssetsDir>$(EngDir)/assets</AssetsDir>
    <IconPath>$(AssetsDir)/logo_256.png</IconPath>
    <LicensePath>$(RootDir)/LICENSE.md</LicensePath>
    <NetLegacy>false</NetLegacy>
    <NetFramework>false</NetFramework>

    <ReferenceBcl>false</ReferenceBcl>
  </PropertyGroup>

  <PropertyGroup Condition="($(TargetFramework.StartsWith('net4')) OR  $(TargetFramework.StartsWith('netstandard2.0')) OR $(TargetFramework.StartsWith('netstandard1')))">
    <DefineConstants>$(DefineConstants);NETLEGACY</DefineConstants>
    <NetLegacy>true</NetLegacy>
  </PropertyGroup>
    
  <PropertyGroup Condition="$(TargetFramework.StartsWith('net4'))">
    <NetFramework>true</NetFramework>
  </PropertyGroup>

  <PropertyGroup Condition="$([MSBuild]::IsOSPlatform('Windows'))">
    <DefineConstants>$(DefineConstants);WINDOWS</DefineConstants>
    <Windows>true</Windows>
  </PropertyGroup>

  <PropertyGroup Condition="$([MSBuild]::IsOSPlatform('OSX'))">
    <DefineConstants>$(DefineConstants);DARWIN</DefineConstants>
    <MacOs>true</MacOs>
  </PropertyGroup>

  <PropertyGroup Condition="$([MSBuild]::IsOSPlatform('Linux'))">
    <DefineConstants>$(DefineConstants);LINUX</DefineConstants>
    <Linux>true</Linux> 
  </PropertyGroup>

  <!-- Centralized static analysis rules and code style settings -->
  <ItemGroup>
    <PackageReference Include="ReflectionAnalyzers" PrivateAssets="all" />
    <PackageReference Include="SecurityCodeScan.VS2019"  PrivateAssets="all"/>
    <PackageReference Include="StyleCop.Analyzers"  PrivateAssets="all"/>
    <PackageReference Include="Text.Analyzers"  PrivateAssets="all"/>
    <PackageReference Include="AsyncFixer" PrivateAssets="all"/>
    <!-- <PackageReference Include="Microsoft.CodeAnalysis.PublicApiAnalyzers" Version="*" PrivateAssets="all" /> -->
    <PackageReference Include="Microsoft.CodeAnalysis.BannedApiAnalyzers"  PrivateAssets="all"/>
  </ItemGroup>
</Project>
```

Reference a common `Directory.Build.props` in multiple solutions:

```xml
<Project>
 <Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../'))" />
</Project>
```

## Csproj Example

```xml
<Project Sdk="Microsoft.NET.Sdk">  
  <!-- build properties specific to this project. centeralizing properties keeps this minimal
      and easy to upgrade -->
  <PropertyGroup>
    <TargetFrameworks>$(Fx)</TargetFrameworks>
    <RootNamespace>Org.LibraryA</RootNamespace>
  </PropertyGroup>

  <!-- nuget package metadata -->
  <PropertyGroup>
    <PackageReadmeFile>README.md</PackageReadmeFile>
    <PackageTags>DI Dependency Injection</PackageTags>
    <Description>
        Package description goes here. Should be a concise summary of what the package does and its key features. This is important for discoverability on NuGet and for users to quickly understand the purpose of the package. Avoid marketing language and focus on clear, factual information about the functionality provided by the package.
    </Description>
    <PackageReleaseNotes Condition="Exists('$(MSBuildProjectDirectory)/CHANGELOG.md')">
      $([System.IO.File]::ReadAllText("$(MSBuildProjectDirectory)/CHANGELOG.md"))
    </PackageReleaseNotes>
    <PackageLicenseFile Condition="Exists('$(LicensePath)')">LICENSE.md</PackageLicenseFile>
    <PackageIcon>$(IconPath)</PackageIcon>
  </PropertyGroup>

 <!-- enable common nuget package files like README.md and LICENSE.md without needing to include them in the csproj -->
  <ItemGroup>
    <None Condition="Exists('README.md')" Include="README.md" Pack="true" PackagePath=""/>
    <None Condition="Exists('$(LicensePath)')" Include="$(LicensePath)" Pack="true" PackagePath=""/>
    <None Condition="Exists('$(IconPath)')" Include="$(IconPath)" Pack="true" PackagePath="" />
  </ItemGroup>

  <!-- Add assembly attributes -->
  <ItemGroup>
      <AssemblyAttribute Include="System.CLSCompliant">
          <_Parameter1>true</_Parameter1>
          <_Parameter1_IsLiteral>true</_Parameter1_IsLiteral>
    </AssemblyAttribute>
  </ItemGroup>
</Project>
```