---
layout: post
title:  "The dreaded 'Trying to get property of non-object'"
date:   2016-08-16 12:30:00
author: david
comments: true
---

So at Cyber-Duck we like to use the excellent [Blade Extensions](http://robin.radic.nl/blade-extensions/index.html) which add
 various helpful features to Blade.
 
Unfortunately at present there's a bug in the package. Typically it manifests itself as error message 

```Trying to get property of non-object```

When trying to access the built-in ```$loop``` variable - a variable so useful, so goddamn glamarous, that they put it
 on the front page of their extension! This is the variable that ```$i``` wants to be when it grows up. Programmers
 lust over it. And suddenly it's protesting it's not an object? What's going on?

We'll explore the problem by provoking the bug and then see what we can do solve it. If you don't care why it works, 
skip to the end and follow the instruction. Meanwhile, I'll try to flesh out a full blog article from it. See you in a moment!

In this somewhat contrived (don't be too shocked) example, let's suppose we want to list some of the latest sites
 we've discovered recently on our web adventures:

{% highlight php startinline %} {% raw %} 
@foreach(
[
    'wibble' => 'http://www.excite.com/',
    'dibble' => 'http://www.altavista.com/'
]
 as $excitingCaption => $linkToOldWebsite)


<li><a href="{{$linkToOldWebsite}}">{{$excitingCaption}}</a></li>

@endforeach
{% endraw %} {% endhighlight %}
So let's stick that in the page and see what that compiles to. 

By the way we can make discovering that considerably easier by appending ```@breakpoint``` to the
above - one of the many useful features that Blade Extensions support. When running with Xdebug and PhpStorm correctly 
configured (and why are you developing any other way?), this will handily stop execution at that point in the 
_compiled_ file (blade can be tricky to debug otherwise). 

{% highlight php startinline %} {% raw %}  
<?php foreach(
[
'wibble' => 'http://www.excite.com/',
'dibble' => 'http://www.altavista.com/'
]
 as $excitingCaption => $linkToOldWebsite): ?>


    <li><a href="<?php echo e($linkToOldWebsite); ?>"><?php echo e($excitingCaption); ?></a></li>

<?php
app('blade.helpers')->get('loop')->looped();
endforeach;
app('blade.helpers')->get('loop')->endLoop($loop);
?>

{% endraw %} {% endhighlight %}
Well, that looks alright doesn't it? Don't know what the last bit's about but hey...

Wrong!

The documentation clearly suggests we should be seeing a ```$loop``` variable being introduced somewhere in the plain php. 
So that in principle we could go like:

{% highlight php startinline %} {% raw %}  
    @foreach(
    [
        'wibble' => 'http://www.excite.com/',
        'dibble' => 'http://www.altavista.com/'
    ]
    as $excitingCaption => $linkToOldWebsite)


        <li><a href="{{$linkToOldWebsite}}">{{$excitingCaption}} - Number {{$loop->index1}}</a></li>

    @endforeach

{% endraw %} {% endhighlight %}
But where's the ```$loop```?

A clue comes if we mix things up a bit and define the array before the ```foreach```. 

{% highlight php startinline %} {% raw %}  
<?php

    $arrayOfExcitement = [
        'wibble' => 'http://www.excite.com/',
        'dibble' => 'http://www.altavista.com/'
    ];

?>

@foreach($arrayOfExcitement as $excitingCaption => $linkToOldWebsite)

    <li><a href="{{$linkToOldWebsite}}">{{$excitingCaption}} - Number {{$loop->index1}}</a></li>

@endforeach

{% endraw %} {% endhighlight %}
That turns out to compile to (give or take a bit of indentation for clarity):

{% highlight php startinline %} {% raw %}  
<?php
    app('blade.helpers')->get('loop')->newLoop($arrayOfExcitement);
    foreach(app('blade.helpers')->get('loop')->getLastStack()->getItems() as  $excitingCaption => $linkToOldWebsite):
        $loop = app('blade.helpers')->get('loop')->loop();
?>

        <li><a href="<?php echo e($linkToOldWebsite); ?>"><?php echo e($excitingCaption); ?> - Number <?php echo e($loop->index1); ?></a></li>

<?php
        app('blade.helpers')->get('loop')->looped();
    endforeach;
    app('blade.helpers')->get('loop')->endLoop($loop);
?>
{% endraw %} {% endhighlight %}
That's rather different, isn't it? What's all this newLoop thingy-gummy about? And why is this suddenly working?
And what can we do to fix it? 

Still, at least the last bit around the foreach actually makes some sense now... 

To answer the above questions, let's start at the beginning.

Possibly the simplest way to extend Blade is to register a custom compiler by calling ```Blade::extend```. These take any non php
strings and return a parsed version - possibly including php.  

As an aside, wondering how they avoid dealing with the php without performing black magic? Blade first uses the php tokenizer to 
split off the php 'bits' - see documentation for [token_get_all](http://php.net/manual/en/function.token-get-all.php) 
then admire the source of [```Illuminate\View\Compilers\BladeCompiler```](https://github.com/laravel/framework/blob/5.1/src/Illuminate/View/Compilers/BladeCompiler.php) to understand how it does this

Anyhoo, the approach taken by Blade Extensions to compiling is extremely simple - none of the stuff you learned about when you last
studied compilers a decade or so ago - it's simply a preg_replace. 

And the pattern it's looking for by default is ```/(?<!\\w)(\s*)@foreach(?:\s*)\((.*)(?:\sas)(.*)\)/```

Yeah, that. Great thing about regular expressions is how even simple ones are immediately clear, right? No, me neither. 
But break that regex down into its constituent parts and we can see it's actually pretty simple:

* ```/(?<!\\w)```
Negative look behind - no word character here. In other words, start the regex at a sensible place!
* ```(\s*)```
Any amount of white space (makes sense)
* ```@foreach```
@foreach
* ```(?:\s*)```
Any amount of white space ahead
* ```\((.*)```
Open bracket, then any amount of any characters...
* ```(?:\sas)```
At least one white space character, then 'as'
* ```(.*)\)/```
Any amount of any character and then close bracket

***WRONG***

See thing is, when I said any character above, I was _lying_. I'm playing games with you. Like a killer whale tossing a seal. 

```.``` in PCRE terms, _without the s modifier_, matches any character. _Apart from new line_. 

And that's why it's not matching our lovely foreach above. So we need to modify it. We could stick in the s modifier, 
but actually this isn't a great idea - I have a lovely proof of this, but it's too big to fit in the margins of this page. 
 
So we have to take an alternative approach. Fortunately, we can keep this simple - we just change the ```.``` to ```.|\n```.

Well, not quite. Obviously we need to stick it in brackets. So we just change the ```.``` to ```(.|\n)```.  Oh wait, everything broke.

The above isn't quite right either. Take a look at the full foreach directive (in ```vendor/radic/blade-extensions/src/directives.php```)

{% highlight php startinline %}
    'foreach'     => [
        'pattern'     => '/(?<!\\w)(\\s*)@foreach(?:\\s*)\\((.*)(?:\\sas)(.*)\\)/',
        'replacement' => <<<'EOT'
$1<?php
app('blade.helpers')->get('loop')->newLoop($2);
foreach(app('blade.helpers')->get('loop')->getLastStack()->getItems() as $3):
    $loop = app('blade.helpers')->get('loop')->loop();
?>
EOT

{% endraw %} {% endhighlight %}
In other words, there's back-references in the replacement string. So actually we need to change the ```.``` to ```(?:.|\n)``` 
to make it a non capturing  group - it's either that or change the replacement string to change the backreference indexes, and somehow
that feels less elegant to me.

So our final regex now becomes 

`'/(?<!\w)(\s*)@foreach(?:\s*)\(((?:.|\n)*?)(?:\sas)((?:.|\n)*?)\)/'`

Alright, we have a regex. How do we use it? A quick peek at the source code makes it clear that over-rides config's 
is loaded from blade_extensions.overrides. 

So tl;dr - the solution to the problem is to create a file in your project called ```config/blade_extensions.php``` with the following content:

{% highlight php startinline %} {% raw %}  
<?php

/*
 * Blade extensions has a bug (feature?) with the regex
 * that you can't do a multiline foreach, which is fine
 * except that I *WANNA* do a multiline foreach
 *
 * So here's my regex for it instead.
 */

return [
    'overrides' => [
        'foreach' => [
            'pattern' => '/(?<!\w)(\s*)@foreach(?:\s*)\(((?:.|\n)*?)(?:\sas)((?:.|\n)*?)\)/',
        ]
    ]
];
{% endraw %} {% endhighlight %}
And now you have access to ```$loop```. Use it wisely.