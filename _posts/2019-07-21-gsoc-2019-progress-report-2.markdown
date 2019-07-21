---
layout: post
title: 'GSoC 2019: Progress Report 2'
categories: [GSoC]
tags: [GSoC]
---
Its been two months since the starting of GSoC Coding Period. The past month has been really frustrating. All requests to the Firefox Token Servers(the server which provides the crypto keys for sync, and the url of storage server) are to be hawk authenticated with credentials obtained from the Firefox Accounts login page, namely, sessionToken and keyFetchToken.

Almost everything was simple, except for the hawk part. Creating hawk request header requires so many steps. And maybe I made one small mistake in some step, and voila, the whole request becomes a "Bad Request". This happened each time I made some change. I almost rewrote the hawk module four times, following line by line, Gnome Browser's hawk implementation, Pyhawk's implementation. No matter what I did, it simply would not work. I had to setup a new Virtual Machine, build Gnome Browser, edit its source code, put in debug statements in its code, to track down where the difference in my implementation was. Luckily, I was able to track down the problem and finally fixed it.

So what now? The only step left before actual syncing is to fetch the url of the storage server. That again requires deriving some keys, creating an appropriate request. Maybe a day or two more in that, and then actual sync part comes. Preparing the data and uploading it to the server.

I was expecting to be able to fetch the url of the storage server before the second evaluation. Lets see what happens. Wish me luck :)