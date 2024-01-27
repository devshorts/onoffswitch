---
layout: post
title: Automating deployments with salt, puppet, jenkins and docker
date: 2015-08-13 22:32:46.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- automation
- docker
- jenkins
- puppet
- salt
- sensu
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560966243;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4673;}i:1;a:1:{s:2:"id";i:4737;}i:2;a:1:{s:2:"id";i:5000;}}}}

permalink: "/2015/08/13/automating-deployments-salt-puppet-jenkins-docker/"
---
I know, its a buzzword mouthful. My team has had good first success leveraging jenkins, salt, sensu, puppet, and docker to package and monitor distributed java services with a one click deployment story so I wanted to share how we've set things up.

First and foremost, I've never been an ops guy. I've spent time on build systems like msbuild and fake, but never a full pipeline solution that also had to manage infrastructure, but there's a first for everything. Most companies I've worked at have had all this stuff set up already, and out of the minds of developers, but the place I am at now does not. I actually think it's been a great opportunity to dive into the full flow of how do you get your damn code out the door. I think if you are developing an application and don't know how it gets from git to your box, you should do some digging, because there is A LOT that happens.

Given that my team doesn't have a one click solution to just so "[magic](https://www.youtube.com/watch?v=gs0w98-ekFk)!" I set out to build an initial workflow for our team that would let us package, deploy, monitor, and version our infrastructure. I wanted the entire setup to be jenkins driven so everything we have is versioned in github enterprise with jenkins hooks to build repositories. Full disclaimer, this isn't meant to work for enormous teams or companies, but just for what mine and another team are doing.

None of this is really all that new or interesting to anyone whose done it, but I wanted to document how and why we did it the way we have.

# Puppet

The first thing we did was create a puppet repo. You may have remembered in a past post I blogged before how that led to testing puppet scripts with docker (_which sort of fizzled due to puppet [not having access to systemd](http://developerblog.redhat.com/2014/05/05/running-systemd-within-docker-container/) for service starting. It would work on a pure linux box, but given boot2docker doesn't have cgroups to virtually mount it wouldn't work on a mac_). Our puppet scripts are set up by environment:

![Puppet directory structure](http://onoffswitch.net/wp-content/uploads/2015/08/Screen-Shot-2015-08-13-at-2.25.37-PM.png)

Where our "/data" folder will get mapped to "/etc/puppet".

We use a custom [facter](https://puppetlabs.com/facter) to identify nodes in our environments. Instead of setting up our site.pp manifest with actual node matching, everyone has a custom role and our site delegates what to do based on that role:

```
  
$rmq\_master = "..."  
$salt\_master = "..."  
$sensu\_host = "..."

node default {  
 case $node\_role{

'rmq\_master': {  
 class { 'domains::monitoring::salt::client':  
 salt\_master =\> $salt\_master  
 }

class { "domains::rmq::master" :  
 sensu\_host =\> $sensu\_host  
 }  
 }

'rmq\_slave': {  
 class { 'domains::monitoring::salt::client':  
 salt\_master =\> $salt\_master  
 }

class { "domains::rmq::slave" :  
 master\_nodename =\> $rmq\_master,  
 sensu\_host =\> $sensu\_host  
 }  
 }

'sensu\_server': {  
 class { 'domains::monitoring::salt::client':  
 salt\_master =\> $salt\_master  
 }

class { "domains::monitoring::sensu::server": }  
 }

'jenkins-master' : {  
 class { 'domains::monitoring::salt::client':  
 salt\_master =\> $salt\_master  
 }

class { "domains::cicd::jenkins-master":  
 sensu\_host =\> $sensu\_host  
 }  
 }

'jenkins-slave' : {  
 class { 'domains::monitoring::salt::client':  
 salt\_master =\> $salt\_master  
 }

class { "domains::cicd::jenkins-slave":  
 sensu\_host =\> $sensu\_host  
 }  
 }

'elk-server': {  
 class { 'domains::monitoring::salt::client':  
 salt\_master =\> $salt\_master  
 }

class { "domains::monitoring::elk::server" :  
 sensu\_host =\> $sensu\_host  
 }  
 }  
 }  
}  

```

With the exceptions of the master hostnames that we need to know about we can now spin up new machines, link them to puppet, and they become who they are. We store the role files in /etc/.config/role on the agents and our facter looks like this

```ruby
  
# node\_role.rb  
require 'facter'  
Facter.add(:node\_role) do  
 confine :kernel =\> 'Linux'  
 setcode do  
 Facter::Core::Execution.exec('cat /etc/.config/role')  
 end  
end  

```

Linking them up to puppet is easy too, we [wrote some scripts](https://gist.github.com/devshorts/e3d89b4e5dce9f145439) to help bootstrap machines since we don't want to have manually install puppet on each box, then configure its puppet.conf file to point to a master, etc. We want to just from our shell spin up new VM's in our openstack instance, and create new roles quickly. And by adding on zsh autocompletion we can get a really nice experience:

![Bootstrapping puppet agent](http://onoffswitch.net/wp-content/uploads/2015/08/Screen-Shot-2015-08-13-at-2.33.05-PM.png)

# Salt

[Saltstack](http://saltstack.com/) is an orchestration tool built on zeromq that we use to delegate tasks and commands. Basically anytime we need to execute a command on a machine, or role, or group of machines, we'll use salt. For example, let me ping all the machines in our salt group:

![Screen Shot 2015-08-13 at 3.24.12 PM](http://onoffswitch.net/wp-content/uploads/2015/08/Screen-Shot-2015-08-13-at-3.24.12-PM.png)

You can run arbitrary commands on machines too, or you can write your own custom python modules to execute (test.ping is a module called test with a method called ping that will get executed on all machines).

Leveraging salt, we can deploy our puppet scripts continuously trigged from a git repo change. When you commit into github a jenkins job is trigged. The jenkins job runs the puppet syntax validator on each puppet file:

```
  
#!/usr/bin/env bash

pushd data

for file in `find . -name "*.pp"`; do  
 echo "Validating $file"

puppet parser validate $file

if [$? -ne 0]; then  
 popd  
 exit 1;  
 fi  
done;

popd  

```

If that succeeds it dispatches a command to salt with a nodegroup of the puppet master. All this does is [execute a remote command](https://wiki.jenkins-ci.org/display/JENKINS/saltstack-plugin) (via the salt REST api) on the puppet master machine (or group of machines, we really dont care) and it checks out the git repo at a particular commit into a temp folder, blows out the environment and custom modules folders in /etc/puppet and then copies over the new files.

The nice thing here, is that even if we screwed up with our puppet scripts, we can still execute salt commands and roll things back with our jenkins job.

We take this salt deployment one step further for deploying applications in docker containers.

# Docker

My team is developing java services that deploy into openstack VMs running centos7. While we could push RPM's into spacewalk and orchestrate uninstalling and reinstalling rpms via puppet, I've had way too many instances in the past of config files getting left behind, or some other garbage mucking up my otherwise pristine environment. We also wanted a way to simulate the exact prod environment on our local machines. Building distributed services is easy, but debugging them is really hard, especially when you have lots of dependencies (rmq, redis, cassandra, decoupled listeners, ingest apis, health monitors, etc). For that reason we chose to package up our java service as a [shaded jar](https://maven.apache.org/plugins/maven-shade-plugin/) (we use [dropwizard](http://www.dropwizard.io/)) and package it into a docker image.

We built a base docker image for java services and then bundle our config, jar, and bootstrap scripts, into the base image. After our jenkins build of our service is complete, our package process takes the final artifact (which actually is an RPM, which is created using the maven RPM plugin), templates out a dockerfile, and creates a tar file with both files (RPM and Dockerfile) to artifact. This way we can use the [jenkins promotion plugin](https://wiki.jenkins-ci.org/display/JENKINS/Promoted+Builds+Plugin) to do the actual docker build. The tar file is there because jenkins promotions are asynchronous and can run on different slaves, so we want to include the raw artifacts as part of the artifacting for access later.

Here is an example of the templated dockerfile which will generates our docker image. The `--RPM_SRC--` and `--RPM_NAME--` placeholders get replaced during build

```
  
FROM artifactory/java-base

# add the rpm  
RUN mkdir /rpms

ADD --RPM\_SRC-- /rpms/

# install the rpm  
RUN yum -y install /rpms/--RPM\_NAME--

# set the service to run  
ENV SERVICE\_RUN /data/bin/service  

```

And below you can see the different jenkins promotion steps here. The first star is artifacting the docker image, and the second is deploying it to puppet via salt:

![Deploy services](http://onoffswitch.net/wp-content/uploads/2015/08/Screen-Shot-2015-08-13-at-2.51.58-PM.png)

Once we artifact the docker image, it's pushed out to artifactory. All our images are tagged with `app_name:git_sha` in artifactory. To deploy the docker image out into the wild we use salt to push the expected git sha for that application type onto the puppet master. I mentioned one custom facter that we wrote that nodes use to report their information back, but we also have a second.

Nodes store a json payload of their docker app arguments, the container version, and some other metadata, so that they know what they are currently running. They also report this json payload back to puppet, and puppet can determine if their currently running git\_sha is the same as the expected one. This way we know all their runtime arguments that were passed, etc. If anything is different then puppet stops the current container, and loads on the new one. All our containers are named, since it makes managing this easier (vs using anonymous container names). When we do update an application to the newest version we write that json back to disk in a known location. Here is the factor that collects all the docker version information for us:

```ruby
  
# Build a hash of all the different app jsons, located in /etc/.config/versions  
# there could be many docker applications running on this machine, which is why we bundle them all up together  
require 'facter'  
require 'json'  
Facter.add(:node\_info) do  
 confine :kernel =\> 'Linux'  
 setcode do  
 node\_information = {}  
 path = '/etc/.config/versions'

if File.exist?(path)  
 Dir.foreach(path) do |app\_type|  
 next if File.directory?(File.join(path, app\_type))  
 content = File.read(File.join(path, app\_type))  
 data\_hash = JSON.parse(content)  
 node\_information[app\_type] = data\_hash  
 end  
 end

node\_information  
 end  
end  

```

To ensure reboot tolerance and application restart tolerance our base docker image runs an instance of monit which is the foreground process for our application. We also make sure all docker containers are started with `--restart always` and that the docker service is set to start on reboot.

# Cluster swinging

We are still working on doing cluster swinging, but we'll leverage salt as well as a/b node roles in puppet to do that. For example, we could tag half of our RMQ listeners as A roles, and the other half B roles. When we go to publish we can publish by application type, so `RMQ_A_ROLE` will be an application type. Those can all switch over while the B roles stay the same. We can also leverage salt to do HAProxy swings. If we send a message to puppet to use a different haproxy config, then tell the haproxy box to update we've done part of a swing. Same in the reverse. All of this can be managed with jenkins promotion steps and can be undone by re-promoting an older step.

# Conclusion

This is just the beginning of our ops story on my team, as we have a lot of other work to do, but now that the basics are there we can ramp much faster. Whats interesting is that there are a lot of different varieties of production level tooling available, but no real consensus on how to do any of it. In my research I found endless conflicting resources and suggestions, and it seemed like every team (including mine!) was just sort if reinventing the process. I'm hoping that how we have things set up will work well, but our application is still in the major development phases so we have yet to really see how this will pan out in production (though our development cluster has 30 machines that we manage this way now).

And of course there are still issues. For example, we are using a puppet sensu plugin that keeps restarting the sensu process every 60 seconds. It's not the end of the world, but it is annoying. We had to write our own docker puppet module since the one out there didn't do what we wanted. The RMQ puppet module doesn't support FQDN mode for auto adding slaves, so we had to write that. We had to write tooling to help bootstrap boxes and deploy the puppet scripts, etc. While none of this is technologically challenging from a purely code perspective, it wasn't an easy process as most of your time spent debugging is "why the hell isn't this working, it SHOULD!" only to find you missed a comma somewhere (why can't we just do everything in strongly typed languages...). And of course, headaches are always there when you try something totally new, but in the end it was an interesting journey and gives me so much more appreciate for ops people. They never get enough credit.

