---
layout: post
title:  "Deploying a Codeigniter application to multiple EC2 instances with Capistrano"
date:   2014-06-16 11:22:52
author: ben
comment: true
---

In this blog I am going to describe how we manage our [Codeigniter](http://ellislab.com/codeigniter) deployments across several load balanced [Amazon EC2](http://aws.amazon.com/ec2/) instances, but this should prove applicable for any PHP based application.

I am not going to go into the benefits and advantages of scalable apps here. If you aren’t aware of why a set-up like this would benefit your application you probably shouldn’t be doing it. Please remember that running an application across multiple instances is usually not necessary for the majority of web apps. However if you think your project might benefit, here are some points we think are important to note before starting.

### Thinking Distributed

The first thing we need to get to grips with is how to configure your application to be run across multiple servers. Once you begin to think of your application as just being a single cog in a bigger machine you can begin to make the necessary changes to it.

Ideally the application has to be dependent only on your web server and of course PHP (Or whatever web language you are using). It should be able to be cloned from GIT and be able to be run immediately and independently (Capistrano can prove useful if you have scripted deployment processes that need to be run, but we will get into that later).

This means the core application has to be separated from other traditional components like the database and file storage. AWS provides products like [RDS](http://aws.amazon.com/rds/) and [S3](http://aws.amazon.com/s3/) that allow you to use separate services to provide individual resources to all your instances. This separation is what makes an application truly distributed and means that individual components can be scaled up and down as required.

Usually the first change is to the database location. If you don’t already, you need a single external database instance or cluster that all of your web applications will talk to. There’s not much point in running a distributed application if they are all talking to individual local databases.

Other changes you might need to make include using a centralized cache like [ElastiCache](http://aws.amazon.com/elasticache/). Or if your application requires file storage you can use S3 to provide a centralized location so each instance has access to the same files. Alternatively you can use Capistrano's [shared folders](http://stackoverflow.com/a/4648328/908257).
There are already [plugins](http://wordpress.org/plugins/amazon-s3-and-cloudfront/) for other web software like WordPress that allow you to use S3 to host your files using the native file manager. This means you don’t need to think about keeping files synchronized across instances.

### Common Pitfalls

Some of the pitfalls we ran into whilst modifying our applications include:

#### SSL Detection

Amazon [ELB](http://aws.amazon.com/elasticloadbalancing/) offers SSL termination so that a certificate can be installed on the load balancer. This is great so you don’t have to configure your web instances to serve certificates. However it does mean that your application will effectively only see unencrypted connections from the load balancer. This poses a problem if for example you want to force HTTPS on a certain controller. How do you know which requests are secure and which are not?
Fortunately the ELB also adds the `HTTP_X_FORWARDED_PROTO` header. So we came up with the following Codeigniter helper function to force SSL based on [this thread](http://ellislab.com/forums/viewthread/83154/).

{% highlight php startinline %}
// Redirect to https if production
function force_ssl()
{

    if(ENVIRONMENT != "production") {
        return false;
    }

    $CI =& get_instance();
    $CI->config->config['base_url'] = str_replace('http://', 'https://', $CI->config->config['base_url']);

    if ($_SERVER['SERVER_PORT'] != 443 && (!empty($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] != 'https'))
    {
        redirect($CI->uri->uri_string());
    }

}
{% endhighlight %}

#### Health Checker

The ELB load balancer uses a [health checker](http://docs.aws.amazon.com/ElasticLoadBalancing/latest/DeveloperGuide/ts-elb-healthcheck.html) based on parameters you define. This makes sure hosts that go down are not included in the load balancing pool. An issue can arise if you use .htacess rules to forward all requests to non-specified subdomains to the main domain for SEO purposes. This will cause a problem when the health checker tries to check a URL on each instance and it is redirected to the main load-balanced domain. Fortunately you can use the `HTTP_USER_AGENT` AWS adds to the health checker to exclude it from your redirect rules.

{% highlight sh %}
# Exclude Amazon ELB Health Checker
RewriteCond %{HTTP_USER_AGENT} !^ELB-HealthChecker
{% endhighlight %}

This means you can get a response and therefore a health status update back from a specific instance and not have it sent through your load balancer.

#### DNS

Amazon recommends that you CNAME your domain to the DNS entry of your loadbalancer. However one caveat of the DNS system is you [can’t CNAME a root domain](http://serverfault.com/a/170200). Again AWS has thought of this with their [Route53](http://aws.amazon.com/route53/) DNS product. With this you can set up [Aliases](http://docs.aws.amazon.com/Route53/latest/DeveloperGuide/CreatingAliasRRSets.html) that map root domains to ELB load balancers. The issue is you will need to move your DNS records over to Route53. Whilst this doesn’t need to be an issue its best to think about it while you are in the planning stage.


#### SSH Keys

Another issue we experienced, which is probably more prevalent on windows based machines. Is that sometimes Capistrano will fail if it doesn’t have access to both the GIT repository and EC2 instance SSH keys. On windows make sure you are running [Pageant](http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html) and all the right keys are in there or your C:\Users\<USERNAME>\.ssh\config file. [Read more](http://nerderati.com/2011/03/simplify-your-life-with-an-ssh-config-file/) about ssh config files.

*Checking that the raw “git” command from the command line can access the repository using your keys is useful when debugging authentication problems in Capistrano.*

### Using Capistrano

[Capistrano](http://www.capistranorb.com/) is a tool we use to perform deployments on multiple machines simultaneously. There are several good articles on how to install and configure Capistrano so I wont go into it here. We found [this article](http://blog.grio.com/2012/07/how-to-deploy-your-web-app-to-amazon-ec2-using-capistrano.html) by Alberto Montagnese very helpful and based our deploy.rb on it.

{% highlight ruby %}
set :stages, %w(beta1 staging production)
set :default_stage, "beta1"
require 'capistrano/ext/multistage'

set :application, "website.com"
set :repository, "git@bitbucket.org:CyberDuck/website.git"
set :scm, "git"
set :deploy_via, :remote_cache

default_run_options[:pty] = true
ssh_options[:forward_agent] = false
set :use_sudo, true
set :user, "ec2-user"

server "ec2-XXX-XXX-XXX-XXX.eu-west-1.compute.amazonaws.com", :app, :web, :db, :primary => true
server "ec2-XX-XXX-XXX-XXX.eu-west-1.compute.amazonaws.com", :app, :web, :db

desc "check production task"
task :check_production do

    if stage.to_s == "production"
        puts " \n Are you REALLY sure you want to deploy to production?"
        puts " \n Type YARLY to continue\n "
        password = STDIN.gets[0..4] rescue nil
        if password != 'YARLY'
            puts "\n !!! NOTRLY !!!"
            exit
        end
    end
end

before "deploy", "check_production"
after "deploy:update", "deploy:cleanup"
{% endhighlight %}

In this config file we deploy our site from our [BitBucket](https://bitbucket.org/) repository to two EC2 instances that are behind our load balancer. We use the excellent [Multistage extension](https://github.com/capistrano/capistrano/wiki/2.x-Multistage-Extension) which means we can specify different additional configuration files for separate deployments to deploy different branches to separate locations.
For example this is the beta1.rb file where we specify a different web root and git branch to use:
{% highlight ruby %}
set :deploy_to, "/var/www/vhosts/website.com/beta1"
set :branch, "beta1"
{% endhighlight %}
Then we can run the following command to deploy our beta site.
{% highlight sh %}
cap beta1 deploy
{% endhighlight %}
We are specifying the servers manually in the file. There are plugins like [getservers](https://github.com/kryptek/capistrano-getservers) that will fetch a list of EC2 instances based on a tag or a security group but we don’t use them here as our server pools are generally not dynamic.
Some extra things to note about our deploy.rb:

* We use `“use_sudo = true”` to make the deployment use sudo for the file operations. This might not be what you need, so alter accordingly.
* We are using the command `“forward_agent = false”` because of our particular ssh key set-up.
* We also found that we needed to set the `“default_run_options[:pty] = true”` flag for EC2 servers.

I would very strongly recommend you read the [Capistrano documentation](https://github.com/capistrano/capistrano/wiki/2.x-Significant-Configuration-Variables) on these options and understand what they do.

### Conclusions

In short it’s not too difficult to get your application running across multiple servers. The benefits for the right type of apps can be enormous and can bring a whole new level of flexibility to the way your app can respond to changing traffic demands. Especially if used with tools like [Auto Scaling](http://aws.amazon.com/autoscaling/).
Also whilst Capistrano just happens to work for us there are other tools out there to help you manage your distributed deployments like [Grunt](http://gruntjs.com/). If you think your application would be suited for a distributed environment then experiment and see what works the best for your particular needs.
