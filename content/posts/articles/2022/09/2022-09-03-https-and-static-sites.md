+++
author = "Saharsh Singh"
title = "Why static web pages should use HTTPS"
slug= "https-and-static-sites"
date = "2022-09-03"
description = "For the longest time, I've been in the camp of 'I don't need to secure my static website with HTTPS'. Boy have I been wrong!"
tags = [
    "html",
    "infosec",
    "software",
    "tech",
    "web development",
    "work",
]
categories = [
    "articles",
]
+++



For the longest time, I’ve been in the camp of “I don’t need to secure my static website with HTTPS”. Boy have I been wrong!

<!--more-->

If my website collects and transmits user data back to a server - sure, let’s encrypt that. Obviously that’s a no brainer if the site is taking sensitive data like passwords, SSNs, addresses, etc from the user. However, even if the user input data is not super sensitive, but the context is that of a private exchange between the user and your site, HTTPS still makes a lot of sense. However, my view was different for publicly viewable, read only web pages. After all, my intention is to share this content as widely as possible, so why distribute it using a mechanism whose primary purpose is to conceal and prohibit access.

There are [four key concerns](https://en.wikipedia.org/wiki/Information_security#Key_concepts) when it comes to information security:

- Confidentiality
- Integrity
- Availability
- Non-repudiation

I have been ignoring the importance of HTTPS in static websites, because I have been thinking only about **confidentiality**. The role of HTTPS in static websites is largely related to **integrity**. [This article does a good job](https://www.troyhunt.com/heres-why-your-static-website-needs-https/) going through some examples that drive the point home, and this was the article that drove the point home for me. I won’t repeat/recreate the scenarios here. They largely come down to man in the middle attacks.

Essentially, unencrypted HTML content can be easily intercepted at various points in the connection between the web server and the user’s browser. Of course I knew this, and didn’t care that the interceptor can read my public content in addition to the reader who requested it in the first place. In fact, the interceptor can also read HTTPS content, since the keys used to decrypt the content are publicly distributed. The gotcha, which is now a painfully obvious realization, is that the interceptor can then also inject their own content into the HTML, usually malicious javascript. In unencrypted HTTP, there is no way for the browser to know whether the content has been tampered/corrupted or if it has maintained its **integrity**!

On the other hand, HTTPS relies on private key based encryption. Not having the web site’s private key, the interceptor is unable to re-encrypt the corrupted content so that it can be decrypted using the web site’s publicly distributed keys.

And that’s the heart of the issue. I had been somewhat aware of the debate for years now, and I had my mind made up about static websites as I stated above. However, recently I showcased this blog on one of the Hugo forums, and had someone immediately comment about my site not being served over HTTPS. This triggered a re-examination on my end, and the result is this article as well as now this site only being available over HTTPS. Cheers!
