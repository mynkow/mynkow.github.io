---
layout: post
title: Windows Service Installer using WiX
---

I will create a simple windows installer. I do not pretend you should use it or the code bellow is a generic solution for building a windows service installers but it is very handy and you could quickly apply it to any project. The full source code is available in [github project StatsD][1]{:target="_blank"}

Currently the most flexible free way for creating installers is [WiX][2]{:target="_blank"}. The WiX installer adds several project templates to Visual Studio and also adds MSBuild targets for [WiX][2]{:target="_blank"} support.

From Visual Studio create *Windows Installer XML => Setup Project*. I want to create a setup project for a windows service project and that windows service project contains dependencies. The main problem is how to handle project dependencies effectively. I could simply add all references by hand and write them in *Product.wxs* file but that is not a good solution for me. So I will directly use [heat.exe][3]{:target="_blank"} which is part of the [WiX][2]{:target="_blank"} tools to collect all files needed and the output result will be *Product.wxs* with all the dependencies. *Heat* will be called before building the setup/installer project on *PreBuildEvent*. This tool has a lot of parameters. The first important parameter is the directory which will be scanned by *heat.exe* but the directory is also used for building the correct paths for all files referenced in *Product.wxs*. In this example all projects output the assemblies in a common bin folder. Defining that directory and passing it as a parameter is done directly within the project file (define *SourceOutput* under the common *PropertyGroup*; define a compiler constant *SourceOutput=$(SourceOutput)*; pass the constant like *heat.exe* dir `$(SourceOutput) -var var.SourceOutput`). At this point all files under the bin folder will be added but heat has a way for transforming the results using **.xslt*. The *PreBuildEvent* is now complete: `<PreBuildEvent>”$(WIX)bin\heat.exe” dir $(SourceOutput) -cg References -srd -o $(ProjectDir)Product.wxs -nologo -gg -g1 -scom -sreg -t $(ProjectDir)Transformer.xslt -var var.SourceOutput</PreBuildEvent>`

```xml
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform" xmlns:msxsl="urn:schemas-microsoft-com:xslt" exclude-result-prefixes="msxsl" xmlns:wix="http://schemas.microsoft.com/wix/2006/wi">
    <xsl:output method="xml" indent="yes" />
    <xsl:strip-space elements="*"/>

    <xsl:variable name="CompanyName">NMSD</xsl:variable>
    <xsl:variable name="ProductName">StatsD PerfMon</xsl:variable>
    <xsl:variable name="UpgradeCode">5BCBE078-89E8-4EEA-9DD3-75A3477CE155</xsl:variable>

    <xsl:variable name="Component1">StatsDPerfMon</xsl:variable>
    <xsl:variable name="Component1_Title">StatsDPerfMon</xsl:variable>
    <xsl:variable name="Component1_ServiceExe">NMSD.StatsD-PerfMon.exe</xsl:variable>
    <xsl:variable name="Component1_ServiceName">StatsD Performance Monitor</xsl:variable>
    <xsl:variable name="Component1_ServiceDescr">Collects performance counters for the machine where installed and sends statistics to a StatsD server.</xsl:variable>

    <xsl:template match='/'>
        <Wix xmlns="http://schemas.microsoft.com/wix/2006/wi" xmlns:util="http://schemas.microsoft.com/wix/UtilExtension">

            <Product Id="*" Name="{$ProductName}" Language="1033" Version="1.0.0.0" Manufacturer="{$CompanyName}" UpgradeCode="{$UpgradeCode}">
                <Package InstallerVersion="200" Compressed="yes" InstallScope="perMachine" />

                <MajorUpgrade DowngradeErrorMessage="A newer version of {$ProductName} is already installed." />
                <MediaTemplate EmbedCab="yes" />

                <Feature Id="{$Component1}" Title="{$Component1_Title}" Level="1">
                    <ComponentGroupRef Id="{$Component1}_ServiceComponents" />
                </Feature>
            </Product>

            <Fragment>
                <Directory Id="TARGETDIR" Name="SourceDir">
                    <Directory Id="ProgramFilesFolder">
                        <Directory Id="CompanyFolder" Name="{$CompanyName}">
                            <Directory Id="ProductFolder" Name="{$ProductName}">
                                <Directory Id="{$Component1}_INSTALLFOLDER" Name="{$Component1}Service" />
                            </Directory>
                        </Directory>
                    </Directory>
                </Directory>
            </Fragment>

            <Fragment>
                <ComponentGroup Id="{$Component1}_ServiceComponents">
                    <Component Id="{$Component1}_ServiceComponent" Directory="{$Component1}_INSTALLFOLDER" Guid="ab621e57-b52d-4bbe-853e-a5f0ca312a73">
                        <xsl:call-template name="Component1_ReferencesTemplate" />
                        <ServiceInstall
							Id="QBServiceInstaller"
							Type="ownProcess"
							Vital="yes"
							Name="{$Component1_ServiceName}"
							DisplayName="{$Component1_ServiceName}"
							Description="{$Component1_ServiceDescr}"
							Start="auto"
							Account="LocalSystem"
							ErrorControl="ignore"
							Interactive="no"
						>
                            <!--<ServiceDependency Id="[DependencyServiceName]"/>-->
                            <!--<util:PermissionEx
									User="Authenticated Users"
									GenericAll="yes"
									ServiceChangeConfig="yes"
									ServiceEnumerateDependents="yes"
									ChangePermission="yes"
									ServiceInterrogate="yes"
									ServicePauseContinue="yes"
									ServiceQueryConfig="yes"
									ServiceQueryStatus="yes"
									ServiceStart="yes"
									ServiceStop="yes" />-->
                        </ServiceInstall>
                        <ServiceControl Id="{$Component1}StartService" Start="install" Stop="both" Remove="uninstall" Name="{$Component1_ServiceName}" Wait="yes" />
                    </Component>

                </ComponentGroup>

            </Fragment>

        </Wix>
    </xsl:template>

    <xsl:template name="Component1_ReferencesTemplate" match="@*|node()">
        <xsl:copy>
            <xsl:for-each select="wix:Wix/wix:Fragment/wix:ComponentGroup/wix:Component/wix:File[@Source and not (contains(@Source, '.pdb')) and not (contains(@Source, '.vshost.')) and (contains(@Source, 'NMSD.StatsD-PerfMon'))]">
                <xsl:apply-templates select="."/>
            </xsl:for-each>
        </xsl:copy>
    </xsl:template>

    <xsl:template match="wix:Wix/wix:Fragment/wix:ComponentGroup/wix:Component/wix:File">
        <xsl:copy>
            <xsl:choose>
                <xsl:when test="not (contains(@Source, 'NMSD.StatsD-PerfMon.exe')) or (contains(@Source, '.config'))">
                    <xsl:apply-templates select="@*[name()!='KeyPath']"/>
                    <xsl:attribute name="Vital">
                        <xsl:value-of select="'yes'"/>
                    </xsl:attribute>
                </xsl:when>
                <xsl:otherwise>
                    <xsl:apply-templates select="@*"/>
                    <xsl:attribute name="Vital">
                        <xsl:value-of select="'yes'"/>
                    </xsl:attribute>
                </xsl:otherwise>
            </xsl:choose>
        </xsl:copy>
    </xsl:template>

</xsl:stylesheet>
```

Essentially the Transformer.xslt is the place where all the changes must be done because the *Product.wxs* file will be overwritten every time the solution is built (*-o $(ProjectDir)Product.wxs*). After adding the *Transformer.xslt* to the project all the infrastructure is done and the project file looks like this:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="4.0" DefaultTargets="Build" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <PropertyGroup>
    <Configuration Condition=" '$(Configuration)' == '' ">Debug</Configuration>
    <Platform Condition=" '$(Platform)' == '' ">x86</Platform>
    <ProductVersion>3.8</ProductVersion>
    <ProjectGuid>bdf158c9-3718-4ce7-b595-ca89c641a39b</ProjectGuid>
    <SchemaVersion>2.0</SchemaVersion>
    <OutputName>NMSD.StatsD-PerfMon.Setup</OutputName>
    <OutputType>Package</OutputType>
    <WixTargetsPath Condition=" '$(WixTargetsPath)' == '' AND '$(MSBuildExtensionsPath32)' != '' ">$(MSBuildExtensionsPath32)\Microsoft\WiX\v3.x\Wix.targets</WixTargetsPath>
    <WixTargetsPath Condition=" '$(WixTargetsPath)' == '' ">$(MSBuildExtensionsPath)\Microsoft\WiX\v3.x\Wix.targets</WixTargetsPath>
    <SourceOutput>$(SolutionDir)..\bin\$(Configuration)</SourceOutput>
  </PropertyGroup>
  <PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Debug|x86' ">
    <OutputPath>..\..\bin\$(Configuration)\</OutputPath>
    <IntermediateOutputPath>obj\$(Configuration)\</IntermediateOutputPath>
    <DefineConstants>Debug;SourceOutput=$(SourceOutput)</DefineConstants>
  </PropertyGroup>
  <PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Release|x86' ">
    <OutputPath>..\..\bin\$(Configuration)\</OutputPath>
    <IntermediateOutputPath>obj\$(Configuration)\</IntermediateOutputPath>
    <DefineConstants>SourceOutput=$(SourceOutput)</DefineConstants>
  </PropertyGroup>
  <ItemGroup>
    <Compile Include="Product.wxs" />
  </ItemGroup>
  <ItemGroup>
    <Content Include="Transformer.xslt" />
  </ItemGroup>
  <Import Project="$(WixTargetsPath)" />
  <PropertyGroup>
    <PreBuildEvent>"$(WIX)bin\heat.exe" dir $(SourceOutput) -cg References -srd -o $(ProjectDir)Product.wxs -nologo -gg -g1 -scom -sreg -t $(ProjectDir)Transformer.xslt -var var.SourceOutput</PreBuildEvent>
  </PropertyGroup>
</Project>
```

[WiX][2]{:target="_blank"} has the notion of [Product][4]{:target="_blank"}, [Feature][5]{:target="_blank"}, [Component][6]{:target="_blank"}. I see it like this way. Product is the main place for configuring the installer package. It also holds one or more Features. Defining a Feature enables the option whether to install or not install a particular feature at setup time. Component describes the content of the Feature and this is the place which is important.

The first thing to do is to change the variable section `<xsl:variable>`.

In the *bin* folder there are other projects’ assemblies which should not be in the final installer package. I want only the windows service output directory NMSD.StatsD-PerfMon to be added. Easy. The template called *Component1_ReferencesTemplate* filters the directories but you have to manually write the name of the desired directory in the foreach statement: *and (contains(@Source, ‘NMSD.StatsD-PerfMon’))*

After heat has finished its job all the file tags in Product.wxs  will have a property called *KeyPath* which is wrong. Only the windows service entry point *NMSD.StatsD-PerfMon.exe* must contain that property. The xslt template at the bottom corrects this but you have to manually write the name of the file which will contain the *KeyPath* property.

In my case I did an installer with two features. Both features were developed heavily and I did several releases/deploys using the installer and I did not edit the xslt even once. I just do rebuild with psake and everything is working.

------------------------------

Software is fun! Happy coding!

------------------------------

[1]: https://github.com/Elders/StatsD
[2]: http://wixtoolset.org/releases/
[3]: http://wixtoolset.org/documentation/manual/v3/overview/heat.html
[4]: http://wixtoolset.org/documentation/manual/v3/xsd/wix/product.html
[5]: http://wixtoolset.org/documentation/manual/v3/xsd/wix/feature.html
[6]: http://wixtoolset.org/documentation/manual/v3/xsd/wix/component.html