+++
author = "Saharsh Singh"
title = "Hugo now powers this website"
slug= "blog-and-hugo"
date = "2022-08-27"
description = "This article discusses different features of Hugo that have proved beneficial for this website."
tags = [
    "html",
    "hugo",
    "software",
    "tech",
    "web development",
    "work",
]
categories = [
    "articles",
]
aliases = [
    "/blog-and-hugo/",
]
+++

 The **BIT That Matters** is back after a long hiatus. It's also getting a long overdue makeover. I started hosting this site starting late 2017 / early 2018 on an AWS EC2 instance, using WordPress as the main part of the stack. Since then, I have been increasingly using Hugo as my go-to tech for building static web sites. Naturally, when opportunity came to do a tech refresh for this site, I went with Hugo. In this article, I want to capture some of the cool features of Hugo that came in quite handy during this tech refresh.

<!--more-->

## Why Hugo?

I had been looking for a lightweight approach to building and hosting static HTML/CSS/JS content. For quite some while now, I've relied pretty successfully on cloud storage configured as a web site that pretty much all major cloud providers offer. The tricky part has always been how to create the content. It's actually pretty straight forward for me to build out an interactive web application using React / Angular / w.e. flavor of the month JS framework. However, that kind of tooling seems overkill for just some read only content like this website.

On the other hand, I am not a seasoned website developer. My lack of skills are especially apparent on the U/X and CSS side of things. To make things worse on this front, I have very little intrinsic interest in spending much time honing my skills and intuitions in U/X and CSS. Now, I do have enough motivation to not settle for just jumping into fully managed website development services with their drag and drop visual editors. However, it's still not very natural for me to just build out a modern site from scratch using nothing but raw HTML/CSS/JS.

So, my requirements came down to following:
- does the heavy lifting on the CSS front
    - either directly or via easily plugged-in projects from the relevant community
- focused on generating static HTML/CSS/JS content
- ability to host that content myself using a standard web server of my choice
- preferably open source and supports a local development workflow
- easy to configure/modify to acheive the website structure and appearance I want

Hugo meets all these criteria. The option to pick from [ready-to-go set of themes](https://themes.gohugo.io/) is what initially drove my attraction to Hugo. That continues to be a huge driver in me continuing to use Hugo. I have been able to find a decent, unique theme for whatever I'm trying to do each time I have used Hugo for a new project. For example, for this site, I was able to find the [mini](https://github.com/nodejh/hugo-theme-mini) theme that you see now. 

## Feature Shout-Outs

I won't go into exactly how it all works and exactly how to fully set up a new project. Hugo and most published themes have decent documentation that already cover getting started. Instead, I want to share some features that stood out to me while transitioning from WordPress to Hugo.

### Write content in Markdown

This one is just a joy. One of the reasons working with raw HTML can be a real drag is all the noisy, tedious tagging all across the document. The built-in WYSIWYG editor on my WordPress instance helped, but was still less than ideal compared to my typical software development experience in local IDEs like VSCode. With Hugo, you can easily separate your content from its HTML presentation. You can [define what each page type should look like in HTML once](https://github.com/nodejh/hugo-theme-mini/tree/7f6f395052486d8cc52f768c1519dbe1c93afcd0/layouts/_default), and Hugo will map individual pieces of content, as well [auto-generated meta content](https://gohugo.io/content-management/taxonomies/) like tags, categories, and content lists to its correct presentation at website build time. This basically means I can [write all my articles in Markdown](https://github.com/nodejh/hugo-theme-mini/tree/7f6f395052486d8cc52f768c1519dbe1c93afcd0/exampleSite/content/posts) in an editor/IDE of my choosing.

### URL Aliasing

Between GitHub and various social media, I have quite a few incoming links to my old content I don't want to break. At the same time, I am no longer a fan of the `/:year/:month/:date/:slug` format for the paths my old web site had. So, I was interested in picking a new path convention for the refreshed site, and have the old existing links redirect to their new counterparts. Before digging into the Hugo's feature set, I thought I would have to do this myself via either server side redirects or create a builder that injects alias/duplicate pages at website build time. To my delight, Hugo's [URL Management](https://gohugo.io/content-management/urls/) support made this all very seamless.

### Shortcodes

[Shortcodes](https://gohugo.io/content-management/shortcodes/) are Hugo's answer to extending Markdown to match the rich content capabilities of HTML5. You can add these simple one line snippets to pull in rich content like audio, images, and video into an article. Hugo comes with [a few built-in shortcodes](https://gohugo.io/content-management/shortcodes/#use-hugos-built-in-shortcodes), but it's also fairly trivial to [implement your own](https://gohugo.io/templates/shortcode-templates/) as needed. For example, Hugo's built-in `figure` shortcode didn't seem to support SVGs, so I wrote my own shortcode to pull in my SVG files. Also, I grabbed this [custom audio shortcode](https://gist.github.com/arungop/3a15a2a99b7fffbf0176e8dcdb803db6) floating around on the Internet to migrate couple of posts that publish a podcast episode.

### Auto-Generated Meta Content 

I referenced this above in the Markdown section. Hugo will automatically generate pages that collect links to different [taxanomies](https://gohugo.io/content-management/taxonomies/) you define across your content. This gives you a lot of flexibility in designing how you want your readers to browse your content. I use this in three places - [my main feed](/), [browse](/categories) page, and [tags](/tags) page - that can be accessed via the site header. Defining a tag or a category for an article is also pretty straightforward using the [front matter](https://gohugo.io/content-management/front-matter/) of each piece content. Not to mention, you have a lot of freedom in exactly what type of taxanomies you want to define.

## What's next?

I am done with this round of updates. However, as time allows, I plan to implement following enhancements to this website.

### Free Text Search

There are some approaches [Hugo](https://gohugo.io/tools/search/) recommends that look pretty lightweight and follow the pattern already being used for building taxanomies pages. It will be nice to build out dynamic search hosted purely inside my static HTML/CSS/JS assets.

### Comments

During this WordPress to Hugo migration, I dropped the comments I had on my WordPress site. The old comments are gone forever now, but the mini theme I am using has [pretty good support](https://github.com/nodejh/hugo-theme-mini#23-add-comments) for collecting and presenting comments underneath each article. As time allows, it will be a fun exercise to integrate Disqus or similar service to power comments on this website.

## Closing Thoughts

I am not affiliated with the Hugo team whatsoever. I have just found a lot of value in this nifty tool. One thing I didn't mention is just how fast the website build process is made by Hugo. Hugo also has a `hugo server` command that stands up the whole site locally and starts watching for changes in the source files. The moment any change is saved in any of the files, the watchers get triggered, and quickly build and redeploy the whole site locally, giving you a nice [live reload](https://gohugo.io/getting-started/usage/#livereload) feature as you develop. Features like this and others I mentioned make building websites with Hugo straightforward and fun.
