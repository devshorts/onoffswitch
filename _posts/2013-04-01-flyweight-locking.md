---
layout: post
title: Flyweight Locking
date: 2013-04-01 08:00:57.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- c#
- synchronization
- threading
- wcf
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560570566;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:2447;}i:1;a:1:{s:2:"id";i:738;}i:2;a:1:{s:2:"id";i:2365;}}}}

permalink: "/2013/04/01/flyweight-locking/"
---
[Locking](http://en.wikipedia.org/wiki/Lock_(computer_science)) is a necessary aspect of multithreading code: it prevents unpredictable behavior and makes sure code that is expected to run synchronously does so. Some situations can leverage [lockless](http://yinsochen.com/thread-safe-and-or-lockless-data-structures/) code, but not always. When you do need to do a lock you shouldn't do it carelessly, if you lock a section of code that does some major work (such as database access) and it blocks other pending calls you need to be cognizant that there could be a delay or bottleneck. However, just because we have to lock doesn't mean we can't do some simple optimizations depending on what our business logic is. If we only need to lock items per a defined group then we can leverage flyweight locking. Lets go through an example to make this scenario clearer.

Imagine we have a WCF service that signs a student into a class where the student has a name, an id, and a classroom id that they belong to. Something like this:

[csharp]  
[ServiceContract]  
public interface ISchoolService  
{  
 [OperationContract]  
 void SignIntoClass(Student student);  
}

[DataContract]  
public class Student  
{  
 [DataMember]  
 public int ClassRoomNumber { get; set; }

[DataMember]  
 public string StudentName { get; set; }

[DataMember]  
 public int StudentId { get; set; }  
}  
[/csharp]

And our service implementation could be

[csharp]  
public class SchoolService : ISchoolService  
{  
 public void SignIntoClass(Student student)  
 {  
 if (!StudentStorage.Instance.IsStudentInClass(student))  
 {  
 StudentStorage.Instance.AddStudenToClass(student);  
 }  
 }  
}  
[/csharp]

Remember that entry point for this service is multi-threaded, the same student could log in from multiple locations simultaneously and that would add them to the class twice, since both threads could evaluate

[csharp]  
StudentStorage.Instance.IsStudentInClass(student)[/csharp]

as false if the student hadn't been added yet (assuming our internal storage calls weren't atomic or threadsafe).

We'd probably be inclined to just throw a lock statement around `SignIntoClass` using a static lock object for the class, but that locks every call. We can do better that that if we know how our data is grouped. If we only care about synchronizing students _per class_ then we can use what is called a [flyweight](http://en.wikipedia.org/wiki/Flyweight_pattern) lock and still be multi-threaded but synchronized.

A flyweight locking mechanism uses two sets of locks. One is a global lock, and one is a context lock. The global lock is used to synchronize getting context locks and the context locks are used to lock on the critical section for the action group. Lets add a flyweight lock to our student class and see what this really means

[csharp]  
public class SchoolService : ISchoolService  
{  
 private static readonly IDictionary\<int, object\> \_classroomLocks = new Dictionary\<int, object\>();

public void SignIntoClass(Student student)  
 {  
 object flyweightLock;

// lock everyone here on the global lock so you can get a local context lock  
 lock(\_classroomLocks)  
 {  
 if (!\_classroomLocks.TryGetValue(student.ClassRoomNumber, out flyweightLock))  
 {  
 flyweightLock = new object();  
 \_classroomLocks[student.ClassRoomNumber] = flyweightLock;  
 }  
 }

// now that we have a context lock we can lock our action group  
 // this is where the heavy processing happens  
 lock (flyweightLock)  
 {  
 if (!StudentStorage.Instance.IsStudentInClass(student))  
 {  
 StudentStorage.Instance.AddStudenToClass(student);  
 }  
 }  
 }  
}  
[/csharp]

Now what we're doing is getting a context level lock for the classrooms and using the dictionary as the global lock. All requests for a specific classroom are synchronized, but other classrooms can continue to do work even while one classroom could be busy inside of the lock.

