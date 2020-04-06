---
layout: post
title: Run with real data
date: 2012-10-29 17:29:24.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
- Imported
tags:
- Best Practices
meta:
  _edit_last: '1'
  dsq_thread_id: '880047619'
  _wp_old_slug: development-datasets
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1559148092;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:1268;}i:1;a:1:{s:2:"id";i:4800;}i:2;a:1:{s:2:"id";i:4028;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2012/10/29/run-with-real-data/"
---
_This article was originally published at [tech.blinemedical.com](http://tech.blinemedical.com/run-with-real-data/)_

More often than not the test data in development environments is full of garbage. Most applications have a baseline set of test data, just enough to get the database to function. As engineers develop they tend to make heavy use of "_asdf_" and "_dfkkfklkasdflsaf_," combined with a liberal sprinkling of [lorem ipsum](http://en.wikipedia.org/wiki/Lorem_ipsum) and other nonsense, to populate new data. This is fine to get started, since frequently as you develop you need to wipe the dataset clean and start over but this false data gives an incorrect view of the applications interface and performance. No client is going to have 5 "_asdf_" fields as their data. Instead, visible fields get stressed and assumptions about how your application handles data are challenged. You may not have expected a particular combobox to display a 200 character item, but that's what the client needs. Maybe you didn't expect a certain list page to grow to 20,000 items, so you never paginated or set default filters. But, that's the data the client created, and you need to account for it.

An application that is sleek and zippy at first, can become a monstrous behemoth when faced with data sets it doesn't expect to handle. It can freeze or crash, but even worse is the "slow death." The app loses speed slowly, over time, and becomes frustrating or annoying to use.

There are two sides to the story here; testing with empty data sets, and testing with real world large data sets (if you can get a hold of them).

# Clean Data Sets

Clean data sets expose a specific set of problems relating to the client's first application experience. It should have the bare minimum of what a client will see when they first install, or start using your application. As you're developing, it's easy to forget what things are like for a client. You work in a world of test data, but the clean data set can expose all sorts of small things: something didn't align properly, some box isn't populated with a "None" entry, all that empty space looks goofy and should be dynamically sized, or countless other minor details. These kinds of small bugs make for a bad user experience. Clean data sets expose the way your app works in the absence of data, and that's important.

The absence of user data also can expose the kind of default data you should be shipping with your app. Is your usage that every time you load up the app with a clean data set, you create item x,&nbsp; y, and z? Are these common items? Will a client appreciate these things being pre-populated? If so, you should include them. Casual users appreciate default values, since it can get them up and running quickly without the need for boilerplate. Maybe you should offer batch input functionality, so the client can quickly go from zero to usable. Working with a constant set of test data will never reveal these things, you only notice them when you start fresh.

Clean data can also mean a clean install. If you are working in your development environment, or even some testing environment, make sure to fully wipe the target test machines. Go so far as reinstalling the operating system, and start from scratch. Is there some install step that you have to set a registry key by hand? What about setting permissions on a folder for some service account? You would've long since forgotten what you did, since incremental deployments don't have to do those steps. Starting fresh every so often is always a good idea.

# Large Data Sets

While the clean data set matters for new users, what matters for keeping users in your app is testing the large data set. Maybe there is a power user who is using your app more than an average user, and generating tons of data. You need to account for this. Do you have automation functionality that can that be leveraged to create lots of data? If so, you should certainly be developing and testing your application with that same data. This is where you'll really notice big performance problems. Service calls can grind to a halt, display pages take a long time to render, race conditions are exposed, bottlenecks uncovered, etc. There are numerous tools available on the internet to help generate realistic data. [This one](http://www.fakenamegenerator.com/order.php), for example, creates an identity complete with credit card, social security number, birthdate, height, etc. and lets you order in bulk for free (up to 50,000 users).

It's one thing to use a large data set of test data you generate, but it's another to get a hold of real client data. A client can slowly generate data over years of use, and it's extremely difficult to mimic that much real data in a controlled QA environment. When you use real client data, you can almost immediately find aggravation points and quickly address them. Does the app take longer than normal to load? Maybe the user thinks that's normal, but you know it's not. Do visual elements still work properly with the client data set? Do things need to be tweaked so you can see the data better, and the client can more easily do their work? The client may never have reported an issue but just because they never reported it doesn't mean it's not there.

Client data also can have missing data in areas you wouldn't expect. Test data often fills in all the blanks, but clients could be focusing heavily in an area that wasn't originally designed to be used that heavily, or vice versa, they aren't using an area designed for heavy load. These are details you only find when running "[as the client does](http://en.wikipedia.org/wiki/Eating_your_own_dog_food)".

# Client data and privacy

When using client data, you can come across sensitive personal information like credit card numbers, addresses, and [health records](http://www.hhs.gov/ocr/privacy/index.html). You should take great care to [obfuscate](http://en.wikipedia.org/wiki/Data_masking) this data before ever using it. At the same time, you should strive to preserve data integrity, since completely obfuscated data can be meaningless or invalid. There's a balancing act here, but in the end it's more important to respect the privacy of clients. There are a few ways to do this, and all of these things can be automated.

- **Full masking**. Here, you would replace all words and numbers with other random words and numbers. You want to preserve capitalization, punctuation, and word length, but you can replace words with garbage. This gives you an indication of length, format, and usage. Don't just replace them randomly, though. If you find a word, create a replacement for it and keep track of it. If you see the same word again, use the same replacement. This can tell you word frequency, and if something is being continually re-iterated by the client. These patterns can also help identify areas to automate client actions.
- **Partial masking**. With partial masking, you can just do sensitive areas such as usernames, addresses, phone numbers, statistics (such as test scores or health records) and any other kind of personal identifying information. Doing partial masking maintains data context, from which you can infer client intentions.
- **Adjust dates**. Instead of using the actual date, offset all date groupings by a random time. By doing this you can maintain date relationships (i.e. if A and B are related and happened at 10 minutes apart, you will maintain that relationship) but you don't need to know what was the original A and B. Offsetting all related groups gives you meaningful, but at the same time obfuscated, dates.
- **Network addresses**. This is an easy one to overlook. If you store network addresses anywhere, you should change them to point to known local machines, or make them invalid. If your client is open via publicly accessible routes, or even through an internal provided VPN, you don't want your development or test machines to accidentally contact their computers and apply edits.
- **Encryption**. If possible, encrypt your client data when you store it. If your machines get compromised you don't want to have accidentally also compromised your clients data.

# Application under load

If you use a large data set you should simulate client load scenarios. The application may function wonderfully when only one or two people use it, assuming a shared distributed app. What happens when 200 people hammer on it at once? What about 2000? Is it possible for a client to have a virus scanner running? Is it thrashing the disk? Can we make optimizations that help these scenarios? Can we make these optimizations configurable? I frequently find myself writing test-apps that spawn multiple threads, and do actions at some insane interval (like every 10ms). This way, you can test areas of the application thoroughly for performance and reliability of your system.

I think it's also worthwhile to use your application, like a user would, when you are stress testing it. Develop against it! You'll find what annoys you, what doesn't work, what works well, etc. Bullet proofing your app as best as possible against these scenarios is what is going to make a client love using your program.

# Improvements

I've found that there are a few quick places you can always look to find improvements

- **Data over the wire**. Always check data transport costs. Whether this is inter process communication, or client to server, it doesn't matter. Sending data isn't cheap and you should minimize what you send. Check the cost of serialization, remove extraneous data, sever object cycles, ideally map things to a DTO.
- **Add a facade**. Sometimes things just aren't designed to handle the data from the get go. An easy way to separate logical concerns is to put a facade in front of problem areas. This is mostly useful for storage classes and service calls. Having a facade that can translate your storage calls into view specific DTOs keeps your storage logic separate from your view logic. It also acts as a programmatic firewall. You can work behind the facade to fix underlying issues. As long as the front of the facade still works, then you've segmented the areas.
  - It should be noted that there is a limit to things you want to put a facade over. Don't use a facade to sweep problems under the rug. It should be used to give you better decoupling and breathing room to work.
- **Side effects**. Is there something that is hanging around after some action? Are file's not cleaned up? Is the disk fragmented? Are sockets sitting in CLOSED\_WAIT and not properly getting [disposed](http://stackoverflow.com/questions/898828/c-sharp-finalize-dispose-pattern) of? Sometimes you won't notice these things on the small scale, but in a larger scale they can become major issues.
- **Run a profiler**. For managed (C#) applications, a profiler is a no-brainer. Find that random linq statement that is getting re-evaluated in a loop. You'd be surprised at how many easy wins you can find when you profile your code.
- **Think about caching**. I'm not suggesting everything should be cached, or that it's always an appropriate solution. Frequently, though, slow-downs can be related to pulling more data than you need, or more often than is necessary. I once found a bug where we were marshaling 4MB of data from unmanaged to managed code every 50ms. This was a huge bottleneck, but we only found it when we simulated heavy user load. A simple cache on the data, that was invalidated when the source data was changed, let us scale to 10 times the load with minimal effort and no major code changes.
- **Think about bottlenecks**. Is something CPU bound or IO bound? If it's IO bound, does it have to be? Can you do it in chunks? Can it be asynchronous? If it's CPU bound, can it be batched? Does it have to happen now? Can it be distributed? Run tools like perfmon and the sysinternals suite to see what things are actually doing. Maybe you don't realize how often you are hitting the disk. If you avoided opening the file 50 times, and just read it all at once, things would go faster. Maybe use a memory map and dump it to disk at intervals. Is there a database call in a loop? Change the SQL to pull back more at once, instead of little by little. Do you have a for loop that you're hitting frequently? Maybe translate it to a dictionary, and use it as a lookup instead. Small changes that are run frequently can add up.
- **Ordering**. Sometimes, all you need to do is change the order in which something happens. Is something that is taking a long time, blocking something that takes a short time? Invert the sequence. Have the fast thing happen first, then the later ones. This gives the user an impression that something is happening, and not just broken.
- **Optimize the 90% case**. If something happens a lot, try and optimize it. This can cut down logarithmically on perceived performance.
- **Progress**. If you can't speed something up, give an indication of progress. People are less likely to get frustrated if they know things are actually happening, and not just sitting there
- **Cancellation**. If something takes a long time, give the user the option to cancel it. Maybe they didn't mean to hit that button that kicks off 20 minutes of work. They should be able to end the sequence, if they want.

There are obviously lots of other things you can do. In the end, you should remember the following: when you get pissed off at your tools for being slow, non-responsive, or difficult to use, think about how your application is the client's tool. Save them the grief.

