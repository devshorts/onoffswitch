---
layout: post
title: Scripting  deployment of clusters in asgard
date: 2016-06-08 00:38:19.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Snippets
tags:
- asgard
- aws
- cicd
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561497021;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:5000;}i:1;a:1:{s:2:"id";i:2274;}i:2;a:1:{s:2:"id";i:4737;}}}}

permalink: "/2016/06/08/scripting-deployment-clusters-asgard/"
---
We use asgard at work to do deployments in both qa and production. Our general flow is to check in, have jenkins build, an AMI is created, and then ... we have to manually go to asgard and deploy it. That sucks.

However, its actually not super hard to write some scripts to find the latest AMI for a cluster and prepare an automated deployment pipeline from a template. Here you go:

[code]  
function asgard(){  
 verb=$1  
 url="https://my.asgard.com/us-east-1/$2"  
 shift  
 http ${VERB} --verify=no "$url" -b  
}

function next-ami(){  
 cluster=$1

prepare-ami $cluster true | \  
 jq ".environment.images | reverse | .[0]"  
}

function prepare-ami(){  
 cluster=$1

includeEnv=$2

asgard GET "deployment/prepare/${cluster}?deploymentTemplateName=CreateAndCleanUpPreviousAsg&includeEnvironment=${includeEnv}"  
}

function get-next-ami(){  
 cluster=$1

next=`next-ami ${cluster} | jq ".id"`

prepare-ami ${cluster} "false" | jq ".lcOptions.imageId |= ${next}"  
}

function start-deployment(){  
 cluster=$1  
 payload=$2

echo $payload | asgard POST "deployment/start/${cluster}"  
}  
[/code]

The gist here is to

- Find the next AMI image of a cluster
- Get the prepared JSON for the next deployment
- Update the prepared json with the new ami image

To use it you'd do

[code]  
\> clusterName="foo"  
\> next=`get-next-ami $clusterName`  
\> start-deployment $clusterName $next  
{  
 "deploymentId": "1773"  
}  
[/code]

And thats it!

