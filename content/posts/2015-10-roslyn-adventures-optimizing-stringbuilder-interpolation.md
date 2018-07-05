+++
title = "Roslyn Adventures: Optimizing StringBuilder string interpolation"
date = "2015-10-19"
categories = [ "Development" ]
+++

C# string interpolation is awesome. But we can make it even more awesome by
making it less wasteful. Consider this line of code:

```csharp
new StringBuilder().Append($"foo {1}");
```

Currently Roslyn emits the following IL for this call:

```
IL_0001:  newobj      System.Text.StringBuilder..ctor
IL_0006:  ldstr       "foo {0}"
IL_000B:  ldc.i4.1
IL_000C:  box         System.Int32
IL_0011:  call        System.String.Format
IL_0016:  call        System.Text.StringBuilder.Append
```

You see what it does there? Let's translate that back to C#, to make it more
obvious:

```csharp
new StringBuilder().Append(string.Format("foo {0}", 1));
```

It [allocates](http://blog.marcgravell.com/2013/11/allocaction-allocation-
allocation.html) another string and possibly even
[another](http://referencesource.microsoft.com/#mscorlib/system/string.cs,bdf91919a8d3537e)
[StringBuilder](http://referencesource.microsoft.com/#mscorlib/system/text/stringbuildercache.cs,9ce41f3defeef16f).
The thing is, you wouldn't be using a `StringBuilder` if you weren't concerned
about allocations. My initial idea of how to solve this was
`FormattableString`. So basically something like this:

```csharp
class Foo
{
  public static void Bar(FormattableString str)
  {
    Console.WriteLine("FORMAT: " + str.Format, str.GetArguments());
  }

  public static void Bar(string str)
  {
    Console.WriteLine("STRING: " + str);
  }

  public static void Main()
  {
    Bar($"a test {42}");
  }
}
```

Unfortunately overload resolution doesn't work in favor of the method
accepting the `FormattableString` whenever there is an overload for the string
parameter. So the example above would write `STRING: a test 42` to the
console. If we could make the overload resolution smarter (e.g. make the
compiler create FormattableString instances wherever there's an matching
overload accepting FormattableString instead of a string argument) the
solution would be as easy as creating `Append/AppendLine(FormattableString)`
extension methods for `StringBuilder`.

##  Roslyn to the rescue

Luckily we can use Roslyn to do some metaprogramming magic and work around
this. Basically we need to rewrite `StringBuilder.Append/AppendLine` calls to
`StringBuilder.AppendFormat`:
```csharp
new StringBuilder().AppendFormat("foo {0}", 1);
```

... in IL speak:
```
IL_0001:  newobj      System.Text.StringBuilder..ctor
IL_0006:  ldstr       "foo {0}"
IL_000B:  ldc.i4.1
IL_000C:  box         System.Int32
IL_0011:  call        System.Text.StringBuilder.AppendFormat
```

Let's create a reusable Roslyn-based solution that knows how to do that
optimization.  
We can use the class above to rewrite each `SyntaxTree` in a
`CSharpCompilation`:

```csharp
class StringBuilderInterpolationOptimizer : CSharpSyntaxRewriter
{
  private readonly SemanticModel _model;
  public StringBuilderInterpolationOptimizer(SemanticModel model)
  {
    _model = model;
  }

  // we use the semantic model to get the type information of the method being called
  private static bool CanRewriteSymbol(SymbolInfo symbolInfo, out bool appendNewLine)
  {
    appendNewLine = false;
    IMethodSymbol methodSymbol = symbolInfo.Symbol as IMethodSymbol;
    if (methodSymbol == null) return false;

    switch (methodSymbol.Name)
    {
      case "AppendLine":
      case "Append":
        if (methodSymbol.ContainingType.ToString() == "System.Text.StringBuilder")
        {
          appendNewLine = methodSymbol.Name == "AppendLine";
          return true;
        }
        break;
    }
    return false;
  }

  public override SyntaxNode VisitInvocationExpression(InvocationExpressionSyntax node)
  {
    var memberAccess = node.Expression as MemberAccessExpressionSyntax;
    if (memberAccess != null && node.ArgumentList.Arguments.Count == 1)
    {
      // check if the single method argument is an interpolated string
      var interpolatedStringSyntax = node.ArgumentList.Arguments[0].Expression as olatedStringExpressionSyntax;
      if (interpolatedStringSyntax != null)
      {
        bool appendNewLine; // this distinguishes Append and AppendLine calls
        if (CanRewriteSymbol(_model.GetSymbolInfo(memberAccess), out appendNewLine))
        {
          var formatCount = 0;
          var formatString = new StringBuilder();
          var formatArgs = new List<ArgumentSyntax>();

          // build the format string
          foreach (var content in interpolatedStringSyntax.Contents)
          {
            switch (content.Kind())
            {
              case SyntaxKind.InterpolatedStringText:
                var text = (InterpolatedStringTextSyntax)content;
                formatString.Append(text.TextToken.Text);
                break;
              case SyntaxKind.Interpolation:
                var interpolation = (InterpolationSyntax)content;
                formatString.Append("{");
                formatString.Append(formatCount++);
                formatString.Append(interpolation.AlignmentClause);
                formatString.Append(interpolation.FormatClause);
                formatString.Append("}");

                // the interpolations become arguments for the AppendFormat call
                formatArgs.Add(SyntaxFactory.Argument(interpolation.Expression));
                break;
            }
          }
          if (appendNewLine)
          {
            formatString.AppendLine();
          }

          // the first parameter is the format string
          formatArgs.Insert(0,
            SyntaxFactory.Argument(
              SyntaxFactory.LiteralExpression(
                SyntaxKind.StringLiteralExpression,
                SyntaxFactory.Literal(formatString.ToString()))));
                return node
                  .WithExpression(memberAccess.WithName(SyntaxFactory.IdentifierName("AppendFormat")))
                  .WithArgumentList(SyntaxFactory.ArgumentList(SyntaxFactory.SeparatedList(formatParams)));
        }
      }
    }
    return base.VisitInvocationExpression(node);
  }
}
```

Originally I wanted the optimization to do the same for
`TextWriter.Write/WriteLine` and `Console.Write/WriteLine` calls, but it turns out
that they actually [call
`string.Format`](http://referencesource.microsoft.com/#mscorlib/system/io/textwriter.cs,534)
internally anyway. So, add that to the list of possible optimizations.

##  Conclusion

As you've seen by yourself, it's not particulairly difficult to optimize the
back and forth between string interpolation and `StringBuilder`. I really
think optimizations like that should be in Roslyn. As long as that's not
implemented though, using string interpolation to build big string fragments
(like for example HTML...) might be a bit more expensive than you might think.

Stay tuned for my next blog post, where I'll show you how to plug the
optimization into the metaprogramming infrastructure of DNX and/or
[StackExchange.Precompilation](https://blog.stackoverflow.com/2015/07/announcing-
stackexchange-precompilation/), if you're not ready to migrate to vNext yet.
