---
layout: post
title: Auto scaling akka routers
date: 2015-03-11 23:26:35.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- akka
- scala
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1555264548;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4627;}i:1;a:1:{s:2:"id";i:4456;}i:2;a:1:{s:2:"id";i:4596;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2015/03/11/auto-scaling-akka-routers/"
---
I'm working on a project where I need to multiplex many requests through a finite set of open sockets. For example, I have 200 messages, but I can only have at max 10 sockets open. To accomplish this I've wrapped the sockets in akka actors and am using an akka routing mechanism to "share" the 10 open sockets through a roundrobin queue.

This works out great, since now the consumers (who are rabbit mq listeners) just post messages to a facacde on the resource, and akka will route the request and do the appropriate work for me.

However, I wanted to know of a clean way to be able to add more resources (or remove them). Say at runtime I am asked to add 10 more open connections, or that suddenly we need to scale down to 5 connections. I'd like the router to be able to manage that for me.

It took a little poking around, but its not that complicated to do. The router manages a list of routees and you can pick a random one you want to remove (or add new ones). To remove one, send it a poison pill, and have the context unwatch it so the supervisor stops caring if it fails or not. Then tell the router to stop routing messages to it. When the poison pill reaches the actor (it'll finish processing its messages first) then it'll stop itself and you can do cleanup. In my case this is where I'd close the open socket.

A full scala example is here:

[scala]

import akka.actor.\_  
import akka.routing.\_

case class Add()

case class Remove()

class Worker(id: Integer) extends UntypedActor {  
 println(s"Made worker $id")

@throws[Exception](classOf[Exception]) override  
 def preStart(): Unit = {  
 println(s"Starting $id")  
 }

@throws[Exception](classOf[Exception]) override  
 def postStop(): Unit = {  
 println(s"Stopping $id")  
 }

@throws[Exception](classOf[Exception])  
 override def onReceive(message: Any): Unit = message match {  
 case \_ =\> println(s"Message received on actor $id")  
 }  
}

class Master extends Actor {

var count = 0

def makeWorker() = {  
 val id = count

count = count + 1

context.actorOf(Props(new Worker(id)))  
 }

var router = {  
 val startingRouteeNumber = 2

val initialRoutees = Seq.fill(startingRouteeNumber) {  
 val worker = makeWorker()  
 context watch worker  
 ActorRefRoutee(worker)  
 }

Router(RoundRobinRoutingLogic(), initialRoutees.toIndexedSeq)  
 }

def receive = {  
 case Remove =\>  
 println("Removing route")

val head = router.routees.head.asInstanceOf[ActorRefRoutee].ref

head ! PoisonPill

context unwatch head

router = router.removeRoutee(head)

printRoutes()

case Add =\>  
 println("Adding route")

val worker = makeWorker()

context watch worker

router = router.addRoutee(worker)

printRoutes()

case w: AnyRef =\>

printRoutes()

router.route(w, sender())  
 }

def printRoutes(): Unit = {  
 val size = router.routees.size

println(s"Total routes $size")  
 }  
}

object Main extends App {  
 var system = ActorSystem.create("foo")

var master = system.actorOf(Props[Master])

master ! "do work"

master ! Remove

master ! "do more work"

master ! "do even more work"

master ! Add

master ! "do work again"  
}  
[/scala]

