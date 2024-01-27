---
layout: post
title: Testing puppet with docker and python
date: 2015-07-08 00:50:45.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- docker
- jenkins
- puppet
- python
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1556506239;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4699;}i:1;a:1:{s:2:"id";i:4737;}i:2;a:1:{s:2:"id";i:4978;}}}}

permalink: "/2015/07/08/testing-puppet-docker-python/"
---
In all the past positions I've been in I've been lucky enough to have a dedicated ops team to handle service deployment, cluster health, and machine managmenent. However, at my new company there is much more of a "self serve" mentality such that each team needs to handle things themselves. On the one hand this is a huge pain in my ass, since really the last thing I want to do is deal with clusters and machines. On the other hand though, because we have the ability to spin up openstack boxes in our data centers at the click of a button, each team has the flexibility to host their own infrastructrure and stack.

For the most part my team and I are deploying our java services using dockerized containers. Our container is a centos7 base image with a logstash forwarder in it and some other minor tooling, and we run our java service in the foreground. All we need to have on our host boxes is a bootloader script that we execute to shut down old docker containers and spin up new docker containers, and of course docker. To get docker and our bootloader (and of course manage things like our jenkins instances, RMQ clusters, cassandra nodes, etc) we are using puppet.

After deep diving into puppet my first question was "how do I test this?". Most suggestions indicate testing is two fold

1. Syntax checking
2. Integration testing on isolated machines

The first element is a no brainer. You run the puppet syntax checker and you get some output. That's not that helpful though, other than making sure I didn't fat finger something. And the second point really sucks. You have to manually check if everything worked. As an engineer I shudder at the word "manual", so I set out to create an isolated test framework that my team can use to simulate and automatically test puppet scripts both local and on jenkins.

To do that, I wrote [puppety](https://github.com/devshorts/Puppety). It's really stupidly simple. The gist is you have a puppet master in a docker container who auto signs anyone who connects, and you have a puppet agent in a docker container who connects, syncs, and then runs tests validating the sync was complete.

## Puppety structure

If you look at the git repo, you'll see there are two main folders:

```
  
/data  
/test  

```

The `/data` folder is going to map to the `/etc/puppet` folder on our puppet master. It should contain all the stuff we want to deploy as if we plopped that whole folder onto the puppet root.

The test folder contains the python test runners, as well as the dockerized containers for both the master and the agent.

## Testing a node

If you have a node configuration in an environment you can test a node by annotating it like so:

```
  
# node-test: jenkins/test-server  
node "test.foo.com" {  
 file {'/tmp/example-ip': # resource type file and filename  
 ensure =\> present, # make sure it exists  
 mode =\> 0644, # file permissions  
 content =\> "Here is my Public IP Address: ${ipaddress\_eth0}.\n", # note the ipaddress\_eth0 fact  
 }  
}  

```

Lets say this node sits in a definition file in `/etc/puppet/environments/develop/manifests/nodes/jenkins.pp`

Our test runner can pick up that we asked to test the jenkins node, and template our manifests such that during run time the actual node definition looks like

```
  
# node-test: jenkins/test-server  
node /docker\_host.\*/ {  
 file {'/tmp/example-ip': # resource type file and filename  
 ensure =\> present, # make sure it exists  
 mode =\> 0644, # file permissions  
 content =\> "Here is my Public IP Address: ${ipaddress\_eth0}.\n", # note the ipaddress\_eth0 fact  
 }  
}  

```

Now, when the dockerized puppet container connects, it assumes the role of the jenkins node!

The tests sit in a folder called tests/runners and the test name is the path to the test to run. It's that simple.

We are also structuring our puppet scripts in terms of roles. Roles using a custom facter who reads from `/etc/.role/role` to find out the role name of a machine. So this way, when a machine connects to puppet it'll say "I'm this role" and puppet can switch on the role to know what configurations to apply.

To support this, we can annotate role tests like so

```
  
node default {  
 case $node\_role{  
 # role-test: roles/slave-test  
 'slave': {  
 file {'/tmp/node-role': # resource type file and file  
 ensure =\> present, # make sure it exists  
 mode =\> 0644, # file permissions  
 content =\> "Here is my Role ${$node\_Role}.\n", # note the node role  
 }  
 }  
 # role-test: roles/listener-test  
 'listener': {  
 file { '/tmp/listener': # resource type file and file  
 ensure =\> present, # make sure it exists  
 mode =\> 0644, # file permissions  
 content =\> "I am a listener", # note the node role  
 }  
 }  
 }  
}  

```

When the `roles/slave-test` gets run the test runner will add the role `slave` to the right file, such that when the container connects it'll assume that role.

The tests themselves are trivial. They use `pytest` syntax and look like this:

[python]  
from util.puppet\_utils import \*

@agent  
def test\_file\_exists():  
 assert file\_exists("/tmp/example-ip")

@agent  
def test\_ip\_contents\_set():  
 assert contents\_contains('/tmp/example-ip', 'Here is my Public IP Address')

@master  
def test\_setup():  
 print "foo"  
[/python]

Tests are annotated by where they'll run at. Agent tests run after a sync, but master tests will run BEFORE the master runs. This is so you can do any setup on the master you need. Need to drop in some custom data before the agent starts? Perfect place to do it.

## Getting test results on jenkins

The fun part about this is that we can output the result of each test into a linked docker volume. Our jenkins test runner just looks like:

[shell]  
cd test

PATH=$WORKSPACE/venv/bin:/usr/local/bin:$PATH

virtualenv venv

. venv/bin/activate

pip install -r requirements.txt

python test-runner.py -e develop --all

python test-runner.py -e production --all  
[/shell]

And we can collect our results to get a nice test graph

![Screen Shot 2015-07-07 at 5.38.23 PM](http://onoffswitch.net/wp-content/uploads/2015/07/Screen-Shot-2015-07-07-at-5.38.23-PM.png)

To deploy we have cron job on the puppet master to pull back our puppet scripts git repo and merge the `data` folder into its /etc/puppet folder.

## Debugging the containers

Sometimes using puppety goes wrong and it's nice to see whats going on. Because each container exposes an entrypoint script we can pass in a debug flag to get access to a shell so we can run the tests manually:

```
  
$ docker run -it -h docker\_host -v ~/tmp/:/opt/local/tmp puppet-tests/puppet-agent --debug /bin/bash  

```

Now we can execute the entrypoint by hand, or run puppet by hand and play around.

## Conclusion

All in all this has worked really well for our team. It's made it easy for us to prototype and play with our infrastructure scripts in a controlled environment locally. And since we are able to now actually write tests against our infrastructure we can feel more comfortable about pushing changes out to prod.

