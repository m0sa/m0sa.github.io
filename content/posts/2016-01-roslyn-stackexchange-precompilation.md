+++
title = "Roslyn Adventures: Metaprogramming with StackExchange.Precompilation"
description = "Roslyn Adventures: Metaprogramming with StackExchange.Precompilation"
date  = "2016-01-24"
categories = [ "Development" ]
+++


In this article we'll wrap the
[StringBuilderInterpolationOptimizer](https://gist.github.com/m0sa/f086bb0e9f62e05995c6#file-
stringbuilderinterpolationoptimizer-cs) from my [previous
article](http://blog.m0sa.net/2015/10/roslyn-adventures-optimizing.html) into
an
[StackExchange.Precompiltion](https://blog.stackoverflow.com/2015/07/announcing-
stackexchange-precompilation/) module, and use it to optimize an existing C#
project.

First things first, we start off with an empty console project, to which we
add some sample `StringBuilder.Append` calls, passing interpolated strings as
parameters:

[![image](https://lh3.googleusercontent.com/-MmCCbMx5JL8/VqQb3c5rMdI/AAAAAAAAH6U/Bf97X1HhcJE/image_thumb%25255B3%25255D.png?imgmax=800)](https://lh3.googleusercontent.com/-MEXierP8YzM/VqQb2iKFYKI/AAAAAAAAH6M/eM85aqsdVyo/s1600-h/image%25255B5%25255D.png)

Now, let's add the
[StackExchange.Precompilation.Build](https://www.nuget.org/packages/StackExchange.Precompilation.Build)
package to our demo project, and make sure it still builds. This package
replaces `csc.exe` with a custom and hookable compiler, which is basically a
wrapper around the roslyn API. Unfortunately, roslyn doesn't expose any hooks
for adding custom processing from the command line
([yet](https://github.com/dotnet/roslyn/issues/3356)). Luckily the API let's
you do all kinds of crazy stuff, it even comes with a parser for the `csc.exe`
command line arguments...

After this package is successfully installed, you should see the precompiler
executable being called instead of `csc.exe` in the build output window:

[![image](https://lh3.googleusercontent.com/-MmGN28b_uQA/VqQb7YcPNpI/AAAAAAAAH6k/F_xINPY3pjY/image_thumb%25255B6%25255D.png?imgmax=800)](https://lh3.googleusercontent.com/-cR1QmqlA4ok/VqQb5zoDQwI/AAAAAAAAH6c/vuEnV8Xh8pc/s1600-h/image%25255B10%25255D.png)

Next, we need to build the compilation module that can be loaded into the
compilation. So we create an empty class library, and install the
[StackExchange.Precompilation.Metaprogramming](https://www.nuget.org/packages/StackExchange.Precompilation.Metaprogramming/)
package (contains all the required base types).

After we've pulled in the `SyntaxTree` rewriter from the last article, we can
implement the hook. The interface we need to implement is `ICompileModule`. We
want to implement the `BeforeCompilation` method, which allows us to inspect
and modify our compilation before any IL is emitted. Inside this method we
have full access to all syntax trees in the compilation and their semantic
model. Note that `CSharpCompilation` is immutable, so we have to replace the
one in the compilation context with the modified one:

[![image](https://lh3.googleusercontent.com/-thmO6g0pfsA/VqQb88L6pZI/AAAAAAAAH60/C4RFJ_ulDtc/image_thumb%25255B13%25255D.png?imgmax=800)](https://lh3.googleusercontent.com/-ukDU5Cz6lv4/VqQb79lcZHI/AAAAAAAAH6s/qMPl9UpS2_Y/s1600-h/image%25255B21%25255D.png)

All that's left to do now is to wire up our shiny new compile module, so it's
picked up as by the compiler. This is done in the app.config (or web.config
for ASP.NET project) file of the console application:

[![image](https://lh3.googleusercontent.com/-nfjOiGIXEW8/VqQb-j5W9qI/AAAAAAAAH7E/Lcdxf6VhVjY/image_thumb%25255B16%25255D.png?imgmax=800)](https://lh3.googleusercontent.com/-c0O9RpidPHA/VqQb94kYdQI/AAAAAAAAH68/xfsT_G3v2ts/s1600-h/image%25255B26%25255D.png)

The modified build script forwards the project's config file to the compiler
(this is especially useful when it needs to actually precompile razor views,
but this is out of scope of this post). Also, be sure to add the
precompilation module as a project reference to the console application.

After everything is set up correctly, a quick build and decompilation can
verify that the emitted assembly contains optimized string builder calls:

[![image](https://lh3.googleusercontent.com/-XZVP33Q7iG8/VqQcAMkUfjI/AAAAAAAAH7U/n7cfwoXJLUY/image_thumb%25255B20%25255D.png?imgmax=800)](https://lh3.googleusercontent.com/-zjm-
pQfLE1s/VqQb_GwLFPI/AAAAAAAAH7I/Af-N2jpH5qk/s1600-h/image%25255B32%25255D.png)

The sources for this demo, along with commits reflecting the described steps,
can be found on [GitHub](https://github.com/m0sa/PrecompilationDemo).

-----

P.S. a useful trick for debugging your compilation module is to add a
`Debugger.Launch()` call to attach your current VS instance to the currently
executing compilation.