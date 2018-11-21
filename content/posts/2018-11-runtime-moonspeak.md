+++
title = "Getting Stack Overflow Localization Tooling Ready for .NET Core"
date = "2018-11-22"
description = "Getting Stack Overflow Localization Tooling Ready for .NET Core"
categories = [ "Development", "Stack Overflow", "Localization", "Roslyn" ]
+++

By popular demand, this is my writeup of our latest localization tooling refactoring at [Stack Overflow](https://stackoverflow.com)

{{< tweet 1064913213232427009 >}}

<!--more-->

# History

Lets talk about localization in general a bit first, though, to give you some background on how we ended up where here.

It's a story full of tears, tech debt and regret.

## 1. Careers / Talent

Your usual of-the-mill localization library works by having some sort of indirection (e.g. a key lookup) to get a string to be displayed e.g:

```csharp
Translate(SomeIDForAString, parameters)
```

Such an approach already provides you with a place where all the original strings are stored.
On the other hand, it's much less friendly towards the person reading (or writing) the code.
Ever since we started localizing products at Stack Overflow (which dates before my time), we've been using a different approach.
One that is much more developer-friendly, and much easier to read.
Let me give you an example:

```csharp
_s("Hello $name$", new { name = GetCurrentUserName() })
```

This means that all of the strings that we use are directly _in our code_.

As a consequence, though, we need tooling that knows how to extract all the string templates (so we know what to give to translators).
Historically, this tooling has been using [Roslyn](https://github.com/dotnet/roslyn) (the C# compiler, written in C#), since before it's RTM.
Originally, the tooling only did what I've highlighted above;
it extracted all string templates used to call our localization library, and verified that all the replacement tokens were passed in as arguments.

Since we use ASP.NET MVC and Razor, it also had to extract strings from there, and in order to do that we had to generate C# from the `.cshtml` files.
This turned out to also be the slowest step inside of `aspnet_compiler.exe` while precompiling the views (it had to re-do the same work, we already did before).

If you're interested in more background, you can read [@mjibson](https://twitter.com/mjibson)'s Careers Localization series ([part 1](https://mattjibson.com/careers-localization/), [part 2](https://mattjibson.com/careers-api/), [part 3](https://mattjibson.com/careers-extraction/))

## 2. Stack Overflow

The real fun started when we had to localize Stack Overflow, where we care a bit more about perf, and we tried to avoid the long build times.

We figured out we can hook into Razor, and affect what code it generates.
This allowed us to attach attributes to the generated C# views.
We only needed tooling to inspect attributes from all the classes from the precompiled `.dll`.
What it also allowed us to do, was to generate the localization code (a switch statement that only required the current locale at runtime, everything else was codegen-ed) directly in the C# produced by Razor.
For a while, on Stack Overflow, we put everything that we wanted to have localized inside Razor / `.cshtml` files.
This is the original source of "precompile localization" idea.

But then, somewhere in the middle of localizing everything, we gave up.
Turns out the perf hit of rendering a Razor view for every string you want to render, ever, was just to big.
We also needed to localize `.cs` files.

[TODO IIRC there should be benchmark screenshots somewhere in hardcore][]

Enter Roslyn tooling, again, but with a twist!
At that time, `csc.exe` was still the default C# compiler, and `aspnet_compiler.exe` was (and still is super slow).
So we replaced _both_ of them, with the Roslyn-based [StackExchange.Precompilation](https://github.com/StackExchange/StackExchange.Precompilation), and to top it of, we made it extensible so it could handle all of our localization needs (both extracting the strings, as well as precompiling localization in both `.cshtml` and `.cs` files).

You can read more about this whole project [here](https://stackoverflow.blog/2015/07/23/announcing-stackexchange-precompilation/).

# .NET Core

We currently still run on ASPNET MVC 5, on the full framework. Our ultimate goal is to be able to run on .NET Core. AspNetCore is the first stepping stone towards that.

## Present

As we were evaluating ASPNET vNext (== 5+), we had always hoped to roll with the metaprogramming features that were available in the previews, either there, or what was worked on in [Roslyn generators](https://github.com/dotnet/roslyn/blob/master/docs/features/generators.md).
But the time to move to .NET Core, and ASPNET DotNetCore is now, and unfortunately none of those features are around anymore / yet.
Luckily, though, .NET tooling introduced some high-impact perf improvements that make the CLI tools much faster (e.g. `dotnet build-servers`).
Additionally, AspNetCore Razor view pre-compilation is fast!

We had to future-proof our tooling, while still maintaining backwards compatibility with our old MVC stack.

Our localization needs are still the same:

### 1. No perf regressions
Our code-gen wasn't perfect.
It relied on the Roslyn `SyntaxTree` only, we didn't use the semantic model much.
A lot of the generated code was not optimal (e.g. we passed around all of the template properties as `object`s), but calling it was still way faster that having the entire Razor pipeline on the stack.
This gave us some wiggle room for writing an optimized runtime implementation.
And we managed to make the runtime implementation as fast as the old (albeit not very optimized) compile-time code-gen-ed localization.
Kudos to [@marcgravell](https://twitter.com/marcgravell), [@Nick_Craver](https://twitter.com/Nick_Craver) for helping out here. [BenchMarkDotNet](https://github.com/dotnet/BenchmarkDotNet) was also insanely helpful.

### 2. Compile-time verification of string templates / parameters

We've already written Roslyn analyzers that did this.
Since running the whole compilation with ASP.NET MVC 5 ~2 minutes on a beefy machine, was really a show stopper for developers, we wanted to give them instant-feedback inside the VS, which squiglies and all.

### 3. Tooling to extract strings, but AspNetCore compatible

The only new requirement here is that everything it has to work with the `.NET Core CLI`
Turns out the analyzer mentioned above already has all the information we need!
AspNetCore already pre-compiles `.cshtml` -> `.cs` as part of the build, those `.cs` files can be analyzer-ed.

The main part of the work there was to port all our builds and build tooling to something that can potentially run on .NET Core CLI in the future.
The best candidate for that seemed to be a MSBuild `ILogger` implementation, which can intercept the diagnostic messages, pin-pointing them to specific locations within a source file, and spit out the artifacts we can send out for translation in the end.
This is super application, and build configuration specific, so this implementation lives in our Stack Overflow solution.

I've ran into an interesting edge case with [Roslyn vs MSBuild diagnostic levels](https://github.com/dotnet/roslyn/issues/30637) while doing this.

I've skipped on one other important detail here, we have a separate tool that rewrites JavaScript, we just made it generate the same diagnostic messages as the C# tools.
This allows us to run both that tool and the main build step in parallel, and get a unified view dump of all used strings in the end.

## Future

With all of that in place, we now finally have a super-slim MoonSpeak library, that provides:

- template syntax processing
    - compile-time safe
    - design-time feedback
    - support multiple pluralization tokens per template (e.g. 
      `_s('User $userName$ has asked #numQuestions# questions and #numAnswers# answers')`)
    - built-in Markdown support
      `@_m('hello [world](https://example.org)')`
- a super-fast runtime base implementation which can work on both MVC5, and AspNetCore
- C# string extraction via diagnostics

All the other tooling that generates the translator resources, and provide translated strings to the implementation at runtime, are up to the consuming app.
Even internally, we have different ways of doing that (Talent has a different workflow and intermediate formats there as Q&A).

We might be finally getting to a point where we can open source MoonSpeak.
