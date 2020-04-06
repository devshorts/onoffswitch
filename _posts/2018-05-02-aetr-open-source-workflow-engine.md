---
layout: post
title: AETR an open source workflow engine
date: 2018-05-02 17:37:43.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- aetr
- postgres
- scala
meta:
  _wpcom_is_markdown: '1'
  _edit_last: '1'
  _su_rich_snippet_type: none
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560876621;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4699;}i:1;a:1:{s:2:"id";i:4737;}i:2;a:1:{s:2:"id";i:4394;}}}}
  _wpas_done_all: '1'

permalink: "/2018/05/02/aetr-open-source-workflow-engine/"
---
For the past several years I've been thinking about the idea of an open source workflow execution engine. Something like AWS workflow but simpler. No need to upload python, or javascript, or whatever. Just call an API with a callback url, and when the API completes its step, callback to the coordinator with a payload. Have the coordinator then send that payload to the next step in the workflow, etc.

This kind of simplified workflow process is really common and I keep running into it at different places that I work at. For example, my company ingests client catalogs to augment imagery with their SKU numbers and other metadata. However that ingestion process is really fragmented and asynchronous. There's an ingestion step, following that there is a normalization step, then a processing step, then an indexing step, etc. In the happy case everyone is hooked together with a queue pipeline where once ingestion is done it publishes a message to the normalizer, etc. But what happens when you want to take the principal components of this pipeline and create an adhoc pipeline? We don't necessarily want the ingestor to always write to the normalizer. It'd be great to be able to compose these steps into different step trees that can own their own flow.

This is what [AETR](https://github.com/paradoxical-io/aetr) lets you do.

## How it works

The primary building blocks in AETR are

1. A step tree
2. A run tree

A step tree is literally a tree structure that represents what a sequence of steps is. Leaf nodes in the tree are all API actions, and parent nodes in the tree are either a sequential or a parallel parent. What this means is you can have trees like this:

[code lang=text]  
Sequential  
 |- Sequential  
 |- API1  
 |- API2  
 |- Parallel  
 |- API3  
 |- API4  
[/code]

In this tree the root is sequential, which means its child nodes must run... sequentially. The first child is also a sequential parent, so the ordering of this node is the execution of `API1` followed by `API2` when `API1` completes. When that branch completes, the next branch can execute. That branch is parallel, so both `API3` and `API4` execute concurrently. When both complete, the final root node is complete!

Int the nomenclature of AETR when you go to run a step tree, it becomes a run tree. A run tree is the same tree as a step tree but includes information such as state, timing, inputs/outputs, etc. For example:

[code lang=scala]  
case class Run(  
 id: RunInstanceId,  
 var children: Seq[Run],  
 rootId: RootId,  
 repr: StepTree,  
 executedAt: Option[Instant] = None,  
 completedAt: Option[Instant] = None,  
 version: Version = Version(1),  
 createdAt: Instant = Instant.now(),  
 updatedAt: Instant = Instant.now(),  
 var parent: Option[Run] = None,  
 var state: RunState = RunState.Pending,  
 var input: Option[ResultData] = None,  
 var output: Option[ResultData] = None  
)  
[/code]

## DB layer

Run trees are stored in a postgres DB and are easy to reconstitute from a storage layer. Since every row contains the root, we can in one DB call get all the nodes for a run tree and then rebuild the graph in memory based on parent/child links.

Step trees related to run trees are a bit more complicated to rebuild since step trees can point to other step trees. To rebuild a step tree there's a step tree table which contains each individual step node as a row in the db. And there is also a table called `step_children` which relates a parent to its ordered set of children. We need a children link instead of a parent link for the reason described above. Step trees can be modified to link to other trees, and they can be re-used in many composable steps. This means that there's no clear parent of a tree, since the action of `API1` can be re-used in many different trees.

Here's an example of rebuilding a step tree:

[code lang=scala]  
def getStep(stepTreeId: StepTreeId): Future[StepTree] = {  
 val idQuery = sql"""  
 WITH RECURSIVE getChild(kids) AS (  
 SELECT ${stepTreeId}  
 UNION ALL  
 SELECT child\_id FROM step\_children  
 JOIN getChild ON kids = step\_children.id  
 )  
 SELECT \* FROM getChild""".as[StepTreeId]

val nodesQuery = for {  
 ids \<- idQuery  
 treeNodes \<- steps.query.filter(\_.id inSet ids).result  
 treeChildren \<- children.query.filter(\_.id inSet ids).result  
 } yield {  
 (treeChildren, treeNodes)  
 }

provider.withDB(nodesQuery.withPinnedSession).map {  
 case (children, nodes) =\>  
 val allSteps = composer.reconstitute(nodes, children)

allSteps.find(\_.id == stepTreeId).get  
 }  
 }  
[/code]

We can use a recursive CTE in postgres to find all the children starting at a given tree id, then we can slurp those childrens identities and rebuild the graph in memory.

Storing the children in a separate table also has an advantage that parent are child aware. Why does this matter? Well AETR wouldn't be as useful as it is if all it did was strictly call API's. We need a way to _transform_ payloads between calls and we need a way to _reduce_ parallel calls into a singular output, so that nodes can be composed. This matters because assume that `API1` returns some json shape, and `API2` requires a different json shape as its input. If we hooked `API`` ->`API2`directly it'd never work. There needs to be a _mapper_. But mapping functions are only related to their relative placement in the graph. If we rehook`API1`->`API3` now it may need a different mapping function. To that end you can't store mappers directly on step nodes themselves, it has to be on the _child_ relationship.

On top of that we have the concept of reducers in AETR. Parallel parents can take the result set of all their children and reduce the payloads into one result.

Lets look at a concrete example:

![null](https://github.com/paradoxical-io/aetr/raw/master/wiki:img/root_tree_1.png)

Here's a root tree that does things in parallel and has some sequential sub nodes.

If we look at one of the parallel parents we can see how to reduce the data:

![](https://github.com/paradoxical-io/aetr/raw/master/wiki:img/parallel_parent.png)

Much the same way we can see how to map data between nodes for sequential parents

![](https://github.com/paradoxical-io/aetr/raw/master/wiki:img/sequential_parent.png)

Mappers and reducers are executed in a sandboxed nashorn engine.

## Concurrency

It's important in AETR to make sure that as we work on these trees that concurrent access doesn't introduce race conditions. AETR internally supports some optimistic locking on the trees as well as atomic version updates to prevent any concurrency issues.

## Example

Lets take a look at a full flow!

![](https://github.com/paradoxical-io/aetr/raw/master/wiki:img/demo.gif)

In this example we run the full tree and we can see the inputs and outputs of the data as they are mapped, and finally reduced. When the entire tree is complete the root node of the tree contains the _final_ data. In this way the root is the final result and can be used to programmatically poll the AETR api.

Give AETR a shot and please feel free to leave feedback here or in the github issues!

