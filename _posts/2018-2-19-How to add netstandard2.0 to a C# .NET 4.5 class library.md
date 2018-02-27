---
layout: post
title: How to add netstandard2.0 to a C# .NET 4.5 class library
---

The goal today is to add [netstandard2.0][3] support to a class library which targets full .NET 4.5

In the past, multi-targeting with portable class libraries in .net was not an easy thing to do neither it was a good support and maintenance experience. With recent updates of nuget and msbuild this has changed for good.

The library which will be converted today is [RedLock][1]. It is a c# implementation of [Distributed locks with Redis][2]. As of today the library is built with .NET 4.5 but we want to support [netstandard2.0][3] as well. There is a [table][4] where we can see what platforms can use our [netstandard2.0][3] library. IMO targeting `2.0` is your best choice today but it all depends on your vision ;)

# Step 0 - Check if all dependencies support netstandard2.0 and .NET 4.5
* [Redis][5] supports `4.5`, `4.6` and `netstandard15` which is great
* [Newtosoft JSON][6] supports basically every platform :) NICE
* [LibLog][7] is kind of 50/50. You can get it working (as described in the readme) but the installation experience is not as it was before. There is a [PR][8] which is still in progress...

# Step 1 - Convert the library to netstandard2.0

This should be relatively simple task by just editing the csproj file.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
  </PropertyGroup>
  <PropertyGroup>
    <DefineConstants>TRACE;DEBUG;NETSTANDARD2_0;LIBLOG_PORTABLE;NETSTANDARD2_0</DefineConstants>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="LibLog" Version="4.2.6" />
    <PackageReference Include="Microsoft.CSharp" Version="4.4.1" />
    <PackageReference Include="Newtonsoft.Json" Version="11.0.1" />
    <PackageReference Include="StackExchange.Redis" Version="1.2.6" />
  </ItemGroup>
</Project>
```

One IMPORTANT step is deleting the `packages.config` file. Do it :)

OK, what else. Ahh, the nuspec file is now part of the csproj file. Here it is:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
  </PropertyGroup>
  <PropertyGroup>
    <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
    <PackageId>RedLock</PackageId>
    <PackageLicenseUrl>https://github.com/Elders/RedLock/blob/master/LICENSE</PackageLicenseUrl>
    <PackageProjectUrl>https://github.com/Elders/RedLock</PackageProjectUrl>
    <PackageTags>distrubited lock redis redlock cronus</PackageTags>
    <RepositoryUrl>https://github.com/Elders/RedLock</RepositoryUrl>
    <RepositoryType>Framework</RepositoryType>
    <Authors>Elders</Authors>
  </PropertyGroup>
  <PropertyGroup>
    <DefineConstants>TRACE;DEBUG;NETSTANDARD2_0;LIBLOG_PORTABLE;NETSTANDARD2_0</DefineConstants>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="LibLog" Version="4.2.6" />
    <PackageReference Include="Microsoft.CSharp" Version="4.4.1" />
    <PackageReference Include="Newtonsoft.Json" Version="11.0.1" />
    <PackageReference Include="StackExchange.Redis" Version="1.2.6" />
  </ItemGroup>
</Project>
```

So far so good. We are ready with netstandard2.0 version of the RedLock library but we lost the .net45 support.

# Step 2 - Add support for .NET 4.5

By modifying the `TargetFrameworks` we can give the compiler a hint which frameworks we want to target. In this case we want to do something like this (notice the `s`):

```xml
<PropertyGroup>
  <TargetFrameworks>netstandard2.0;net45</TargetFrameworks>
</PropertyGroup>
```

From now on we have to add `IF` statements in the csproj file to enable multiple frameworks. It goes like `If the target framework is netstandard20 use the following references`. In the csproj language it is:

```xml
<ItemGroup Condition="'$(TargetFramework)'=='netstandard2.0'">
    <PackageReference Include="LibLog" Version="4.2.6" />
    <PackageReference Include="Microsoft.CSharp" Version="4.4.1" />
    <PackageReference Include="Newtonsoft.Json" Version="11.0.1" />
    <PackageReference Include="StackExchange.Redis" Version="1.2.6" />
</ItemGroup>
```

You could see the relevant part for .NET 4.5 in the full `csproj` snippet bellow:

```xml
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <TargetFrameworks>netstandard2.0;net45</TargetFrameworks>
    </PropertyGroup>

    <!--Nuget-->
    <PropertyGroup>
        <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
        <PackageId>RedLock</PackageId>
        <PackageLicenseUrl>https://github.com/Elders/RedLock/blob/master/LICENSE</PackageLicenseUrl>
        <PackageProjectUrl>https://github.com/Elders/RedLock</PackageProjectUrl>
        <PackageTags>distrubited lock redis redlock cronus</PackageTags>
        <RepositoryUrl>https://github.com/Elders/RedLock</RepositoryUrl>
        <RepositoryType>Framework</RepositoryType>
        <Authors>Elders</Authors>
    </PropertyGroup>

    <!--netstandard2.0-->
    <PropertyGroup Condition="'$(TargetFramework)'=='netstandard2.0'">
        <DefineConstants>TRACE;DEBUG;NETSTANDARD2_0;LIBLOG_PORTABLE;NETSTANDARD2_0</DefineConstants>
    </PropertyGroup>
    <ItemGroup Condition="'$(TargetFramework)'=='netstandard2.0'">
        <PackageReference Include="LibLog" Version="4.2.6" />
        <PackageReference Include="Microsoft.CSharp" Version="4.4.1" />
        <PackageReference Include="Newtonsoft.Json" Version="11.0.1" />
        <PackageReference Include="StackExchange.Redis" Version="1.2.6" />
    </ItemGroup>

    <!--net45-->
    <PropertyGroup Condition="'$(TargetFramework)'=='net45'">
        <DefineConstants>TRACE;DEBUG;LIBLOG_PORTABLE</DefineConstants>
    </PropertyGroup>
    <ItemGroup Condition="'$(TargetFramework)'=='net45'">
        <Reference Include="mscorlib" />
        <Reference Include="System" />
        <Reference Include="System.Core" />
        <Reference Include="Microsoft.CSharp" />
        <PackageReference Include="StackExchange.Redis" Version="1.2.6" />
        <PackageReference Include="Newtonsoft.Json" Version="11.0.1" />
        <PackageReference Include="LibLog" Version="4.2.6" />
    </ItemGroup>
</Project>

```

# Step 3 - Enjoy
Now we just need to compile and publish the [nuget package][9] GG

# Problems with the tooling
There is one drawback and it is the tooling. I am using VisualStudio 15.5.7 and I am not able to update nuget package dependencies without editing the `csproj` file. Notice the nuget reference `StackExchange.Redis` and how it appears in two places. What will happen if there is a new version of the package and we try to update? Well, VisualStudio will update only the moniker which appears first inside `TargetFrameworks` and the rest have to be updated manually. In this case the version of `StackExchange.Redis` will be updated only inside `<ItemGroup Condition="'$(TargetFramework)'=='netstandard2.0'">`.

Here is the [open issue][10] in github.

------------------------------

Software is fun! Happy coding!

------------------------------

[1]: https://github.com/Elders/RedLock
[2]: https://redis.io/topics/distlock
[3]: https://github.com/dotnet/standard/blob/master/docs/netstandard-20/README.md
[4]: https://github.com/dotnet/standard/blob/master/docs/versions.md
[5]: https://www.nuget.org/packages/StackExchange.Redis
[6]: https://www.nuget.org/packages/Newtonsoft.Json/
[7]: https://github.com/damianh/LibLog
[8]: https://github.com/damianh/LibLog/pull/149
[9]: https://www.nuget.org/packages/RedLock/2.0.0
[10]: https://github.com/NuGet/Home/issues/4681