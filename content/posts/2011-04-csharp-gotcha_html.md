+++
title = "C# gotcha"
description = "C# gotcha"
date = "2011-04-14"
categories = [ "Development" ]
+++

Consider you have the following code:

```csharp
SomeClass reference = null;  
Console.WriteLine(reference.Echo("echo"));
```

Your first thought might be `NullReferenceException`. Ha, think again! What if
you have an class defining an extension method like this:

```csharp
public static class SomeClassExt
{
    public static string Echo(this SomeClass x, string str)
    {
        return str;
    }
}
```

You have been warned :)