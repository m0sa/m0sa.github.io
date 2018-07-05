+++
title = "Unit Testing JsonResult Actionss"
description = "Unit Testing JsonResult Actionss"
date = "2011-02-10"
categories = [ "Development" ]
+++

Today I had to write a unit test for a Asp.net MVC controller that returned a
JsonResult. The Controller's code looked somewhat like this:

```csharp
    [HttpPost]
    public JsonResult JsonData(Query query)
    {
      // ... process query, get total_row_cnt and rows
      return Json(new { total_row_cnt = rowCount, rows });
    }
```

I often return anonymous types as JsonResult, just because I don't want to
have additional _return model classes_ for JavaScript that are used only by
one controller action and nowhere else.

And what about unit testing such actions? Sure JsonResult exposes a property
called Data (of type object), that contains the argument passed to the
Controller's Json() call. If the type of the Data is known you can easily cast
it to the known type and do further processing. But what if you pass an
anonymous type as data to the JsonResult? How can you verify the results then?

Well you have the following options:

  * convert the anonymous type to a class,
  * directly use reflection on the returned object (magic strings!),
  * or make use of C# 4 dynamics!

The first two options are trivial, and I'm going to discuss only the last
option. Simply casting the JsonResult.Data won't do, since the type object
does not implement the necessary infrastructure. I decided to implement a
wrapper class that implements DynamicObject. Since anonymous types can offer
only readonly properties all I had to do was to override the TryGetMember
method. So here is the implementation:

```csharp
using System;
using System.Dynamic;
using System.Reflection;

public sealed class AnonymusDynamicGetWrapper : DynamicObject
{
  private readonly Type _subjectType;
  private readonly object _subject;
  public static dynamic Create(object subject)
  {
    return new AnonymusDynamicGetWrapper(subject);
  }

  private AnonymusDynamicGetWrapper(object subject)
  {
    _subject = subject;
    _subjectType = subject.GetType();
  }

  public override bool TryGetMember(GetMemberBinder binder, out object result)
  {
    result = null;
    var propertyInfo = _subjectType.GetProperty(binder.Name);
    if (propertyInfo == null) return false;
    var getter = propertyInfo.GetGetMethod();
    if (getter == null) return false;
    result = getter.Invoke(_subject, null);
    return true;
  }
}
```

And finally the unit test method:

```csharp
[TestMethod]
public void Simple_Data_Test()
{
  // ... setup the test, controller and query
  var data = controller.JsonData(query).Data;
  var rowCnt = AnonymusDynamicGetWrapper.Create(data).total_row_cnt
  Assert.AreEqual(25, rowCnt);
}
```