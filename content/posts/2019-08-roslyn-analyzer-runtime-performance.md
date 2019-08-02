+++
title = "How To Find Roslyn Analyzer Performance"
date = "2019-08-02"
description = "A quick tip on how to easily surface roslyn analyzer performance numbers"
categories = [ "Development", "MSBuild", "Roslyn", "Performance" ]
+++

# TL;DR

Do you write Roslyn analyzers? Ever wondered what impact they have on your compilation times? Let me show you how!

To get the numbers one can use the [ReportAnalyzer](https://github.com/microsoft/msbuild/blob/vs16.0/src/Tasks/Microsoft.CSharp.CurrentVersion.targets#L286) property, that gets passed in to the C# / VB compilation tasks.
It's the MSBuild equivalent of the [`-reportanalyzer`](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/compiler-options/listed-alphabetically) C# compiler command line switch.

I usually use it in combination with the [`-bl` MSBuild switch](https://github.com/Microsoft/msbuild/blob/master/documentation/wiki/Binary-Log.md), so I can inspect the output with the the excellent [MSBuild Binary and Structured Log Viewer](http://www.msbuildlog.com/).

```bash
msbuild Your/Project.csproj /p:ReportAnalyzer=true /bl:out.binlog
```

After the build is done, you can find the relevant bits right after the `Csc` task

![roslyn analyzer performance numbers](https://i.stack.imgur.com/TN04A.png)

----

# Backstory

I was looking into increased build times (see image above) after introducing some new analyzers, and I couldn't find a good reference for this kind of investigation.

This is another Stack Overflow .NET Core migration war story.

As we're working on porting Stack Overflow to .NET Core, there will be a temporary phase, where some of the applications in our codebase will run on ASP.NET MVC, while others will already be ported to ASP.NET CORE (see my previous [blog post](https://m0sa.net/posts/2019-02-msbuild-global-properties-defineconstants/)).
For our domain logic and data models this means that they will have to be compiled against both.
The most simple way to achieve this is if they don't depend on ASP.NET at all.
We've worked hard on this decoupling but we were still left with the occasional IHtmlString reference, etc.
Some ASP.NET references had to stay, but we didn't want them to grow unwieldy.
So we created Roslyn analyzers that check for ASP.NET usage, and fail the build if they find an usage that is not explicitly opted in (e.g. by using [`#pragma warning disable ASPNETUSAGE`](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/preprocessor-directives/preprocessor-pragma-warning) or via [`SuppressMessageAttribute`](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.codeanalysis.suppressmessageattribute)).
Developers that do feature work get a nice error at build time, and a squiggly in their editor, whenever they accidentally introduce a new ASP.NET reference into the shared library.
This prevents them from stepping on the toes of the developers working on the port.
