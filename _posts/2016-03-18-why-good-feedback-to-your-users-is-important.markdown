---
layout: post
title:  "Why close production data during QA, poor connection testing and good feedback to your user are important for a mobile project"
date:   2016-03-18 15:07:32
categories: jekyll update
author: clem
comment: true
---

Just wanted to post something really quickly, not really technical (that you might be already aware) and as it's been (quite) a while since we haven't posted anything here, let get rid of the dust!

![](http://i.imgur.com/8wHGogr.gif)

We've recently developed an iPad application for one of our client and it's been quite a successful project so far.
Except maybe the fact that we've been using Parse. Yeah, I know but who could have predicted that the leader on the MBaaS market would shutdown right? They've announced **literally** a week after the project was very close from completion that they will be shutting down their services in 2017. I was with that guy, in the same liquor store...

![](http://s3.amazonaws.com/www.appcelerator.com.images/parse-tweet-sentiments.png)

But that's a completely different story, that we might cover here in the future, as we had to move away from Parse services as well.

### Introduction

From that successful project, we've been learning some lessons, among those, two were quite interesting and we thought it could be useful to share them out there. First of all, a quick explanation on what this project was about... roughly.

We developed an iPad application exposing some content. The content was mainly composed by raw text data, images and PDF files. Because of the needs and the requirements of our client, an initial version of that content (which we agreed on and helped them to finalized) would be shipped within the application upon installation. The application is using an SQLite database as well as files like PDFs and images. Nothing really fancy here. That was basically the *phase 1* of the project.

*Phase 2* was about giving the ability to the application to sync its entire content with a cloud hosted service: Parse. That's getting cool, right? There is also a CMS, developed using Laravel, sitting somewhere else, which gives our client the ability to access/edit the content, which is simple and client friendly (but this is also another story).

### Poor connection testing and feedback to your user are important

Something _funny_ we experienced with our clients when the final version of the project was live. It could sound dumb but honestly it's not and as technical developers we don't necessarily consider that type of things.

When coding our sync functionality for this project, we've been using Promises to queue smaller functionalities like "check amount of objects to sync", "download a file", "go through all the remote records", "cleanup local records"...
If anything is wrong, we catch any error potentially thrown (locally or remotely) using `promise.catch()`.

That's great and very useful but we were giving full details on the error in the logs only, Parse error number, Parse error message (if it was coming from Parse), local error message if it was locally...

The only thing the user was able to see if anything is wrong was "An error occurred. Please try again later." which is a bit confusing, we have to admit.

Our client was keeping back to use saying that sync functionality wasn't working at all and we spent quite some time trying the sync in our office with a real device, with production like data (spoil!)... I couldn't reproduce the same error they were experiencing, with the same outcome...

After really struggling to reproduce the issue, we decided to organize a Skype call with our client to try to understand what was going on exactly. Skype call was almost impossible, the quality of the call was really poor, couldn't even understand what they were saying. After some checks, they found out that they had issues with their broadband and after trying to sync the application using a good connection was smooth!

What did we do? Two things:

1. Firstly, we modified the error message the end user would see to "An error occurred during the synchronization. Please make sure your internet connection is good enough and try again later."
2. Secondly, we made sure in our internal QA processes to include a "poor connection testing" phase.

### Close production data is key for QA

We've all been in that situation, at some point, where we just want simple data to hack things and develop features. It's easier for us (yes, let's face it, we're all lazy), lighter and most of the time simpler. Disclaimer: don't get me wrong! We're totally into mocking things using cool tools and complex data **BUT** sometimes, you think that if it's working with simple dev data `{ foo: "bar" }`, there is no reason why it won't work with [live data](http://pastebin.com/raw/4a8V5QBB#). Well...

![](https://media4.giphy.com/media/WXtccLGTLB1NS/200_s.gif)

Either if it's for your own internal QA (Quality Assurance) or your client's UAT (User Acceptance Testing), you need **production like data**. When we say production like, we mean data as close as possible from what's your application/architecture will have to deal with once deployed live. We assume that for development purposes that could be fine, you don't have necessarily to use that kind of data but as soon as it gets tested and reviewed, it's becoming crucial.

Here with our case, luckily we had some production data. We've worked on making sure that during *Phase 1*, we were duplicating on Parse the data and keeping track on the initial version of the content delivered with the application. This way, we built our production data on Parse from what we agreed with our client, and this was done way before the sync was happening on the iPad application itself. However, the mistake we did was not to use until the sync was live. What happened then was pretty simple: it simply didn't work properly. Our internal QA and our client's UAT where not using production like data either, we were both using simplistic data.

If you don't have production like data yet, try to build yourself (mocking) a **proper** set of data. Depending on the technology you use, there are plenty of tools out there to help you to do that.

When we say "proper", we mean the **type** of data needs to be similar and we also mean **a lot**.
For example one of our issue here was that Parse is maxing up the amount of objects you can retrieve from their API. The default limit is set to 100 and can be increased to 1000. Above that limit, well you need to find another way around. This is how we found our way around: [How to retrieve all objects on Parse without API limitation](http://cyber-duck.github.io/2016-03-18-how-to-retrieve-all-objects-on-parse-without-api-limitation).

Other issues bubbled up to use at this point and all of then could have been spotted with "production like" data.

That's all folks! Stay tuned for other cool stuff we might post here. In the meantime, try not to forget: [testing !== production](http://thecodinglove.com/post/141247506660/testing-vs-production)
