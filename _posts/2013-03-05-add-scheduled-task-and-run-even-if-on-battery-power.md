---
layout: post
title: Add scheduled task and run even if on battery power
date: 2013-03-05 22:04:26.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
- Snippets
tags:
- c#
- scheduled tasks
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561852696;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:155;}i:1;a:1:{s:2:"id";i:4394;}i:2;a:1:{s:2:"id";i:265;}}}}

permalink: "/2013/03/05/add-scheduled-task-and-run-even-if-on-battery-power/"
---
Just wanted to share a little helpful snippet in case anyone needs it. To add a scheduled task and make sure it starts even when on battery power do this:

[csharp]  
using (var taskService = new TaskService())  
{  
 TaskDefinition task = taskService.NewTask();

var action = new ExecAction  
 {  
 Path = "test.exe",  
 Arguments = "",  
 WorkingDirectory = ""  
 };

task.RegistrationInfo.Description = "Test";

var trigger = new TimeTrigger(DateTime.Now);  
 trigger.Repetition.Interval = TimeSpan.FromMinutes(5);

task.Triggers.Add(trigger);

task.Actions.Add(action);

task.Settings.DisallowStartIfOnBatteries = false;  
 task.Settings.StopIfGoingOnBatteries = false;

// Register the task in the root folder  
 taskService.RootFolder.RegisterTaskDefinition("test", task, TaskCreation.CreateOrUpdate, "SYSTEM", null, TaskLogonType.ServiceAccount, null);  
}  
[/csharp]

TaskService is part of [`Microsoft.Win32.TaskScheduler`](http://taskscheduler.codeplex.com/).

By default when you create a new task `DisallowStartIfOnBatteries` and `StartIfGoingOnBatteries` are true, so that's something to keep in mind if you are writing code that can be deployed on a laptop and you must have your scheduled task continue to run.

Quick side note, I personally think negative property names are hard to follow (`DisallowStartIfOnBatteries`). It's hard to follow when it becomes a double negative. I think it would've been much better to name the property

[code]  
AllowStartOnBatteries  
[/code]

Especially since it has nothing to do with starting the task when its not on batteries. It's interesting that the UI control for this doesn't use a double negative to display the context (even though the phrasing is logically inverse). Still, case in point

![taskScheduler.](http://onoffswitch.net/wp-content/uploads/2013/03/taskScheduler.-600x451.png)

