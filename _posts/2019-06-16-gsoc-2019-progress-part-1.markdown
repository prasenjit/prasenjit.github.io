---
layout: post
title: 'GSoC 2019: Progress Report 1'
categories: [GSoC]
tags: [GSoC]
---
It has been around three weeks since the starting of the coding period, and these three weeks have been crazy hectic. Coding, reading code, designing classes and implement them, then change them, reading articles, blogs, etc. are some of the few things I have been doing. But I am enjoying this. I have learnt about so many new stuffs that I didn't know about earlier.

Ok, coming back to the point. My GSoC project is to incorporate the feature of syncing user data of the Falkon browser. It is to be done using Firefox Accounts server. It involves of the following steps:

1. Load the Firefox Accounts login page. Let the user fill up the details and click login.
2. Recieve the initial login credentials(keys) using webchannel.
3. Process the initial login keys to get the required keys for encrypting and decrypting the user data.
4. Get Browser assertion certificates, i.e. register the user's browser for sync.
5. Make necessary requests to the Firefox Server to get the url of the Storage server.
6. Convert the user data into storage objects
7. Implement the sync logic for uploading/downloading user data.
8. Upload/Download user data from the Storage server.


The first one was something that was done in the first week of the coding period. Its the second one that took a long time. It took me a long time to understand how communication between C++/Qt and HTML happens using QtWebChannel. But at last, its done. I have been able to setup the webchannel and recieve the initial login credentials.

What follows next is processing the initial login keys. It involves using HKDF algorithm using HMAC_SHA256. Falkon uses the OpenSSL library for some of its operation. And OpenSSL provides the HKDF algorithm, but HMAC_SHA256 is something thats not available. HMAC and SHA256 are present independently. Lets see how long it takes. Sigh!

The journey so far has been hectic but interesting. And if not for my mentor and members of the #falkon irc channel, it would have been much more harder. My mentor David Rosca and user #SGOrava on channel #falkon on IRC have been really helpful.

Wish me luck for the journey ahead...

Thank you for reading...