+++
date = "2016-01-12T15:52:03Z"
tags = ["Development"]
title = "Thoughts on web development"

+++

I found this article doing the rounds on twitter:

https://medium.com/@wob/the-sad-state-of-web-development-1603a861d29f#.jobjrxvoc

While I can't say I get as worked up about web development as much as this guy does, I certainly share a few of his opinions.

I'm not going to re-iterate his entire argument here, but I've recently completed a personal project that was Java on the server with an Angular based UI, and have a few observations in the aftermath of this.

My biggest source of frustration with working in UI-land is the sheer fragmentation of it all.  It's turned into a bit of a meme in the last few years of a new JS framework being born every week, with developers seemingly spending more time migrating code to the new shiny thing than actually building new things.

I automatically started with Angular on the UI for this, being the framework I know the most, but halfway through I did take a look at what was available to see if there was anything new or better I could be using.

After a few days I gave up, frustrated by the sheer volume of tools that all do very similar, if not the same, things.  I felt lost in a sea of cutesy names, beautiful websites and less than stellar documentation.  I found dozens of github repositories that contain complex boilerplate build systems using a hodge-podge of libraries to basically spit out a JS file and a CSS file.

I do think there's a reason tooling like this exists: throwing a bunch of javascript into a single file doesn't scale well after the first few hundred lines, and maybe you have so much CSS a preprocessor makes sense.

I do wonder whether there are parts of the whole thing that are overengineered, however, by developers who enjoy solving meta-problems more than business problems.

This isn't an attack on web developers - developing for browsers is challenging and comes with a whole can of worms you don't need to worry about so much on the server.  I just wish more effort was put into making existing libraries and tooling work the way they want, than throwing the baby out with the bathwater and adding yet another framework to the growing list of things most people will never get a chance to look at.