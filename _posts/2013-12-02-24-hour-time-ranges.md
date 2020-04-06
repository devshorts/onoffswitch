---
layout: post
title: 24 hour time ranges
date: 2013-12-02 08:00:23.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- c#
- date
- time
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  _su_rich_snippet_type: none
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1560668063;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:3128;}i:1;a:1:{s:2:"id";i:3803;}i:2;a:1:{s:2:"id";i:2274;}}}}
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2013/12/02/24-hour-time-ranges/"
---
Dealing with time is hard, it's really easy to make a mistake. Whenever I'm faced with a problem that deals with time I tend to spend an inordinate amount of time making sure I'm doing things right.

Today I ran into a situation where I needed to be able to calculate durations and ranges from the current time compared to 24 hour block time. The current time however has the full date, but the 24 hour times are just relative. For example, if the current time is 17:00, and the range is 15:00 to 1:00, then I want to say that the current time is within the range. Also, lets say I have the current time is 17:00 but my range is 1:00 to 5:00. I want to know how far it is from now to the start of the 24 hour range. The ranges though, don't have date information, it's just generic time.

It took a bit of thinking but here is what I got. First, checking if a time is in the 24 hour range. Here we need to know what kind of range boundaries we have. The first check checks a normal boundary, where the start time is less than the end time. If that's the case then we can do a pretty easy range check. The second case checks if the range is an overnight boundary condition. In that case it needs to know if the current time is greater than the start OR if the current time is less than the end. But that OR can only work if the range is in overnight mode.

[csharp]  
/// \<summary\>  
/// Checks if the current time falls within a 24 hour range  
/// the date/year/month etc of the comparison dates WILL not be checked  
/// only the TimeOfDay is checked.  
///  
/// For example if the time is 17:00, and we check if we are in the range of  
/// 15:00 and 1:00 then the return will be true.  
/// \</summary\>  
/// \<param name="time"\>\</param\>  
/// \<param name="dtStart"\>\</param\>  
/// \<param name="dtEnd"\>\</param\>  
/// \<returns\>\</returns\>  
public static bool IsIn24HourRange(this DateTime time, DateTime dtStart, DateTime dtEnd)  
{  
 if (dtStart.TimeOfDay \< dtEnd.TimeOfDay && time.TimeOfDay \< dtEnd.TimeOfDay && time.TimeOfDay \> dtStart.TimeOfDay)  
 {  
 return true;  
 }

if (dtStart.TimeOfDay \> dtEnd.TimeOfDay && (time.TimeOfDay \< dtEnd.TimeOfDay || time.TimeOfDay \> dtStart.TimeOfDay))  
 {  
 return true;  
 }

return false;  
}

[/csharp]

To be paranoid, here is a unit test for it

[csharp]  
[TestCase(15, 3, true)]  
[TestCase(15, 16, false)]  
[TestCase(2, 1, true)]  
[TestCase(1, 2, false)]  
public void TestIsIn24HourRange(int startHour, int endHour, bool valid)  
{  
 var dtStart = new DateTime(1, 1, 1, startHour, 0, 0);  
 var dtEnd = new DateTime(1, 1, 1, endHour, 0, 0);

var n = new DateTime(1999, 12, 9, 17, 0, 29);

Assert.True(n.IsIn24HourRange(dtStart, dtEnd) == valid);  
}  
[/csharp]

Next up is calculating the time offset from one of these generic times. Since the time that is passed in has no relevant date information, you can't just do a simple subtraction on the times. You first have to normalize the time to be relative to the date.

[csharp]  
/// \<summary\>  
/// Determines the time range from the time to the the 24 hour time.  
///  
/// For example, if now is 17:00, and the end time is passed in (regardless of date)  
/// to be 2:00, then the duration will be 540 minutes. If the time is now 17:00 and the  
/// passed in time is 18:00, the duration will be 60 minutes.  
/// \</summary\>  
/// \<param name="time"\>\</param\>  
/// \<param name="end"\>\</param\>  
/// \<returns\>\</returns\>  
public static TimeSpan DurationFrom24HourRange(this DateTime time, DateTime end)  
{  
 var normalizedTime = new DateTime(time.Ticks).Trim(TimeSpan.TicksPerDay).Add(end.TimeOfDay);

if (time.TimeOfDay \> end.TimeOfDay)  
 {  
 var newTime = normalizedTime.AddDays(1);

return newTime - time;  
 }

return normalizedTime - time;  
}  
[/csharp]

The trim function can truncate a date to different granularities:

[csharp]  
/// \<summary\>  
/// Usage:  
/// DateTime.Now.Trim(TimeSpan.TicksPerDay));  
/// DateTime.Now.Trim(TimeSpan.TicksPerHour));  
/// DateTime.Now.Trim(TimeSpan.TicksPerMillisecond));  
/// DateTime.Now.Trim(TimeSpan.TicksPerMinute));  
/// DateTime.Now.Trim(TimeSpan.TicksPerSecond));  
/// \</summary\>  
/// \<param name="date"\>\</param\>  
/// \<param name="roundTicks"\>\</param\>  
/// \<returns\>\</returns\>  
public static DateTime Trim(this DateTime date, long roundTicks)  
{  
 return new DateTime(date.Ticks - date.Ticks % roundTicks);  
}  
[/csharp]

Again this involves an overnight boundary check. If the current time is greater than the passed in time, then it means the passed in time is in the next day. At that point we need to just add a day to the truncated (normalized) date and perform a timespan difference. Otherwise, it's all part of the current day and we can do a regular difference.

As usual, here's the unit test

[csharp]  
[TestCase(18, 60)]  
[TestCase(2, 540)]  
[TestCase(17, 0)]  
[TestCase(0, 420)]  
public void DurationFrom24Range(int startHour, int totalMinutes)  
{  
 var dtStart = new DateTime(1, 1, 1, startHour, 0, 0);

var time = TimeSpan.FromMinutes(totalMinutes);

var n = new DateTime(1999, 12, 9, 17, 0, 0);

Assert.True(n.DurationFrom24HourRange(dtStart) == time);  
}  
[/csharp]

