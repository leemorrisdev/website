+++
date = "2016-01-04T21:41:03Z"
tags = ["Development"]
title = "Testing with mocks"

+++

I stumbled on an article whilst browsing hacker news earlier that I found interesting.

http://arlobelshee.com/the-no-mocks-book/

The article argues that if, as a developer, you come up against code requiring a mock, then the code should be refactored in a manner that allows you to avoid using a mock.

My problem is that we have to make a big assumption right off the bat that mocks are bad.  I don't view mocks as inherently evil, or that the use of mocks tells us anything in particular about the codebase.  In many cases they are a simple, quick and effective way of setting up your tests, avoiding the need to create interfaces purely so you can create test versions of classes.

I do find mocks to be fairly brittle at times, as they can require a lot of extra work to maintain when refactoring API's, but it can be a price worth paying.
