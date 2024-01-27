---
layout: post
title: Chaos monkey for docker
date: 2018-02-16 23:47:25.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- fault tolerance
- ruby
- service oriented architecture
meta:
  _wpcom_is_markdown: '1'
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561472557;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4699;}i:1;a:1:{s:2:"id";i:4673;}i:2;a:1:{s:2:"id";i:5000;}}}}
  _wpas_done_all: '1'
  _jetpack_dont_email_post_to_subs: '1'

permalink: "/2018/02/16/chaos-monkey-docker/"
---
I work at a mostly AWS shop, and while we still have services on raw EC2, nearly all of our new development is on Amazon ECS in docker. I like docker because it provides a unified unit of operation (a container) that makes it easy to build shared tooling regardless of language/application. It also lets you reproduce your applications local in the same environment they run remote, as well as starting fast and deploying fast.

However, many services run on a shared ECS node in a cluster, and so while things like Chaos Monkey may run around turning nodes off it'd be nice to have a little less of an impact during working hours while still being able to stress recovery and our alerting.

This is actually pretty easy though with a little docker container we call `The Beast`. All the beast does is run on a ECS Scheduled event every 15-30 minutes from 10am - 3pm PST (we have teams east and west coasts) and the beast kills a random container from whatever cluster node its on. It doesn't do a lot of damage, but it does test your fault tolerance.

Here's The Beast:

```ruby
  
#!/usr/bin/env ruby

require 'json'  
require 'pp'

class Hash  
 def extract\_subhash(\*extract)  
 h2 = self.select{|key, value| extract.include?(key) }  
 self.delete\_if {|key, value| extract.include?(key) }  
 h2  
 end  
end

puts "UNLEASH THE BEAST!"

ignore\_image\_regex = ENV["IGNORED\_REGEX"]

raw = "[#{`docker ps --format '{{json .}}'`.lines.join(',')}]"

running\_services = JSON.parse(raw).map { |val| val.extract\_subhash("ID", "Image")}

puts running\_services

puts "Ignoring regex #{ignore\_image\_regex}"

if ignore\_image\_regex && ignore\_image\_regex.length \> 0  
 running\_services.delete\_if {|value|  
 /#{ignore\_image\_regex}/ === value["Image"]  
 }  
end

if !running\_services || running\_services.length == 0  
 puts "No services to kill"

Process.exit(0)  
end

puts "Bag of services to kill: "

to\_kill = running\_services.sample

puts "Killing #{pp to\_kill}"

`docker kill #{to_kill["ID"]}`

prng = Random.new

quips = [  
 "Dont fear the reaper",  
 "BEAST MODE",  
 "You been rubby'd",  
 "Pager doody"  
]

puts "#{quips[prng.rand(0..quips.length-1)]}"  

```

Beast supports a regex of ignored images (so critical images like the ecs\_agent and itself) can be marked as ignore. This can also be used to update the beast to allow it to ignore services temporarily/etc.

We deploy The Beast with terraform, the general task definition looks like:

```
  
[  
 {  
 "name": "the-beast",  
 "image": "${image}:${version}",  
 "cpu": 10,  
 "memory": 50,  
 "essential": true,  
 "logConfiguration": {  
 "logDriver": "awslogs",  
 "options": {  
 "awslogs-group": "${log\_group}",  
 "awslogs-region": "${region}",  
 "awslogs-stream-prefix": "the-beast"  
 }  
 },  
 "environment": [  
 {  
 "name": "IGNORED\_REGEX", "value": ".\*ecs\_agent.\*|.\*the-beast.\*"  
 }  
 ],  
 "mountPoints": [  
 { "sourceVolume": "docker-socket", "containerPath": "/var/run/docker.sock", "readOnly": true }  
 ]  
 }  
]  

```

And the terraform:

```
  
resource "aws\_ecs\_task\_definition" "beast\_rule" {  
 family = "beast-service"  
 container\_definitions = "${data.template\_file.task\_definition.rendered}"

volume {  
 name = "docker-socket"  
 host\_path = "/var/run/docker.sock"  
 }  
}

data "template\_file" "task\_definition" {  
 template = "${file("${path.module}/files/task-definition.tpl")}"

vars {  
 version = "${var.beast-service["version"]}"  
 region = "${var.region}"  
 image = "${data.terraform\_remote\_state.remote\_env\_state.docker\_namespace}/the-beast"  
 log\_group = "${var.log-group}"  
 }  
}

resource "aws\_cloudwatch\_event\_target" "beast\_scheduled\_job\_target" {  
 target\_id = "${aws\_ecs\_task\_definition.beast\_rule.family}"  
 rule = "${aws\_cloudwatch\_event\_rule.beast\_scheduled\_job.name}"  
 arn = "${data.aws\_ecs\_cluster.default\_cluster.id}"  
 role\_arn = "${data.aws\_iam\_role.ecs\_service\_role.arn}"  
 ecs\_target {  
 task\_count = 1  
 task\_definition\_arn = "${aws\_ecs\_task\_definition.beast\_rule.arn}"  
 }  
}

resource "aws\_cloudwatch\_event\_rule" "beast\_scheduled\_job" {  
 name = "${aws\_ecs\_task\_definition.beast\_rule.family}"  
 description = "Beast kills a container every 30 minutes from 10AM to 3PM PST Mon-Thu"  
 schedule\_expression = "cron(0/30 18-23 ? \* MON-THU \*)"  
 is\_enabled = false  
}

resource "aws\_cloudwatch\_log\_group" "beast\_log\_group" {  
 name = "${var.log-group}"  
}  

```

We can log to cloudwatch and correlate back information if a service was killed by the best as well. It's important to note that you need to mount the docker socket for beast to work, since it needs docker to run. A sample dockerfile looks like:

```
  
FROM ubuntu:xenial

RUN apt-get update && apt-get install -y ruby-full docker.io build-essential

RUN gem install json

ADD beast.rb /app/beast.rb

RUN chmod +x /app/beast.rb

ENTRYPOINT "/app/beast.rb"  

```

It's bare bones, but it works, and the stupid quips at the end always make me chuckle.

