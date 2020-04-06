---
layout: post
title: A toy generational garbage collector
date: 2016-02-20 20:55:03.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- gc
- java
- toy
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561521171;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3435;}i:1;a:1:{s:2:"id";i:3452;}i:2;a:1:{s:2:"id";i:4699;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2016/02/20/toy-generational-garbage-collector/"
---
Had a little downtime today and figured I'd make a toy generational garbage collector, for funsies. A friend of mine was once asked this as an interview question so I thought it might make for some good weekend practice.

For those not familiar, a common way of doing garbage collection in managed languages is to have the concept of multiple generations. All newly created objects go in gen0. New objects are also the most probably to be destroyed, as there is a lot of transient stuff that goes in an application. If an element survives a gc round it gets promoted to gen1. Gen1 doesn't get GC'd as often. Same with gen2.

A GC cycle usually consists of iterating through application root nodes (so starting at main and traversing down) and checking to see where a reference lays in which generation. If we're doing a gen1 collection, we'll also do gen0 and gen1. However, if you're doing gen0 only and a node is already laying in gen1, you can just bail early and say "_meh, this node and all its references are probably ok for now, we'll try this later_".

For a really great visualization checkout this [msdn article](http://blogs.msdn.com/b/abhinaba/archive/2009/03/02/back-to-basics-generational-garbage-collection.aspx) on generation garabage collection.

And now on to the code! First lets start with what is an object

[java]  
@Data  
@EqualsAndHashCode(of = "id")  
public class Node {  
 private final String id;

public Node(String id) {  
 this.id = id;  
 }

private final List\<Node\> references = new ArrayList\<\>();

public void addReference(Node node) {  
 references.add(node);  
 }

public void removeReference(Node node) {  
 references.removeIf(i -\> i.getId().equals(node.getId()));  
 }  
}  
[/java]

For the purposes of the toy, its just some node with some unique id.

Lets also define an enum of the different generations we'll support and their ordinal values

[java]  
public enum Mode {  
 Gen0,  
 Gen1  
}  
[/java]

Next, lets make an allocator who can allocate new nodes. This would be like a `new` syntax behind the scenes

[java]  
public class Allocator {  
 @Getter  
 private Set\<Node\> gen0 = new HashSet\<\>();

@Getter  
 private Set\<Node\> gen1 = new HashSet\<\>();

public Node newNode() {  
 return newNode("");  
 }

public Node newNode(String tag) {  
 final Node node = new Node(tag + UUID.randomUUID());

getGen0().add(node);

return node;  
 }

public Mode locateNode(Node tag) {  
 if (gen1.contains(tag)) {  
 return Mode.Gen1;  
 }

return Mode.Gen0;  
 }  
 ....  
[/java]

At this point we can allocate a new node, and assign nodes references.

[java]  
final Allocator allocator = new Allocator();

final Node root = allocator.newNode();

root.addReference(allocator.newNode());  
root.addReference(allocator.newNode());  
root.addReference(allocator.newNode());  
[/java]

Still haven't actually collected anything though yet. So lets write a garbage collector

[java]  
public class Gc {  
 private final Allocator allocator;

public Gc(Allocator allocator) {  
 this.allocator = allocator;  
 }

public void collect(Node root, Mode mode) {  
 final Allocator.Marker marker = allocator.markBuilder(mode);

mark(root, marker, mode);

marker.sweep();  
 }

private void mark(Node root, Allocator.Marker marker, Mode mode) {  
 final Mode found = allocator.locateNode(root);

if (found.ordinal() \> mode.ordinal()) {  
 return;  
 }

marker.mark(root);

root.getReferences().forEach(ref -\> mark(ref, marker, mode));  
 }  
}  
[/java]

The GC dos a DFS on the root object reference and marks all visible nodes with some marker builder (yet to be shown). If the generational heap that the node lives in is less than or equal to the mode we are on, we'll mark it, otherwise just skip it. This works because later we'll only prune from generation heaps according to the mode

Now comes the fun part, and its the marker

[java]  
public static class Marker {

private final Set\<String\> marks;  
 private final Allocator allocator;

private final Mode mode;

public Marker(Allocator allocator, final Mode mode) {  
 this.allocator = allocator;  
 this.mode = mode;  
 marks = new HashSet\<\>();  
 }

public void mark(Node node) {  
 marks.add(node.getId());  
 }

public void sweep() {  
 Predicate\<Node\> remove = node -\> !marks.contains(node.getId());

allocator.getGen0().removeIf(remove);

allocator.promote(Mode.Gen0);

switch (mode) {  
 case Gen0:  
 break;  
 case Gen1:  
 allocator.getGen1().removeIf(remove);  
 allocator.promote(Mode.Gen1);  
 break;  
 }  
 }  
}  
[/java]

All we do here is when we mark, tag the node in a set. When we go to sweep, go through the generations less than or equal to the current and remove unmarked nodes, as well as promote the surviving nodes to the next heap!

Still missing two last functions in the allocator which are promote and the marker builder

[java]  
public Marker markBuilder(final Mode mode) {  
 return new Marker(this, mode);  
}

private void promote(final Mode mode) {  
 switch (mode) {  
 case Gen0:  
 gen1.addAll(gen0);  
 gen0.clear();  
 break;  
 case Gen1:  
 break;  
 }  
}  
[/java]

Now we can put it all together and write some tests:

Below you can see the promotion in action.  
[java]  
final Allocator allocator = new Allocator();

final Gc gc = new Gc(allocator);

final Node root = allocator.newNode();

root.addReference(allocator.newNode());  
root.addReference(allocator.newNode());  
root.addReference(allocator.newNode());

final Node removable = allocator.newNode("remove");

removable.addReference(allocator.newNode("dangle1"));  
removable.addReference(allocator.newNode("dangle2"));

root.addReference(removable);

assertThat(allocator.getGen0().size()).isEqualTo(7);

gc.collect(root, Mode.Gen0);

assertThat(allocator.getGen0().size()).isEqualTo(0);  
assertThat(allocator.getGen1().size()).isEqualTo(7);  
[/java]

Nothing can be collected since all nodes have references, but we've cleared the gen0 and moved all nodes to gen1

[java]  
root.removeReference(removable);

gc.collect(root, Mode.Gen1);

assertThat(allocator.getGen0().size()).isEqualTo(0);  
assertThat(allocator.getGen1().size()).isEqualTo(4);  
[/java]

Now we can actually remove the reference and do a gen1 collection. You can see now that the gen1 heap size went down by 3 (so the removable node, plus its two children) since those nodes are no longer reachable

And just for fun, lets show that gen1 collection works as well

[java]  
final Node gen1Remove = allocator.newNode();

root.addReference(gen1Remove);

gc.collect(root, Mode.Gen1);

assertThat(allocator.getGen0().size()).isEqualTo(0);  
assertThat(allocator.getGen1().size()).isEqualTo(5);

root.removeReference(gen1Remove);

gc.collect(root, Mode.Gen1);

assertThat(allocator.getGen0().size()).isEqualTo(0);  
assertThat(allocator.getGen1().size()).isEqualTo(4);  
[/java]

And there you have it, a toy generational garbage collector :)

For the full code, check out this [gist](https://gist.github.com/devshorts/1d8d422790c6fc509760)

