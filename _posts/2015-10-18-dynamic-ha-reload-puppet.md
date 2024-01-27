---
layout: post
title: Dynamic HAProxy configs with puppet
date: 2015-10-18 20:48:30.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- haproxy
- ops
- puppet
- ruby
- salt
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560772678;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4699;}i:1;a:1:{s:2:"id";i:4673;}i:2;a:1:{s:2:"id";i:2985;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2015/10/18/dynamic-ha-reload-puppet/"
---
I've posted a little about puppet and our teams ops in the past since my team has pretty heavily invested in the dev portion of the ops role.  Our initial foray into ops included us building a pretty basic puppet role based system which we use to coordinate docker deployments of our java services.

We use HAProxy as our software load balancer and the v1 of our infrastructure managment had us versioning a hardcoded haproxy.cfg for each environment and pushing out that config when we want to add or remove machines from the load balancer. It works, but it has a few issues

Cluster swings involve checking into github. This pollutes our version history with a bunch of unnecessary toggling

Difficult to automate swings since its flat file config driven and requires the config to be pushed out from puppet

Our team did a little brainstorming and came up with a nice solution which is to data drive it from some sort of json blob.  By abstracting who provides the json blob and just building out our ha proxy config from structured data we can move to an API to serve this up for us.  Step one was to replace our haproxy.conf with some sort of flat file json. The workflow we have isn't changing, but its setting us up for success. Step two is to tie in something like consul to provide the json for us.

The first thing we need to do to support this is get puppet to know how to load up json from either a file or from an api. To do that we built an extra puppet custom function which we put into our /etc/puppet/modules/custom/lib/puppet/functions folder:

```ruby
require 'json'
require 'rest-client'
module Puppet::Parser::Functions
newfunction(:json_provider, :type => :rvalue) do |args|
    begin
      url=args[0]
      info("Getting json from url #{url}")
      if File.exists?(url)
        raw_json = File.read(url)
      else
        raw_json = RestClient.get(url)
      end
      data = JSON.parse(raw_json)
      info("Got json #{data}")
      data
    rescue Exception => e
      warning("Error accessing url #{url} from args '#{args}' with exception #{e}")
      raise Puppet::ParseError, "Error getting value from url #{url} exception #{e}"
    end
end
end

```

And we need to make sure the puppetmaster knows where all its gems are so we we've added

```

if ! defined(Package['json']) {
    package { 'json':
      ensure   => installed,
      provider => 'gem'
    }
}
if ! defined(Package['rest-client']) {
    package { 'rest-client':
      ensure   => installed,
      provider => 'gem'
    }
}

```

To our puppet master role .pp.

At this point we can define what our ha proxy json file would look like. A sample structure that we've settled on looks like this:

```

{
"frontends": [
    {
      "name": "main",
      "bind": "*",
      "port": 80,
      "default_backend": "app"
    },
    {
      "name": "legacy",
      "bind": "*",
      "port": 8080,
      "default_backend": "app"
    }
],
"backends": [
    {
      "name": "app",
      "options": [
        "balance roundrobin"
      ],
      "servers": [
        {
          "name": "api1",
          "host": "api1.cloud.dev:8080",
          "option": "check"
        },
        {
          "name": "api2",
          "host": "api1.cloud.dev:8080",
          "option": "check"
        }
      ]
    }
]
}


```

Using this structure we can dynamically build out our haproxy.conf using ruby's erb templating that puppet hooks into. Below is our ha proxy erb template. It assumes that @config is in the current scope which should be a json object in the puppet file.  While the config is pretty basic, we don't use any ACLs or too many custom options, we can always tweak the base haproxy config or add more metadata to our json structure to support more options.

{% raw %}
```ruby


#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   http://haproxy.1wt.eu/download/1.4/doc/configuration.txt
#
#---------------------------------------------------------------------
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats level admin
#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
listen stats :1936
    mode http
    stats enable
    stats hide-version
    stats realm Haproxy\ Statistics
    stats uri /
    stats auth admin:password
#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------
<% @config["frontends"].each do |frontend| %>
frontend  <%= frontend["name"] %> <%= frontend["bind"] %>:<%= frontend["port"] %>
    default_backend             <%= frontend["default_backend"] %>
<% end %>
#---------------------------------------------------------------------
# backends
#---------------------------------------------------------------------
<% @config["backends"].each do |backend| %>
backend <%= backend["name"] %>
<%- if backend["options"] != nil -%>
<%- backend["options"].each do |option| -%>
<%= option %>
<%- end -%>
<%- end -%>
<%- backend["servers"].each do |server| -%>
server <%= server["name"] %> <%= server["host"] %> <%= server["option"] %>
<%- end -%>
<% end %>


```
{% endraw %}
This builds out a simple set of named frontends that point to a set of backends. We can populate backends for the different swing configurations (A cluster, B cluster, etc) and then toggle the default frontend to swing.

But, we still have to provide for a graceful reload. There is [a lot](http://engineeringblog.yelp.com/2015/04/true-zero-downtime-haproxy-reloads.html) of [documentation](http://serverfault.com/questions/580595/haproxy-graceful-reload-with-zero-packet-loss) out there on this, but the gist is that you want to cause clients to retry under the hood while you restart, so that the actual requester of the connection doesn't notice a blip in service. To do that we can leverage the codified structure as well with another template

{% raw %}
```ruby
  
#!/bin/bash

# hold/pause new requests  
<% @config["frontends"].each do |frontend| %>  
/usr/sbin/iptables -I INPUT -p tcp --dport <%= frontend["port"] %> --syn -j DROP  
<% end %>

sleep 1

# gracefully restart haproxy  
/usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /var/run/haproxy.pid -sf $(cat /var/run/haproxy.pid)

# allow new requests to come in again  
<% @config["frontends"].each do |frontend| %>  
/usr/sbin/iptables -D INPUT -p tcp --dport <%= frontend["port"] %> --syn -j DROP  
<% end %>  

```
{% endraw %}

This inserts a rule for each frontend port to drop SYN packets silenty. SYN is the first packet type used in the [tcp 3 way handshake](https://en.wikipedia.org/wiki/Transmission_Control_Protocol#Connection_establishment) and by dropping it the client will retry a few times [after some interval](http://www.streppone.it/cosimo/blog/2011/07/how-to-detect-tcp-retransmit-timeouts-in-your-network/) to reconnect. This does mean the initial client will experience a slight delay, but their request will go through vs getting completely dropped.

Now our final haproxy.pp file looks like

```
  
class custom::loadbalancers::dynamic\_ha(  
 $load\_balance\_path = undef,  
 $identity = undef # a unique seed to make sure the haproxy reloads dont stomp  
)  
{

if $load\_balance\_path == undef {  
 fail 'Pass in a load balance source path. Can be either a file on disk or a GET json url'  
 }

if $identity == undef {  
 fail "Identity for ha should be unique and set. This creates a temp file for reloading the haproxy gracefully"  
 }

package { 'haproxy':  
 ensure =\> installed  
 } -\>

service { 'haproxy':  
 enable =\> true,  
 ensure =\> running,  
 } -\>

package { 'haproxyctl':  
 ensure =\> installed,  
 provider =\> "gem"  
 }

$config = json\_provider($load\_balance\_path)

$rand = fqdn\_rand(1000, $identity)

$file = "/tmp/$identity-ha-reload.sh"

file { '/etc/haproxy/haproxy.cfg':  
 ensure =\> present,  
 mode =\> 644,  
 notify =\> Exec['hot-reload'],  
 content =\> template("custom/app/ha.conf.erb")  
 }

file { $file:  
 content =\> template("custom/app/ha\_reload.conf.erb"),  
 mode =\> 0755  
 } -\>  
 exec { 'hot-reload' :  
 require =\> File[$file],  
 command =\> $file,  
 path =\> "/usr/bin:/usr/sbin",  
 refreshonly =\> true  
 }  
}  

```

With this, we can now drive everything from either a json file, or from a GET rest endpoint that provides JSON. We're planning on using consul as a simple key value store with an api to be able to drive the json payload. At that point our swings get the current json configuration, change the default endpoint for the frontned, post it back, and issue a puppet command to the ha proxies via salt nodegroups and we're all good!

