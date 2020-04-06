---
layout: post
title: Inter process locking
date: 2012-10-12 10:54:01.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
- Imported
tags:
- c#
- synchronization
meta:
  _syntaxhighlighter_encoded: '1'
  _edit_last: '1'
  dsq_thread_id: '853995183'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561472792;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:383;}i:1;a:1:{s:2:"id";i:390;}i:2;a:1:{s:2:"id";i:2447;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2012/10/12/inter-process-locking/"
---
_This article was originally published at [tech.blinemedical.com](http://tech.blinemedical.com/inter-process-locking/)_

Locking in a single-process multi-threaded application is important enough to understand, but locking in a multi-process application takes on a new level of complexity. [Locking](http://www.albahari.com/threading/part2.aspx#_Locking) makes sure that only one execution unit ever accesses a [critical section](http://en.wikipedia.org/wiki/Critical_section). This is just fancy way of saying everyone can't access the same resource at the same time; the critical section is the code path that is synchronized.

# Inter process locking

There are resources that can be accessed outside of the [logical address space](http://en.wikipedia.org/wiki/Virtual_address_space) of a process, such as files, and these are available to all processes. If you are writing a multiple process application, and are sharing these resources, you should synchronize them. For these situations, you should use a named mutex. A named mutex registers a global handle in the operating system that any process can request and use.

By giving a mutex a name, anyone can access it via its name. This is cool, but it's trickier than standard intra process locking (like using the `lock` statement on a reference object). If the mutex isn't properly handled, you can easily corrupt other programs that are expecting this mutex to function. So now instead of just crashing (or [deadlocking](http://en.wikipedia.org/wiki/Deadlock)) your program you can crash a bunch of others! You have to really take care and understand the locking mechanism to get this right.

Though most of the complexity in locking is abstracted away you can still run into issues if you don't handle your locks properly in .NET. We always look to encapsulate reusable, and especially complex, functionality into wrapper classes or utility functions so I created the [`InterProcessMutex`](https://github.com/devshorts/Inter-process-mutex/blob/master/InterProcessMutex/InterProcessMutexLock.cs) class that handles the major pitfalls of named mutexes:

- **Permissions**. One process can create a mutex that another process doesn't have access to.
- **Abandoned mutexes**. If a process or thread holds a mutex but doesn't release it and then exits, it will count as an [abandoned mutex](http://msdn.microsoft.com/en-us/library/system.threading.abandonedmutexexception.aspx)
- **Initial ownership**. It can be somewhat confusing as to who owns the mutex initially. The wrapper makes sure that nobody initially owns the mutex, so its open for the taking by anyone

All you have to do now is use the `InterProcessMutex` in a `using` block and it clearly indicates the critical section. Any process can instantiate the same lock and the wrapper takes care of the rest. Take a look at our [github](https://github.com/devshorts/Inter-process-mutex) for full source and unit tests.

[csharp]  
using (new InterProcessMutexLock(mutexName))  
{  
 // critical section  
}  
[/csharp]

Beyond using locking mechanisms built into the framework and helpful wrapper classes, it's also important to understand exactly how locking works (both intra- and inter-process locks). Since it's good to know what the magic behind the scenes is doing, we'll first go over a general definition of locking, and then delve into a couple different types of locks and how they are implemented. For anyone interested, [_Operating System Concepts_](http://www.amazon.com/Operating-Concepts-Seventh-Abraham-Silberschatz/dp/0471694665/ref=sr_1_1?s=books&ie=UTF8&qid=1348423387&sr=1-1&keywords=operating+systems+and+concepts+7th+edition) is a great book and I recommend you read it if you are curious about operating system algorithms. It's a fun read and has great easy to digest explanations with examples.

# Locking overview

Locking is a general term describing the solution to the critical section problem. The solution has to satisfy three conditions. I'm going to use the term `execution unit` to describe both process and threads.

- **Mutual exclusion**. If an execution unit is in a critical section, then no other execution unit can be in its critical section
- **Progress**. Execution units outside of their critical section can't block other execution units. If more than one execution unit wants to enter the critical section simultaneously, then there is some deterministic outcome of who can enter (i.e. nobody waits forever, someone gets to choose)
- **Bounded waiting**. Anyone waiting on a critical section will eventually be able to get in

# Locking by disabling preemption

In old style kernels, on single processor machines, the original way of doing critical sections was to disable [preemptive](http://en.wikipedia.org/wiki/Preemption_(computing)) [interrupts](http://en.wikipedia.org/wiki/Interrupt). Processors can dispatch interrupts that the kernel can catch, block any currently executing processes, and execute some unit of work before returning processes back to what they were doing. Basically, it's a "_hey, stop what you are doing, do this other thing, then go back to what you are doing_" kind of thing. When a critical section was going to be reached, the kernel paused all the interrupts. When the critical section was done it resumed them. This kind of sucks, though, because it would stop everything (like your clock) from getting updated. On multi-processor systems, which is most modern day computers, this isn't even a feasible solution since you really don't want to stop all interrupts from happening on all cores.

# Spin locks

Spin locking is a type of [busy wait](http://en.wikipedia.org/wiki/Busy_waiting) lock and is used by the kernel internally when there isn't going to be much contention. While it wastes CPU cycles, it saves on overhead in context switching and process rescheduling. It can also be implemented in a single space, user or kernel, so you save on space switching overhead. The downside is that if the spinlock is held for a long duration, it will be pretty wasteful. Just try putting in an empty `while(true);` in your code to see!

In a spinlock, the execution unit continually tests a condition to see if its true. If false, it continues with the critical section. If true, it then just keeps testing. In most modern architectures there are instructions that let us test and toggle a variable [atomically](http://en.wikipedia.org/wiki/Linearizability) (in one CPU instruction) which we can leverage to write a spinlock. There are two ways of doing this:

- [Test and Set](http://en.wikipedia.org/wiki/Test-and-set). This sets the value of the passed in address to a new value, but returns the original value of the address. i.e. If you pass in `&lock` that is set to 0, it will set `lock` to 1 and return 0.
- [Compare and Swap](http://en.wikipedia.org/wiki/Compare-and-swap). This is basically like test-and-set but only toggles the value if the testing address is the same as the input. Compare and swap is a more general version of test and set and is used in modern day architectures. We'll trace through this later in the post

# Test and Set

Test and set is an atomic function that generally looks like this (the function is atomic when it's executed at the processor level, not in c pseudocode)

[csharp]  
bool testAndSet(int \* lock){  
 int previousLockValue = \*lock;  
 \*lock = 1;  
 return previousLockValue == 1;  
}  
[/csharp]

And can be used to spinlock a critical section like below

[csharp]  
int lock = 0;

void synchroFunction(){  
 // check if we can aquire the lock  
 // spin wait here.

while (testAndSet(&lock)){  
 // do nothing  
 }

// critical section

// make sure to release the lock  
 // the next testAndSet will return false, reset the lock  
 // and exit the spinlock  
 lock = 0;  
}  
[/csharp]

Following the example, if it's not locked yet (initial lock is false), then the first execution unit acquires the lock and sets the lock to true. It also bails out of the while loop to execute its critical section, since it returned false from the `testAndSet` function (nobody held the lock). At this point it has the lock, and continues to have the lock, until it later sets the lock to false (which is usually an atomic function as well).

# Compare and Swap

In the [1970's](http://www.garlic.com/~lynn/2001e.html#73), compare-and-swap replaced test-and-set for most architectures and is still used today for lock-free algorithms as well as lock handling. It looks something like this (again remember this example is not atomic code, this is only atomic when this instruction is implemented in the cpu):

[csharp]  
compare\_and\_swap(int \*addr, int currentValue, int newVal){  
 int addressValue = \*addr;  
 if(addressValue == currentValue){  
 \*addressValue = newVal;  
 }  
 return addressValue;  
}  
[/csharp]

Compare and swap takes the address of an item storing the lock as well as a captured snapshot of whatever lock value an execution unit has and the expected new value. It only updates the lock reference if the captured value is equal to the value in the address.

[csharp]  
int lock = 0;

const int LOCKED = 1;

void synchroFunction(){

// check if we can aquire the lock  
 // spin wait here.

while (compare\_and\_swap(&lock, lock, LOCKED)){  
 // do nothing  
 }

// critical section

// make sure to release the lock  
 toggleLock(&lock);  
}  
[/csharp]

Lets trace it, remembering that the lock address only gets set if the passed in lock argument is the same as the address. If the initial value of `lock = 0`, the trace looks like this. Lets pretend the address of `lock` is `0xABC`  
[table]  
PROCESS,ARGUMENTS, \*(0xABC),RETURNS,END RESULT  
ProcessA,(0xABC 0 1), 1, 0, aquired lock  
ProcessB ,(0xABC 0 1), 1, 1, spins  
[/table]

_(`*(0xABC)` is the value at address 0xABC)_

Process A does the compare and swap, passing in what it thinks the value of the current lock is (0). At the same time, Process B executes compare and swap, also passing in the value of 0 for the lock. But only one of them gets to execute the instruction, since the instruction is atomic. Assuming Process A executed first, it sets the value at the address of lock (0xABC) to 1 and returns 0 (the original lock value). This means it acquired the lock and exits its spinlock, since 0 was returned. Then Process B executes its compare-and-swap and finds that the value at address lock (0xABC) is already 1, but it passed it the original value of 0, so it does NOT get to acquire the lock and returns the current value of the lock (1). It keeps spinwaiting.

In C# a compare-and-set equivalent is the [`Interlocked.CompareExchanged`](http://msdn.microsoft.com/en-us/library/bb297966.aspx) function.

# Spin locks without atomic instructions

On processors that didn't have atomic swap functions, spin locks were implemented using [petersons algorithm](http://en.wikipedia.org/wiki/Peterson's_algorithm). The idea here is you have an array keeping track of which process is ready to enter its critical section and a variable that is tracking who is actually in the critical section. Each execution unit only writes to its index in the array, so no contention here, and they all share the tracking variable. Eventually someone "grabs" the lock by both being ready and setting the tracker variable (by being the last to write to it). Here is a rough approximation of what that looks like. ProcessId is the current process.

In a two process example it looks like this.

[csharp]  
 // ready to be in the critical section  
 readyArray[currentProcessId] = true;

// let anyone else get into the critical section  
 turndId = otherProcessId;

while (readyArray[currentProcessId] == true && turndId == otherProcessId)  
 {  
 // busy wait  
 }

// critical section

...

// end of critical section. we're no longer ready to be in the section anymore  
 readyArray[currentProcessId] = false;  
[/csharp]

When a process who is ready to get into the critical section marks that its ready. The next variable `turnId` is the source of the contention. Someone is going to set it, but both won't be able to set it. Whichever write actually succeeds blocks the other process forcing it to go into a spinlock. When the acquired process is done, it'll toggle its `readyArray` value and the waiting process breaks out of its busy wait and executes.

# Mutexes

Mutexes accomplish the same goals as spinlocks, but differ in that they are an operating system provided abstraction that tells the OS to put a thread to sleep, instead of busy wait. With a mutex, threads/processes wait on a certain memory address using a [wait queue](http://www.helenos.org/doc/design/html.chunked/sync.html#id2531479). When the value at that address is changed, the OS wakes up all the waiting execution units and they can attempt to re-acquire a lock. They're more complicated to write, and I won't go into them. For more info read up on [futexes](http://lwn.net/Articles/360699/) in linux which are a good explanation of how to build mutexes.

# Locks in C#

Finally, we can briefly touch on the [`lock`](http://stackoverflow.com/questions/5111779/lock-monitor-internal-implementation-in-net) keyword. C# uses a [monitor,](http://stackoverflow.com/questions/301160/what-are-the-differences-between-various-threading-synchronization-options-in-c) which is basically a combination of kernel space mutexes and user space spinlocking to implement the `lock` keyword. [Internally](http://stackoverflow.com/questions/5111779/lock-monitor-internal-implementation-in-net), it uses the compare-and-swap atomic instruction to first try and aquire the lock, using a spinwait lock. If a thread sits in a spinwait for too long, then it can be switched over to use a mutex. This way it tries to gracefully level the playing field: fast locking if the lock isn't contended, but less cpu cycles if its going to wait too long in a spinlock.

# More information

For more reading check

- [Thin lock vs futex](http://bartoszmilewski.com/2008/09/01/thin-lock-vs-futex/) - By Bartosz Milewski
- [How are mutexes implemented](http://stackoverflow.com/questions/1485924/how-are-mutexes-implemented) - (stackoverflow question)
- [Spinlocking deadlock avoidance](http://www.bluebytesoftware.com/blog/2009/02/24/TheMagicalDuelingDeadlockingSpinLocks.aspx) - by Joe Duffy
- [Locks and hardware support](http://pages.cs.wisc.edu/~remzi/Classes/537/Spring2011/Book/threads-locks-hw.pdf) - chapter 27 from Remzi H. Arpaci-Dusseau and Andrea C. Arpaci-Dusseau's free online operating systems book
- [Efficiency of locking](http://attractivechaos.wordpress.com/2011/10/06/multi-threaded-programming-efficiency-of-locking/) - by Attractive Chaos (anonymous)
- [Effective implementations of spinlocks](http://software.intel.com/en-us/articles/effective-implementation-of-locks-using-spin-locks/) - via intel
