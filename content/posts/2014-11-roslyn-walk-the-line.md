+++
title = "Roslyn Adventures: Walk the #line"
description = "Hittingh debugger breakpoints after generating/rewriting C# code"
date = "2014-11-14"
categories = [ "Development" ]
+++

With Roslyn, C# developers now have a powerful tool which makes modifying the
source code a breeze. At [Stack Exchange](http://stackexchange.com/) we've
invented our own little set of extension to C# for localization purposes,
previously described on [Matt Jibson's
blog](http://mattjibson.com/blog/2013/02/28/careers-localization-part-2-api/).
The project originally supported ASP.NET MVC views only, but we've expanded it
to C# source files (.cs) because our projects have strings that simply don't
fit into MVC views and rendering a view for each string is overkill. In this
blog series I'll try to highlight some of the fun things I've learned on this
`.cshtml` -&gt;`.cs` journey.  

##  Localization Adventures: Walk the `#line`

With code rewriting being the biggest problem, one can quickly forget to
support other things in the development workflow, like **debugging**. Take
ASP.NET MVC's `.cshtml` views for example. Although they are compiled to C#
source code (you get CodeDOM from Razor, ASP.NET vNext will change that) you
can still step through breakpoints in the `.cshtml` file. In C# we have the
`#line` [preprocessor directive](http://msdn.microsoft.com/en-
us/library/34dk387t.aspx) that can enables us to do this. As _trivial_ as it
might seem, it's not so easy to get right. The fun part is having more than
one statement on a single line. Let's say we have this simple program fragment
that we [need to rewrite](https://gist.github.com/m0sa/01a16f5dc056ce0e5c8d#file-program-original-cs):

```csharp
namespace ConsoleApplication1
{
    class Program
    {
        static void Main(string[] args)
        {
            string foobar = _s("$foo$ bar", new { foo = Foo() }), baz = Baz();
        }

        static string Foo() { return "Foo"; }
        static string Baz() { return "Baz"; }
    }
}
```

… and the whole thing get's [(badly) rewritten into](https://gist.github.com/m0sa/01a16f5dc056ce0e5c8d#file-program-rewritten-badly-cs):

```csharp
namespace ConsoleApplication1
{
    class Program
    {
        static void Main(string[] args)
        {
            string _s_123 = string.Concat(Foo(), " bar");
            string foobar = _s_123, baz = Baz();
        }

        static string Foo() { return "Foo"; }
        static string Baz() { return "Baz"; }
    }
}
```

The question is, how to place the `#line` directives to hit a breakpoint on
the `baz = Baz()` statement? A clue lies in the breakpoint itself, which
Visual Studio shows as `Program.Original.cs, line 7 character 67`. As you can
see though, there is no way to specify `character 67` using the `#line`
directive. To solve the problem we need to be aware of two simple facts the
[MSDN page](http://msdn.microsoft.com/en-us/library/34dk387t.aspx) forgets to
mention:

  * the same line number can be used by multiple #line directives
  * whitespace is important when using the #line directive

Which brings us to the
[solution](https://gist.github.com/m0sa/01a16f5dc056ce0e5c8d):

```csharp
#line 1 "Program.Original.cs"
namespace ConsoleApplication1
{
    class Program
    {
        static void Main(string[] args)
        {
#line hidden
            string _s_123 = string.Concat(Foo(), " bar");
#line 7
            string foobar = _s_123                              ,
#line 7
                                                                  baz = Baz();
        }

        static string Foo() { return "Foo"; }
        static string Baz() { return "Baz"; }
    }
}
```

[![pics or it didn't 
happen](http://i.imgur.com/SDYjx73.png)](http://i.imgur.com/SDYjx73.png)
