+++
date = "2016-02-11T13:42:25Z"
tags = ["Development"]
title = "Website security headers"

+++

I found an interesting site while browsing HackerNews today.  

[securityheaders.io](https://securityheaders.io)

It performs some basic header checks on your site to see whether you're as secure as you could be.  As it turns out my basic site wasn't!

I added the recommended headers into a .htaccess file as follows:

    <IfModule mod_headers.c>
        Header set X-XSS-Protection 1;mode=block
        Header set X-Content-Type-Options nosniff
        Header set X-Frame-Options SAMEORIGIN
        Header set Content-Security-Policy "script-src 'self'"
    </IfModule>

Very simple changes but can improve security in the browser.
