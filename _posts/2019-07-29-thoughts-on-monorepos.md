---
layout: post
title: Thoughts on monorepos
date: 2019-07-29 18:05:54.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Discussion
tags:
- architecture
- monorepo
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none

permalink: "/2019/07/29/thoughts-on-monorepos/"
---
<!-- wp:paragraph -->

For the past year I've been working at an organization that structures all their code in monorepos. A monorepo is basically a single version controlled repository that contains all of the organizations sourcecode. This is in contrast to many smaller repositories that contain either one (or a handful) of services/libraries etc.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

In nearly every place I've worked at monorepos are the defacto beginning of an organization. Companies start as single repo's and while a single repo doesn't mean a single service, it means that there is only one (usually) git repo to manage the entire companies engineering property. Each place I've been at this monorepo has grown organically, often times becoming a tangled mess of dependencies due to poor patterns and lack of contracts and encapsulation. After all, it takes significant effort to organize and structure a large codebase. It's almost an art.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

This tangled mess tends to lower productivity, lower morale, impede innovation and growth, and in general hamper scaling. It's been a popular choice to pull a Chernobyl sarcophagus move on the original repo. Expose endpoints with services and move to a micro-repo pattern where teams can own their own repo's independently and communicate with primary services via exposed APIs.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

On the one hand this has a lot of advantages. By ditching the legacy cruft of the old monorepo teams can pivot with new technologies, patterns, deployment mechanisms, etc. Each team becomes a startup in and of itself. Oftentimes there's a team or set of teams that manage shared libraries and infrastructure to empower this, so it's not like each time needs to rebuild the world from scratch (unless they want to). In a world of containerization, deployments can still be uniform. Especially when using orchestrators like ECS or K8. Even delegating builds into containers via Jenkins can allow infrastructure teams to abstract the runtime platform of any different repo. Teams can deploy at independent cadences because their repositories are independent. No other team would accidentally deploy their code.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

But there are issues. Upgrading libraries across N repos is complicated, and you often times get into interdependency conflicts depending on who is consuming what library at which version. The company ecosystem can be fragmented with different languages and different patterns at varying degrees of sophistication. Did every team remember to append trace logging to each request? Are all services using an up to date serialization library (remember equifax)? Moving between teams becomes harder as each team may use a different set of languages and tools.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

Given these problems a lot of companies have chosen to stick with the monorepo. Big famous companies use the monorepo and claim to have a lot of success with it. Google, Facebook, etc. The motivations are that when all the code is co-located it makes navigating dependencies and call graphs easier. You can do large migrations and theoretically migrate everyone to new patterns.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

But again, nothing is perfect. You still often times have the ball of spaghetti that companies started with. Except now there's no way to draw a line in the sand to either fix or move portions out. Many developers working in the same codebase all the time means you need to solve problems like constant conflicts, deployment ordering, accidental changes unrelated to tea commits, version control limitations, dealing with multiple languages, and having to handle different parts of the codebase using different dependencies.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

Some tools exist to help the monorepo. Bazel is a good example, developed by Google. Its hyper-explicit declarations allow you to manage cross language builds and inter dependencies. But it has an enormous learning curve, and most people find it bulky to use. The big companies tend to have teams dedicated to managing the monorepo: solving what happens when you reach git's limitations (which happened to Google), or how to speed up builds (Facebook doing distributed builds).

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

Even with tooling provided by the big players it's incredibly challenging to know what changed in a monorepo at any given point. What artifacts need to be packaged up? Do you build everything and run all the tests for all code? That might be incredibly costly for what could very well be a simple change in one area. What is the cost in developer time in navigating the repo? Building it local? Is that even possible? Does a repo of that size work with standard IDE's? Is it realistic to claim you can do repo wide refactoring when hundreds of developers are working in it at the same time?

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

There are a lot of workflow questions that need to be solved.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

Personally I shy away from monorepos. To me the complexity is just too great to gracefully scale with an organization. I think a lot of the benefits of monorepos can be simulated with things like git submodules (where logical groupings are independently versioned, but you can visualize and edit parts of the org together). Building a company with the ideal of individual repos means that people can experiment and pivot. Monorepos tend to encourage repeating the same patterns, _whether those patterns are good or not_.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

As companies scale you need to have teams dedicated to supporting other teams. I would rather teams specialize by language (and a company have a list of supported languages it allows) to build critical libraries and workflows. Enforcing upgrades of dependencies can be done by plugging into build tools and systems that contain minimum allowances of versions. Just as an example, [SBT can do](https://github.com/Verizon/sbt-blockade) exactly this.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

There's no magic bullet, in either model companies need to be prepared to invest a lot of time and energy in maintaining and growing their systems. But starting with a logical monorepo (and making sure to be very strict about abstractions and cross cutting contracts!) is a good starting place. Making sure infrastructure and tooling works for non-monorepos is critical for empowering growth as well though. Never make assumptions that the one repo will be the only repo in the future, because even in monorepo companies there are always exceptions the rule.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

<!-- /wp:paragraph -->

