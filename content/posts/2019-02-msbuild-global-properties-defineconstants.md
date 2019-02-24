+++
title = "Learning about MSBuild Global Properties - The Hard Way"
date = "2018-11-21"
description = "Learning about MSBuild Global Properties - The Hard Way"
categories = [ "Development", "Stack Overflow", "MSBuild", "FAIL" ]
+++

One of the first steps in the Stack Overflow .NET Core migration was to extract a Common library that would target `netstandard`.
This new project would be referenced in both the old AspNetMvc 5 projects, as well as the net new AspNetCore, so that we could start moving the old projects one-by-one.
The new Common project had a few AspNetMvc / AspNetCore specific regions that were conditionally compiled via `#if NET472 <MVC 5 CODE> #else <ASPNETCORE CODE>` directives (thanks to the new `IHtmlContent` interface which replaced the old `IHtmlString` from `System.Web`, which we use across many models).
The target framework conditional defines are a feature of the new SDK-style builds in the .NET Core ecosystem.

Everything built fine and dandy locally, but the fun started once we promoted our branch to a PR and the automatic build checks we had in place in GitHub / TeamCity kicked.
No matter what we did, we only ever got into one branch of the conditional compilation, the other one was seemingly disregarded.
My team spent around 2 days throwing WTFs around, and trying to figure out what was going on.
Ultimately we started digging into the MSBuild code that powers the multi-targeting builds.

The magic code that introduces the target framework defines lives [here](https://github.com/dotnet/sdk/blob/700964f851905dd55c75d1869129e335fd9d1e91/src/Tasks/Microsoft.NET.Build.Tasks/targets/Microsoft.NET.Sdk.CSharp.targets#L34), and at first glance everything seemed to be in order, and nobody from our team spotted what's going on.
So, I started comparing what our CI build does differently from our local build (which worked fine).
Finally I managed to repro what's going on, by including an additional `/p:DefineContants=WHATEVER` on the `msbuild` command-line, locally.
Our CI PR verification build was passing a conditional compilation symbol in order to speed up the build a bit (we have one that enables us to only localize for English only, which means a lot less IO)
Next, I compared the msbuild binlog (you can get one by running msbuild with the `/bl` switch), and used the excelent [MSBuild Binary and Structured Log Viewer](http://www.msbuildlog.com/) where I searched for `DefineConstants`.
The failing case turned out to short-circuit on whatever was passed in the command-line switch (e.g. WHATEVER), and never re-evaluated the property.
Lucky for me, the binlog contained a hint as to why this is happening; the DefineConstants property was under a "global properties" node in the tree.
Once I googled for "msbuild global properties" I got to [this piece of doc](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-properties#global-properties) fairly quickly.

In the end, the TIL for the day was, if a property is a global property, and you pass it to msbuild via command-line args, the value passed in is used, regardless of what your build scripts do.

I've came up with a trivial example to explain to the team, what was going on in MSBuild:

![MSBuild Global Property Example](https://i.stack.imgur.com/M9EYD.png)

I've also opened up an [Issue on in the DotNet SDK](https://github.com/dotnet/sdk/issues/2854), since using a global property for the multi-targeting part makes the whole thing very brittle, and I'd expect some defensive programming around that.