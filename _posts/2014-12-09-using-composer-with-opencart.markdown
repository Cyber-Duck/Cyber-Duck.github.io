---
layout: post
title:  "Using Composer with Opencart"
date:   2014-12-09 11:22:52
author: ben
comments: true
---

If you haven't already heard of it, [Composer](https://getcomposer.org/) is a dependency manager for php and has really made an impact in the PHP development community.
It brings the same advantages as Node's [NPM](https://www.npmjs.org/) and Ruby's [Gem](https://rubygems.org/) package managers to the PHP world. Lots of other frameworks like Laravel use it to manage their core files and plugins. There are also other projects to try and bring Composer management to products like [Wordpress](http://wpackagist.org/) and with just a little modification we can try to bring some of these same advantages to Opencart.

### Use in Opencart
Unfortunately at the time of writing Opencart does not support Composer out of the box. I have added a suggestion to their [uservoice](http://opencart.uservoice.com/forums/52387-general/suggestions/6817537-use-composer-to-manage-opencart-and-extensions), so if you think its a good idea please vote for it!
However we can quite easily integrate composer with opencart and there is already an extension for 2.0.0.0 that does [just that](http://www.opencart.com/index.php?route=extension/extension/info&extension_id=19646&filter_search=composer).

If you want to do it yourself, you just need to include the autoload.php file somewhere in the Opencart startup. This is the vQmod file we usually use:

{% highlight xml startinline %}
<modification>
        <id>Add composer</id>
        <version>1.0</version>
        <vqmver>1.0.8</vqmver>
        <author>Ben Speakman</author>
        <file name="system/startup.php">
                <operation>
                        <search position="after"><![CDATA[
                        require_once(DIR_SYSTEM . 'library/template.php');
                        ]]></search>
                        <add><![CDATA[
                        require DIR_SYSTEM . '../vendor/autoload.php';
                        ]]></add>
                </operation>
        </file>
</modification>
{% endhighlight %}

That's it and now you can start to require packages for your project. First thing is to [install Composer](https://getcomposer.org/download/) and create a composer.json for your project. Here is a sample one to get you started:

{% highlight json startinline %}
{
    "name": "CyberDuck/my-project",
    "description": "My Project",
    "require": {
        "php": ">=5.2.0"
    },
    "minimum-stability": "dev"
}
{% endhighlight %}

### Packages to install
There are lots of packages to choose from. However as a starter here are some we use regularly in our Opencart installations.

#### Whoops / Sentry
Opencart's error handling leaves a lot to be desired and we love the [Whoops](http://filp.github.io/whoops/) library. We also use [Sentry](https://getsentry.com/welcome/) to report on PHP errors in production. So we created [a repo](https://github.com/Cyber-Duck/opencart-sentry-raven-whoops) that shows how you can integrate Whoops and Sentry into opencart with Composer.

#### Phinx
Another issue with Opencart is that it doesn't have any built in support for database migrations.
However there is a nice stand alone package called [Phinx](https://phinx.org/) that lets you create and manage migrations.
All you need to do is add phinx as a dependency in your composer.json:

{% highlight json startinline %}
{
    "require": {
        "robmorgan/phinx": "*"
    }
}
{% endhighlight %}

Then you can run `php vendor/bin/phinx init` to create your phinx.yml file and add in your database info.
Check the [documentation](https://phinx.readthedocs.org/en/latest/migrations.html) for details on how to write and run migrations.

*Please be aware that by default Opencart does not block access to yml files. So make sure your database details are not available to the public!*

### Going forward

Hopefully we have inspired you to see what other packages you can pull into opencart with Composer and maybe you will be able to enhance your Opencart project.
There are lots of Composer repositories where you can search for and download packages available but the main one is [Packagist](https://packagist.org/).
