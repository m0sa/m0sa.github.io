+++
title = "jQuery.validation breaks jQuery 1.5 ajax API"
date = "2011-02-03"
categories = [ "Development" ]
+++

Today I found out (the hard way) that the use of the
[jQuery.validation](http://bassistance.de/jquery-plugins/jquery-plugin-
validation/) plugin breaks the
[jQuery.ajax](http://api.jquery.com/jQuery.ajax/) API. Since the [Asp.net MVC
3](http://www.asp.net/mvc) uses jQuery.validation for its [unobtrusive
validation](http://weblogs.asp.net/mikaelsoderstrom/archive/2010/10/06/unobtrusive-
validation-in-asp-net-mvc-3.aspx) it is included in the default project
template. You can imagine my surprise when I updated jQuery to v1.5, and my
heavy ajaxified MVC page stopped working correctly. Every ajax request
contained an _parseerror_ with an error message stating that the callback
_jQuery[some random string] was not called_. I tried to fiddle with the [ajax
settings](http://api.jquery.com/jQuery.ajax/#jQuery-ajax-settings) for JSONP
(namely _jsonp_ and _jsonpCallback_), but no matter how I set them, the error
was still present.

[![error_example](https://lh6.googleusercontent.com/_-AYvMXY8i7Q/TUnzg-
ApZ4I/AAAAAAAAAD8/5T_rDsz_0L4/error_example_thumb%5B143%5D.png?imgmax=800)](https://lh5.googleusercontent.com/_-AYvMXY8i7Q/TUnzgfsqExI/AAAAAAAAAD4/-l5paatvD30/s1600-h/error_example%5B145%5D.png)  

So I started digging around, trying to replicate the problem. When I ran
simple html pages [$.ajax](http://api.jquery.com/jQuery.ajax/) worked as
expected. Then I started to gradually test all the jQuery plugins and found
out that the plugin causing the error was
[jQuery.validation](http://bassistance.de/jquery-plugins/jquery-plugin-
validation/). So here is my reproduction and a simple workaround (the whole
project is available [here](http://dl.dropbox.com/u/6819076/jQuery15Test.7z)):

**The controller:**
```csharp
using System.Linq;
using System.Web.Mvc;

public partial class DefaultController : Controller
{
  private static readonly int[] numbers = Enumerable.Range(1, 10).ToArray();

  public virtual ActionResult Index()
  {
    return View();
  }

  [HttpPost]
  [ValidateAntiForgeryToken]
  public virtual JsonResult SimpleArray(int id)
  {
    return Json(numbers.Concat(new [] { id }));
  }
}
```

**The view:**
```cshtml
@{ Layout = null; }  
<!DOCTYPE html>  
<html>  
<head>  
<title>Index</title>  
  <script src="/Scripts/jquery-1.5.js" type="text/javascript"></script>  
  <script src="/Scripts/jquery.validate.js" type="text/javascript"></script>  
</head>  
  <body>  
  <script type="text/javascript">  
    $(function () {  
      $('a').click(function (e) {  
        var req = $.ajax({  
            url: '@Url.Action(MVC.Default.SimpleArray())',  
            type: 'POST',   
            data: $('form').serializeArray(),  
            dataType: 'json'   
        });  
        req.success(function (response, status, xhr) { alert('Success: ' + response); });  
        req.error(function (xhr, error, msg) { alert('Error "' + error + '": ' + msg); });  
      });  
    });  
  </script>  
  <a href="javascript:void(0);">Do request</a>  
  <form action="#">  
    <input type="hidden" value="-1" name="id" />  
    @Html.AntiForgeryToken()  
  </form>  
</body>  
</html>
```

**The workaround:**
```js
$(function () {
  $.ajaxSettings.cache = false;
  $.ajaxSettings.jsonp = undefined;
  $.ajaxSettings.jsonpCallback = undefined;
})
```

The cause of the problem is this line of JavaScript in jQuery.validate.js,
that overrides the settings you pass into the $.ajax call with all the default
ones (and jQuery.ajaxSettings defaults to { jsonp: "callback", jsonpCallback:
function() {...}}):

```js
    // create settings for compatibility with ajaxSetup
settings = $.extend(settings, $.extend({}, $.ajaxSettings, settings));
```
