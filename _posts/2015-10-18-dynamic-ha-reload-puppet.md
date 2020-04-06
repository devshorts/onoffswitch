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

permalink: "/2015/10/18/dynamic-ha-reload-puppet/"
---
<p>I've posted a little about puppet and our teams ops in the past since my team has pretty heavily invested in the dev portion of the ops role.  Our initial foray into ops included us building a pretty basic puppet role based system which we use to coordinate docker deployments of our java services. </p>
<p>We use HAProxy as our software load balancer and the v1 of our infrastructure managment had us versioning a hardcoded haproxy.cfg for each environment and pushing out that config when we want to add or remove machines from the load balancer. It works, but it has a few issues</p>
<ol>
<li>Cluster swings involve checking into github. This pollutes our version history with a bunch of unnecessary toggling</li>
<li>Difficult to automate swings since its flat file config driven and requires the config to be pushed out from puppet</li>
</ol>
<p>Our team did a little brainstorming and came up with a nice solution which is to data drive it from some sort of json blob.  By abstracting who provides the json blob and just building out our ha proxy config from structured data we can move to an API to serve this up for us.  Step one was to replace our haproxy.conf with some sort of flat file json. The workflow we have isn't changing, but its setting us up for success. Step two is to tie in something like consul to provide the json for us. </p>
<p>The first thing we need to do to support this is get puppet to know how to load up json from either a file or from an api. To do that we built an extra puppet custom function which we put into our <code>/etc/puppet/modules/custom/lib/puppet/functions</code> folder:</p>
<p>[ruby]<br />
require 'json'<br />
require 'rest-client'</p>
<p>module Puppet::Parser::Functions<br />
  newfunction(:json_provider, :type =&gt; :rvalue) do |args|</p>
<p>    begin<br />
      url=args[0]</p>
<p>      info(&quot;Getting json from url #{url}&quot;)</p>
<p>      if File.exists?(url)<br />
        raw_json = File.read(url)<br />
      else<br />
        raw_json = RestClient.get(url)<br />
      end</p>
<p>      data = JSON.parse(raw_json)</p>
<p>      info(&quot;Got json #{data}&quot;)</p>
<p>      data<br />
    rescue Exception =&gt; e<br />
      warning(&quot;Error accessing url #{url} from args '#{args}' with exception #{e}&quot;)</p>
<p>      raise Puppet::ParseError, &quot;Error getting value from url #{url} exception #{e}&quot;<br />
    end<br />
  end<br />
end<br />
[/ruby]</p>
<p>And we need to make sure the puppetmaster knows where all its gems are so we we've added</p>
<p>[code]<br />
 if ! defined(Package['json']) {<br />
    package { 'json':<br />
      ensure   =&gt; installed,<br />
      provider =&gt; 'gem'<br />
    }<br />
  }</p>
<p>  if ! defined(Package['rest-client']) {<br />
    package { 'rest-client':<br />
      ensure   =&gt; installed,<br />
      provider =&gt; 'gem'<br />
    }<br />
  }<br />
[/code]</p>
<p>To our puppet master role .pp.</p>
<p>At this point we can define what our ha proxy json file would look like. A sample structure that we've settled on looks like this:</p>
<p>[code]<br />
{<br />
  &quot;frontends&quot;: [<br />
    {<br />
      &quot;name&quot;: &quot;main&quot;,<br />
      &quot;bind&quot;: &quot;*&quot;,<br />
      &quot;port&quot;: 80,<br />
      &quot;default_backend&quot;: &quot;app&quot;<br />
    },<br />
    {<br />
      &quot;name&quot;: &quot;legacy&quot;,<br />
      &quot;bind&quot;: &quot;*&quot;,<br />
      &quot;port&quot;: 8080,<br />
      &quot;default_backend&quot;: &quot;app&quot;<br />
    }<br />
  ],<br />
  &quot;backends&quot;: [<br />
    {<br />
      &quot;name&quot;: &quot;app&quot;,<br />
      &quot;options&quot;: [<br />
        &quot;balance roundrobin&quot;<br />
      ],<br />
      &quot;servers&quot;: [<br />
        {<br />
          &quot;name&quot;: &quot;api1&quot;,<br />
          &quot;host&quot;: &quot;api1.cloud.dev:8080&quot;,<br />
          &quot;option&quot;: &quot;check&quot;<br />
        },<br />
        {<br />
          &quot;name&quot;: &quot;api2&quot;,<br />
          &quot;host&quot;: &quot;api1.cloud.dev:8080&quot;,<br />
          &quot;option&quot;: &quot;check&quot;<br />
        }<br />
      ]<br />
    }<br />
  ]<br />
}<br />
[/code]</p>
<p>Using this structure we can dynamically build out our haproxy.conf using ruby's erb templating that puppet hooks into. Below is our ha proxy erb template. It assumes that <code>@config</code> is in the current scope which should be a json object in the puppet file.  While the config is pretty basic, we don't use any ACLs or too many custom options, we can always tweak the base haproxy config or add more metadata to our json structure to support more options.</p>
<p>[ruby]<br />
#---------------------------------------------------------------------<br />
# Example configuration for a possible web application.  See the<br />
# full configuration options online.<br />
#<br />
#   http://haproxy.1wt.eu/download/1.4/doc/configuration.txt<br />
#<br />
#---------------------------------------------------------------------</p>
<p>#---------------------------------------------------------------------<br />
# Global settings<br />
#---------------------------------------------------------------------<br />
global<br />
    # to have these messages end up in /var/log/haproxy.log you will<br />
    # need to:<br />
    #<br />
    # 1) configure syslog to accept network log events.  This is done<br />
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in<br />
    #    /etc/sysconfig/syslog<br />
    #<br />
    # 2) configure local2 events to go to the /var/log/haproxy.log<br />
    #   file. A line like the following can be added to<br />
    #   /etc/sysconfig/syslog<br />
    #<br />
    #    local2.*                       /var/log/haproxy.log<br />
    #<br />
    log         127.0.0.1 local2</p>
<p>    chroot      /var/lib/haproxy<br />
    pidfile     /var/run/haproxy.pid<br />
    maxconn     4000<br />
    user        haproxy<br />
    group       haproxy<br />
    daemon</p>
<p>    # turn on stats unix socket<br />
    stats socket /var/lib/haproxy/stats level admin</p>
<p>#---------------------------------------------------------------------<br />
# common defaults that all the 'listen' and 'backend' sections will<br />
# use if not designated in their block<br />
#---------------------------------------------------------------------<br />
defaults<br />
    mode                    http<br />
    log                     global<br />
    option                  httplog<br />
    option                  dontlognull<br />
    option http-server-close<br />
    option forwardfor       except 127.0.0.0/8<br />
    option                  redispatch<br />
    retries                 3<br />
    timeout http-request    10s<br />
    timeout queue           1m<br />
    timeout connect         10s<br />
    timeout client          1m<br />
    timeout server          1m<br />
    timeout http-keep-alive 10s<br />
    timeout check           10s<br />
    maxconn                 3000</p>
<p>listen stats :1936<br />
    mode http<br />
    stats enable<br />
    stats hide-version<br />
    stats realm Haproxy\ Statistics<br />
    stats uri /<br />
    stats auth admin:password<br />
#---------------------------------------------------------------------<br />
# main frontend which proxys to the backends<br />
#---------------------------------------------------------------------<br />
&lt;% @config[&quot;frontends&quot;].each do |frontend| %&gt;<br />
frontend  &lt;%= frontend[&quot;name&quot;] %&gt; &lt;%= frontend[&quot;bind&quot;] %&gt;:&lt;%= frontend[&quot;port&quot;] %&gt;<br />
    default_backend             &lt;%= frontend[&quot;default_backend&quot;] %&gt;<br />
&lt;% end %&gt;<br />
#---------------------------------------------------------------------<br />
# backends<br />
#---------------------------------------------------------------------
\<% @config["backends"].each do |backend| %\>  
backend \<%= backend["name"] %\>  
 \<%- if backend["options"] != nil -%\>  
 \<%- backend["options"].each do |option| -%\>  
 \<%= option %\>  
 \<%- end -%\>  
 \<%- end -%\>  
 \<%- backend["servers"].each do |server| -%\>  
 server \<%= server["name"] %\> \<%= server["host"] %\> \<%= server["option"] %\>  
 \<%- end -%\>  
\<% end %\>  
[/ruby]

This builds out a simple set of named frontends that point to a set of backends. We can populate backends for the different swing configurations (A cluster, B cluster, etc) and then toggle the default frontend to swing.

But, we still have to provide for a graceful reload. There is [a lot](http://engineeringblog.yelp.com/2015/04/true-zero-downtime-haproxy-reloads.html) of [documentation](http://serverfault.com/questions/580595/haproxy-graceful-reload-with-zero-packet-loss) out there on this, but the gist is that you want to cause clients to retry under the hood while you restart, so that the actual requester of the connection doesn't notice a blip in service. To do that we can leverage the codified structure as well with another template

[ruby]  
#!/bin/bash

# hold/pause new requests  
\<% @config["frontends"].each do |frontend| %\>  
/usr/sbin/iptables -I INPUT -p tcp --dport \<%= frontend["port"] %\> --syn -j DROP  
\<% end %\>

sleep 1

# gracefully restart haproxy  
/usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /var/run/haproxy.pid -sf $(cat /var/run/haproxy.pid)

# allow new requests to come in again  
\<% @config["frontends"].each do |frontend| %\>  
/usr/sbin/iptables -D INPUT -p tcp --dport \<%= frontend["port"] %\> --syn -j DROP  
\<% end %\>  
[/ruby]

This inserts a rule for each frontend port to drop SYN packets silenty. SYN is the first packet type used in the [tcp 3 way handshake](https://en.wikipedia.org/wiki/Transmission_Control_Protocol#Connection_establishment) and by dropping it the client will retry a few times [after some interval](http://www.streppone.it/cosimo/blog/2011/07/how-to-detect-tcp-retransmit-timeouts-in-your-network/) to reconnect. This does mean the initial client will experience a slight delay, but their request will go through vs getting completely dropped.

Now our final haproxy.pp file looks like

[code]  
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
[/code]

With this, we can now drive everything from either a json file, or from a GET rest endpoint that provides JSON. We're planning on using consul as a simple key value store with an api to be able to drive the json payload. At that point our swings get the current json configuration, change the default endpoint for the frontned, post it back, and issue a puppet command to the ha proxies via salt nodegroups and we're all good!

