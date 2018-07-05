+++
title = "So long blogger"
date = "2018-07-05"
description = "migrating from blogger to hugo on netlify"
categories = [ "Development" ]
+++

It's been almost 10 years since my [first blog posts](http://blog.m0sa.net/2008/11/prva-objava.html), and I've totally neglected blogging in the past year.
I totally blame blogger for that.
I really want to blog more, so I've decided to move to a simpler setup, that'll allow me to blog more quickly, with better tools.
Nowadays I use VS Code for as many things as possible, and I'd really like to use it for blogging to. So I've decided to use a static page generator, that works with markdown.

1. Chosing a static page generator

    In the past I've had experience with Jekyll, and I've also played around with various node-based tools, which are _not very nice_ (*cough*node_modules*cough*).
    Although I quite liked VuePress, it didn't offer any nice blogging templates.
    In the end I decided to give hugo a try, and it's was very simple, with a lot of nice themes (you wouldn't like another designed-by-developer style, would you?).

1. Setting up HUGO and a theme

    ```
    $> chocolatey install hugo -y
    $> hugo new site m0sa.github.io
    $> cd blog
    $/m0sa.github.io> git init
    $/m0sa.github.io> cd themes
    $/m0sa.github.io/themes> git submodule add https://github.com/avianto/hugo-kiera
    ```

    Luckily it's pretty easy to override stuff in the themes. The one I've picked, doesn't have a rss `<link>` tag, and the blog post titles into the `<title>` in the header.
    So I've copied the `partials/header` from the theme into my `layouts` folder and tweaked it.

1. Get markdown content of my blogger posts

    Luckilly I didn't blog all to much (which is hopefully going to change now that I have this new super duper setup), so I didn't have to batch import anything.
    I've used ATS's [html-to-markdown](https://automatethatshit.com/lab/html-to-markdown) tool. I added them on the cookie blacklist in order to get more than a single post converted w/o having to register.
    The bulk of the work is adding markdown front matter, and making sure all the code is in there, formatted correctly, and that everything looks OK.


1. Set up hosting

    Initially I wanted to go with github pages, but I went with netlify instead. It has all the bells and whistles (can force https + hsts with an auto generated let's encrypt cert, CDN), and is more flexible later on (can easily add formst, AWS functions, etc), if I ever need it to be.

    I followed the guide on the [hugo homepage](https://gohugo.io/hosting-and-deployment/hosting-on-netlify/).


1. Point redirect from old blogger URLs to the new ones (not done yet...)

    Injecting `<script type="text/javascript">window.location = "<NEW-URL>";</script>` into single posts should do the trick, as [suggested by Bjørn Bråthen on SO](https://stackoverflow.com/a/20276484/155005).

    Since I only pulled _some_ (== non-crappy) blog posts over, I didn't have to do this for everything.

1. Profit?
