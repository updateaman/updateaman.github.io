---
layout: post
title: 'Dotnet - Central Package Management (CPM)'
tags: nuget package central management
categories: blog
date: 2026-02-24
---

# The Pain of Managing NuGet Packages Across Projects

Managing NuGet packages across multiple projects in a single repository can quickly become a nightmare. As your solution grows and you add more projects, each one ends up with its own `csproj` file, and each project independently specifies the versions of the packages it depends on. This decentralized approach might seem fine at first, but it creates several frustrating problems:

When you're working on a solution with five, ten, or more projects, you inevitably run into **version conflicts**. Project A uses Serilog 2.10.0, Project B uses Serilog 2.12.0, and suddenly your builds fail with cryptic dependency resolution errors. You spend hours hunting down which project has the "wrong" version, trying to figure out which one you should update. Then you update one, only to break another project that relied on the older version.

The manual overhead is exhausting. Want to upgrade a common package like Newtonsoft.Json across your entire solution? You have to open each project file individually, find the version attribute, and update it manually. In a large repository with dozens of projects, this becomes a tedious, error-prone process. You'll inevitably forget one or two projects, leading to inconsistencies.

Auditing package versions becomes a nightmare too. If you need to answer the simple question "which version of Polly are we using?" you can't just look in one place—you have to grep through dozens of project files, multiply versions, and hope you don't miss any.

## The Solution: Central Package Management (CPM)

Central Package Management (CPM) lets you manage NuGet package versions from a single file (typically `Directory.Packages.props`) instead of scattering `Version` attributes across many project files. That central file declares the package versions used by all projects under its directory tree, improving consistency, simplifying upgrades, and making audits easier.

# Benefits

- Single source of truth for package versions across a repository.
- Consistent versions for direct and (optionally) transitive dependencies.
- Simpler upgrades: update one file instead of many project files.
- Cleaner project files (no `Version` attributes on `PackageReference`).

# Prerequisites

- SDK-style projects (SDK project format).
- A NuGet/dotnet toolchain that supports central package management (modern dotnet CLI and NuGet clients). If using older tools, update to a recent .NET SDK / Visual Studio version.

# Enable Central Package Management

1. Create a `Directory.Packages.props` file at the repository root (or at the directory that should act as the scope root).

2. Add package version entries using the `PackageVersion` item. Example minimal `Directory.Packages.props`:

   ```xml
   <Project>
     <ItemGroup>
       <PackageVersion Include="Newtonsoft.Json" Version="13.0.3" />
       <PackageVersion Include="Serilog" Version="2.12.0" />
       <PackageVersion Include="Polly" Version="7.2.3" />
     </ItemGroup>
   </Project>
   ```

3. In individual project files, reference packages without a `Version` attribute. Example `csproj`:

   ```xml
   <Project Sdk="Microsoft.NET.Sdk">
     <PropertyGroup>
       <TargetFramework>net6.0</TargetFramework>
     </PropertyGroup>

     <ItemGroup>
       <PackageReference Include="Newtonsoft.Json" />
       <PackageReference Include="Serilog" />
     </ItemGroup>
   </Project>
   ```

   The build/restore process will resolve package versions from `Directory.Packages.props`.

# Scoping and overrides

- `Directory.Packages.props` applies to projects in the same directory and in subdirectories. You can add another `Directory.Packages.props` in a subfolder to restrict or override versions for that subtree.
- To override a centrally managed version for a single project you can either:
  - place a `Directory.Packages.props` closer to that project with the desired `PackageVersion`, or
  - (less recommended) add an explicit `Version` in the project file (this defeats the purpose of central management and may be disallowed by your tooling/policies).

# Managing transitive and SDK package versions

- You can include entries for packages that are normally transitive to force a specific version (useful for security or compatibility fixes).
- Central declarations affect any `PackageReference` that matches the package id regardless of whether the package was referenced directly or pulled in transitively.

# Additional tips and best practices

- Keep `Directory.Packages.props` small and focused on commonly used packages; very project-specific packages can remain in the project file.
- Use consistent formatting and add comments in the central file to explain why certain versions are pinned.
- When upgrading multiple packages, update `Directory.Packages.props` and then run restores/builds for all projects to confirm compatibility.
- Consider using tools like `dotnet-outdated`, Dependabot/GitHub auto-updates, or scripted automation to propose version bumps for the central file.

# Troubleshooting

- If a project still restores a different version, search for another `Directory.Packages.props` nearer the project or an explicit `Version` on a `PackageReference`.
- Ensure your build agents and CI runners use the same .NET SDK / NuGet client versions as your development environment.

# Conclusion

Central Package Management reduces duplication and simplifies version control for NuGet packages. Adopting `Directory.Packages.props` makes upgrades and auditing easier while keeping project files cleaner.

