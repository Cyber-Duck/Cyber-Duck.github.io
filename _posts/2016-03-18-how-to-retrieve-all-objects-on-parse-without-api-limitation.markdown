---
layout: post
title:  "How to retrieve all objects on Parse without API limitation"
date:   2016-03-18 18:07:32
author: clem
comment: true
---

Parse is great, even if they are planning to shutdown their services in 2017...
However, like a lot of other APIs they limit the amount of data you retrieve from them via a single call. The default value is `100` and can be maxed up to `1000` but what if you have 1001+ objects you'd like to retrieve?

Well... some solutions...
=========================

A lot of people are using solutions are using `limit()` alongside with `skip()` from Parse JS SDK.

[https://parse.com/questions/loading-more-than-100-objects](https://parse.com/questions/loading-more-than-100-objects)
[http://stackoverflow.com/questions/30562620/api-100-objects-limit](http://stackoverflow.com/questions/30562620/api-100-objects-limit)

It's basically like a pagination system where you can use offsets and ask objects skipping the first `1000` and retrieve the next `1000`. You would obviously have to repeat that. It's convenient and you _could_ do that but at some point, the Parse API might throw that nasty error at your face
`Error: Skips larger than 10000 are not allowed`.

![](http://replygif.net/i/1010.gif)

Our solution: Parse CloudCode!
==============================

The CloudCode **cloudcode/cloud/main.js**.
{% highlight javascript startinline %}
Parse.Cloud.define("retrieveAllObjects", function(request, status) {
    var result     = [];
    var chunk_size = 1000;
    var processCallback = function(res) {
        result = result.concat(res);
        if (res.length === chunk_size) {
            process(res[res.length-1].id);
        } else {
            status.success(result);
        }
    };
    var process = function(skip) {
        var query = new Parse.Query(request.params.object_type);
        if (skip) {
            query.greaterThan("objectId", skip);
        }
        if (request.params.update_at) {
            query.greaterThan("updatedAt", request.params.update_at);
        }
        if (request.params.only_objectId) {
            query.select("objectId");
        }
        query.limit(chunk_size);
        query.ascending("objectId");
        query.find().then(function (res) {
            processCallback(res);
        }, function (error) {
            status.error("query unsuccessful, length of result " + result.length + ", error:" + error.code + " " + error.message);
        });
    };
    process(false);
});
{% endhighlight %}

_Disclaimer:_ this was highly inspired by some snippet I found somewhere but unfortunately I can find that link again. I'll keep looking for it...

Plenty of useful information on how CloudCode works and how to deploy it, directly accessible on the [Parse documentation](https://parse.com/docs/cloudcode/guide).
This is the code you need to deploy on your Parse instance and basically this will create a new endpoint for you to consume.

Once it's deployed, it can be consumed by using either a Promise (or not, it's up to you really):
{% highlight javascript startinline %}
Parse.Cloud.run('retrieveAllObjects', {
    object_type: "MyClass", // REQUIRED - string: name of your Parse class
    update_at: moment().toDate(), // OPTIONAL - JS Date object: Only retrieve objects where update_at is higher than...
    only_objectId: true|false // OPTIONAL - boolean: the result will only be composed by objectId + date fields, otherwise all attributes are returned.
}).then(function(objects) {
    /* SUCCESS */
    // if objects.length > 0 objects can be looped through
});
{% endhighlight %}

We assume you know how to use [Parse JS SDK](https://parse.com/docs/js/guide).

By the way, [Promises](https://www.promisejs.org) (`promise.then()`) and [MomentJS](http://momentjs.com) (`moment().toDate()`) are **really** two awesome tools. Definitely recommend you check those out! We use the [Q library](https://github.com/kriskowal/q) for Promises, which is the one supported by the Parse JS SDK as well.

Of course, we're not saying our solution is the best, we simply wanted to share it with you so you can implement it if you were facing that very same problem! ;)

The big advantage of our approach is that we *don't* use `skip` and `limit`, we order our result by `objectId` which is the PK (primary key) on Parse and use a self-invoked function alongside with the PK as a pagination controller for the offset. This way, you won't ever be limited to `10000` results.

Obviously this takes a bit longer to get executed but the workload stays server-side and at least you know that the response you receive contains **everything**.

That's all folks! Checkout out [the gist](https://gist.github.com/Claymm/644eae2426a50cb2b98d) for this and feel free to comment/fork/improve/share it!
