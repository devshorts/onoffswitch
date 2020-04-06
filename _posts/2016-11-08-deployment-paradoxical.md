---
layout: post
title: Deployment the paradoxical way
date: 2016-11-08 08:00:47.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- cicd
- deploy
- maven
- travis
meta:
  _wpcom_is_markdown: '1'
  _edit_last: '1'
  _su_rich_snippet_type: none
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561836196;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4699;}i:1;a:1:{s:2:"id";i:4673;}i:2;a:1:{s:2:"id";i:4800;}}}}
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2016/11/08/deployment-paradoxical/"
---
First and foremost, this is all Jake Swensons brain child. But it's just too cool to not share and write about. Thanks Jake for doing all the hard work :)

At paradoxical, we really like being able to crank out libraries and projects as fast as possible. We _hate_ boilerplate and we _hate_ repetition. Everything should be automated. For a long time we used maven archetypes to crank out services from a template and libraries from a template, and that worked reasonably well. However, deployment was always kind of a manual process. We had scripts in each repo to use the maven release plugin but our build system (Travis) wasn't wired into it. This meant that deploys of libraries/services required a manual (but simple) step to run. We also had some kinks with our gpg keys and we weren't totally sure a clean way of having Travis be able to sign our artifacts in a secure way without our keys being checked into a bunch of different repos

Jake and I had talked a while ago about how nice it would be if we could

- Have all builds to master auto deployed as snapshots
- PR's built but not deployed
- Creating a github release kicked off an actual release

The first two were reasonably easy with the travis scripts we already had, but it was the last one that was fun.

[This article](https://axelfontaine.com/blog/dead-burried.html) was posted not long ago about simplifying your maven release process by chucking the maven release plugin and instead using the maven deploy directly. If you could parameterize your maven artifact version number and have your build pass that in from the git tag, then we could really easily achieve git tag driven development!

To that end, Jake created a [git project](https://github.com/paradoxical-io/deployment) that facilitated setting up all our repo's for tag driven deployment. Each deployable of ours would check out this project as a submodule under `.deployment` which contains the tooling to make git tag releases happen.

## To onboard

First things first, is that we need a way to delegate deployment after our travis build is complete. So you'd add the following to your projects travis file:

[code]  
git:  
 submodules: false  
before\_install:  
 # https://git-scm.com/docs/git-submodule#\_options:  
 # --remote  
 # Instead of using the superproject’s recorded SHA-1 to update the submodule,  
 # use the status of the submodule’s remote-tracking (branch.\<name\>.remote) branch (submodule.\<name\>.branch).  
 # --recursive  
 # https://github.com/travis-ci/travis-ci/issues/4099  
 - git submodule update --init --remote --recursive  
after\_success:  
- ./.deployment/deploy.sh  
[/code]

Which would pull the git deployment submodule, and delegate the after step to its deploy script.

You also need to add the deployment project as a parent of your pom:

[code]  
\<parent\>  
 \<groupId\>io.paradoxical\</groupId\>  
 \<artifactId\>deployment-base-pom\</artifactId\>  
 \<version\>1.0\</version\>  
\</parent\>  
[/code]

This sets up nice things for us like making sure we sign our GPG artifacts, include sources as part of our deployment, and attaches javadocs.

The last thing you need to do is parametarize your artifact version field:

`<version>1.0${revision}</version>`

The parent pom defines `revision` and will set it to be either the git tag or `-SNAPSHOT` depending on context.

But, for those of you with strong maven experience, an alarm may fire that you can't parameterize the version field. To solve that problem, Jake wrote a wonderful [maven parameter resolver](https://github.com/paradoxical-io/resolved-pom-maven-plugin) which lets you white-list which fields need to be pre-processed before they are processed. This solves an issue where a deployed maven pom that has parameterized values that are set at build time only are captured for deployment. Without that, maven has issues resolving transitive dependencies.

Anyways, the base pom handles a lot of nice things :)

## The deploy script

Now lets break down the after build [deploy script](https://github.com/paradoxical-io/deployment/blob/master/deploy.sh). It's job is to take the travis encrypted gpg keys (which are also password secured) and decrypt them, and run the right maven release given the git tags.

[code lang=text]  
if [-n "$TRAVIS\_TAG"]; then  
 echo "Deploying release version for tag '${TRAVIS\_TAG}'"  
 mvn clean deploy --settings "${SCRIPT\_DIR}/settings.xml" -DskipTests -P release -Drevision='' $@  
 exit $?  
elif ["$TRAVIS\_BRANCH" = "master"]; then  
 echo "Deploying snapshot version on branch '${TRAVIS\_BRANCH}'"  
 mvn clean deploy --settings "${SCRIPT\_DIR}/settings.xml" -DskipTests -P snapshot $@  
 exit $?  
else  
 echo "No deployment running for current settings"  
 exit 0  
fi  
[/code]

It's worth noting here a few magic things.

1. The [settings.xml](https://github.com/paradoxical-io/deployment/blob/master/settings.xml) file is provided by submodule and contains a field for the gpg username and the parametrized password the in every repo and contains the gpg user
2. Because the deploy script is invoked from the root of the project, even though the deploy script is in the deployment submodule it resolves paths from the script execution point (not where it the script lives at). This is why the script captures its own path and stores it as the `$SCRIPT_DIR` variable.

## Release time!

Now that it's all set up we can safely merge to master whenever we want to publish a snapshot, and if we want to mark a release as public we just create a git tag for it.

