+++
title = "Stack Overflow .NET Core Migration: IIS"
date = "2020-04-30"
description = "TODO"
categories = [ "Development", "IIS", "Stack Overflow", ".NET Core" ]
+++

## Intro

This is part of a longer series of posts found [here]().

TODO

## Local development

Requirements:

1. check out the repository, build without IDE and the page is up
1. F5 in visual studio (VS) launches the webpage and attaches the debugger to IIS
1. easily switch between configurations (e.g Debug, Release, we also have Enterprise etc)

We have a set of local development environment setup scripts that live in a separate directory, and  automate what every developer or designer was expected to do by hand in the past. It:

1. checks out the repository of the project you want to set up.
1. restores the required databases
1. takes care of DNS by modify the `hosts` file, setting up one or more `local.xyz` domains to point at `localhost`.
1. sets up IIS websites to point at the repo directory, and creates / sets up HTTPS certificates.
1. runs a build script

After all of that it's done, one can open `local.xyz` in a browser and everything is up and running!

Since we basically want to run

```bash
> git checkout
> msbuild
```

and have things up and running, the first thing that we ran into was that the output directory of aspnetcore application now contains the build configuration in the output path, e.g. `/bin/<CONFIGURATION>/<TFM>` where previously this was just `/bin`.
This becomes problematic since the `web.config` (which is still at the root of the directory, and is used by IIS to spin up the app) now needs to contain the path to the assembly used by ASP.NET Core Module (ANCM).
In order for the web.config to still be able to point at `/bin/App.dll` we had to change the `.csproj` to contain:

```xml
  <PropertyGroup>
    <AppendTargetFrameworkToOutputPath>false</AppendTargetFrameworkToOutputPath>
    <AppendRuntimeIdentifierToOutputPath>false</AppendRuntimeIdentifierToOutputPath>
    <OutDir>bin\</OutDir>
    <ANCMPreConfiguredForIIS>true</ANCMPreConfiguredForIIS>
  </PropertyGroup>
```

The initial / official way of doing this was to run everything from VS, where the ASPNETCORE tooling would modify the web.config to point at the correct assembly / path.
This obviously didn't work out for us, since we want to rely on command-line scripts only, so that designers never have to open VS.
Additionally, we don't want to check in accidentally modified web.config files after every time somebody needs to run a different configuration locally.
During the porting effort, we talked a lot with Microsoft and pushed very hard for the addition of the `ANCMPreConfiguredForIIS` property that disables the VS tooling.
Some of the discussion is also public in the aspnetcore [github repository](TODO link).

The changes to the `.csproj` work fine, but only until you're already running, and need to rebuild.
IIS was hosting ASP.NET apps very differently than it hosts the ANCM.
The assemblies were shadow-copied, by IIS, and after new assemblies were built, those were automatically get picked up again.
Unfortunately, ANCM now runs them in-place, so they get locked.
VS's solution is of course to manage the IIS process, but we didn't want to rely on VS for this.
Luckily IIS can be controlled by the [App_offline.htm](https://docs.microsoft.com/en-us/aspnet/web-forms/overview/deployment/advanced-enterprise-web-deployment/taking-web-applications-offline-with-web-deploy) file, so we added this to our `.csproj`:

```xml
<ItemGroup>
    <None Remove="App_offline.htm" /> <!-- Prevent flicker in solution  view on builds -->
</ItemGroup>
<Target Name="WriteAppOffline" BeforeTargets="BeforeBuild;BeforeClean">
    <WriteLinesToFile Lines="Building..." File="App_offline.htm" />
</Target>
<Target Name="RemoveAppOffline" BeforeTargets="AfterBuild;AfterClean">
    <Delete Files="App_offline.htm" />
</Target>
 ```

To get `F5` / debugging work correctly from VS, we had to create the correct `launchSettings.json` file, but since that's a new concept it wasn't problematic at all.

## Production

Requirements:
- [robocopy](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/robocopy) the new binaries/configs from the build server to the production build servers

TODO Running into dlls being locked (for longer periods now, and for real)

## Conclusion

TODO
