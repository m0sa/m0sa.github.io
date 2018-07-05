+++
title = "Obfuscated JS Hello World CodeGolf"
date = "2011-02-04"
categories = [ "Development" ]
+++

Finally the [codegolf Stackexchange](http://codegolf.stackexchage.com) site is in beta. So I decided to give it a
try, as I found
[this](http://codegolf.stackexchange.com/questions/307/obfuscated-hello-world)
nice codegolf:

[![golf](http://lh6.ggpht.com/_-AYvMXY8i7Q/TUtOiX1x-ZI/AAAAAAAAAEE/2ipQ5l8Q4d4/golf_thumb%5B1%5D.png?imgmax=800)](http://lh6.ggpht.com/_-AYvMXY8i7Q/TUtOiKTSVGI/AAAAAAAAAEA/RWA-
_dMVaaE/s1600-h/golf%5B3%5D.png)

So…. Behold my
[solution](http://codegolf.stackexchange.com/questions/307/obfuscated-hello-
world/453#453) with JavaScript:

```javascript
    javascript:(_=(_=[][(f=!!(_='')+_)[3]+(b=({}+_)[1])+($=(c=(d=!_+_)[1])+d[0])])())[f[1]+'l'+(a=d[3])+$]("H"+a+'ll'+b+' W'+b+c+'ld');
```

Feel free to try it out. Just paste it in your browser's address bar and let
it do the magic.

Well if you would like to know how it works I suggest you read this blog post
first (I've actually shared it in my Google Reader feed a month back, so you
might have already seen it). The script uses the same principle to acquire the
sort() function (and consequently the window object and alert() function) as
described there. I also found this nice [video](http://vimeo.com/15961577)
that was very helpful.
