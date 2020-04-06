---
layout: post
title: JIRA CLI Tooling
date: 2019-08-12 19:33:10.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- jira
- productivity
- tooling
meta:
  _edit_last: '1'
  _su_title: jira productivity cli
  _su_rich_snippet_type: none
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2019/08/12/jira-cli-tooling/"
---
<!-- wp:paragraph -->

I love tooling. I am too lazy to do things manually and whenever a process has impedance I have two choices, either don't do it, or make it better. I usually try and choose the latter.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

JIRA is the beast everyone loves to hate. Rightfully so frankly, it's slow, it's tedious, it's repetitive. But issue and project trackers like JIRA have _value_. As you become a more seasoned professional you realize that being able to see what you are working, what _others_ are working on, and the state of the team in general is huge. It means you can plan projects in advance, share status with stakeholders, allow people to see what you're doing and keep you from forgetting the things that are in play.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

While other trackers exist (fogbugz, trello, etc) JIRA is by far the most widely used. For the longest time I hated using JIRA, because it didn't map to my developer workflow. But lately I've been [using a collection of ruby scripts that wrap the go-jira CLI](https://github.com/devshorts/jira-cli-tooling) that makes my life a whole lot easier.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

For example, making a new ticket to track a random idea should be easy:

<!-- /wp:paragraph -->

<!-- wp:image {"id":8028,"width":608,"height":386} -->

<figure class="wp-block-image is-resized"><img src="https://onoffswitch.net/wp-content/uploads/2019/08/jira_new.gif" alt="" class="wp-image-8028" width="608" height="386"></figure>

<!-- /wp:image -->

<!-- wp:paragraph -->

This fires off a ticket into my teams backlog and gives me a link. Now whenever I think to myself "it'd be great to do xyz" I can just pop open the shell and fire that idea off.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

But we can take this so much further! I really like tying my commits to JIRA so I can see in the JIRA opened PR's, merged PR's, commits etc. To do that JIRA needs to integrate with github and know about your commits via (usually) a regex that looks for the jira ticket number in branch, commit, or PR titles.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

Since I'm too lazy to ever remember doing that, I wanted to have my branches auto named with the current JIRA I'm on. If we have the ticket in the branch name, we can use git commit hooks to auto format commit messages that are prefixed with the JIRA.

<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->

```
# file: .git/hooks/prepare-commit-message-std #!/bin/sh COMMIT\_MSG\_FILE=$1 COMMIT\_SOURCE=$2 SHA1=$3 function detect\_branch\_name() { # of the format username-JIRA-3586/description # get the branch, remove the username, # split by / and take first segment, # split by - and take second 2 segments git branch | grep \* | cut -d ' ' -f2 | sed 's/$USER-//' | cut -d '/' -f1 | cut -d '-' -f2,3 } branch\_name=`detect_branch_name` if [["$branch\_name" =~ "TRAFFIC-"]]; then data=`cat $COMMIT_MSG_FILE` echo "$branch\_name - $data" \> $COMMIT\_MSG\_FILE fi
```

<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->

Now if I make a commit like:

<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->

```
~/src (akropp-JIRA-123/title) $ git commit -am "foo"
```

<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->

My git log would look something like

<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->

```
JIRA-123 - foo
```

<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->

What if we could automatically create a JIRA and set it to in progress and add it to my sprint when I make a branch?

<!-- /wp:paragraph -->

<!-- wp:image {"id":8029} -->

<figure class="wp-block-image"><img src="https://onoffswitch.net/wp-content/uploads/2019/08/new_branch.gif" alt="" class="wp-image-8029"></figure>

<!-- /wp:image -->

<!-- wp:paragraph -->

Well we can! For example, we can make a new branch that

<!-- /wp:paragraph -->

<!-- wp:list {"ordered":true} -->

1. Creates a JIRA 
2. Adds it to my sprint
3. Marks it in progress
4. Git checkouts with a standardized branch name scheme that works with my commit hooks

<!-- /wp:list -->

<!-- wp:paragraph -->

All in a single one liner! On top of that making a new branch can pull down the open JIRA's I have and prompt me to start working on it if some exist! This way anytime I start a branch it's automatically tracked. All the random one-offs I do at work now get visibility and I can see at the end of the sprint how much work _I really did_ vs how much work we planned.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

On top of that, imagine I have open JIRAs and my branch names have the JIRA name in them. I can then switch between working tickets easily by resuming work on a ticket

<!-- /wp:paragraph -->

<!-- wp:image {"id":8030} -->

<figure class="wp-block-image"><img src="https://onoffswitch.net/wp-content/uploads/2019/08/resume.gif" alt="" class="wp-image-8030"></figure>

<!-- /wp:image -->

<!-- wp:paragraph -->

My code even works when I have multiple branches for the same JIRA, so if I have multiple logical flows for the same JIRA I can resume different subsections easily (great for large migrations!)

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

As always source code on my [github](https://github.com/devshorts/jira-cli-tooling)

<!-- /wp:paragraph -->

