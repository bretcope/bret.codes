---
layout: post
title: "Schyntax Part 2: The Task Runner (aka \"Schtick\")"
excerpt: "Scheduled task runner built on Schyntax."
tags: [schyntax, programming]
date: 2015-07-20
comments: true
image:
  feature: http://i.stack.imgur.com/STiIU.png
---

In [Part 1](/schyntax-part-1) we looked at the syntax of [Schyntax](https://github.com/schyntax/schyntax), a DSL for defining schedules, but defining schedules isn't all that useful if you don't have a task runner which knows how to execute on that schedule.

There's no reason you couldn't develop your own task runner... if you want to. That was part of the reason for deliberately separating the DSL from the task runner. However, if you feel like wasting less of your own time, there is pre-built task runner called "Schtick" (sch+tick... isn't that clever?) which is available in both JavaScript and C#, just like Schyntax.

## Setting Up a Task

### JavaScript

[![npm version](https://badge.fury.io/js/schtick.svg)](http://badge.fury.io/js/schtick)

{% highlight javascript %}
var schtick = require('schtick');

var s = schtick('minutes(*)', function (timeIntendedToRun) {
  // code to run every minute
});
{% endhighlight %}

That's it. Pretty simple, right?

You can stop the task using `s.stop()`, and resume it again using `s.stop()`. Complete documentation of the JavaScript Schtick can be found on [its GitHub page](https://github.com/schyntax/js-schtick).

### C\#

[![NuGet version](https://badge.fury.io/nu/Schtick.Redis.svg)](http://badge.fury.io/nu/Schtick.Redis)

The C# API is a little bit different.

{% highlight csharp %}
using Schyntax;

var schtick = new Schtick(); // best practice is to create a singleton of Schtick 

// setup an exception handler so we know when tasks blow up
schtick.OnTaskException += (task, exception) => LogException(ex);

// add a task which will call DoSomeTask every minute
schtick.AddTask("task-name", "min(*)", (task, timeIntendedToRun) => DoSomeTask());
{% endhighlight %}

`.AddTask()` returns a `ScheduledTask` object, with methods for `.StopSchedule()`, `.StartSchedule()`, and `.UpdateSchedule()`. This object is also the first argument to the task callback.

For asynchronous task callbacks, use `Schtick.AddAsyncTask()`:

{% highlight csharp %}
schtick.AddAsyncTask("name", "min(*)", async (task, time) => await DoSomethingAsync());
{% endhighlight %}

> The first argument to `.AddTask()` is the name of the task. This name must be unique among all tasks, and will help you identify them easier when debugging. If you're staunchly against human-readable names, you can pass in `null`, and a GUID will be assigned instead.

Complete documentation of the C# Schtick can be found on [its GitHub page](https://github.com/schyntax/cs-schtick).

## Distributed Locking via Redis

The above code is great for environments where you only have one server, or in cases where you want a task to run on _all_ servers in your environment. But what about when you want to run a task on only one server in a multi-server environment? In that case, you'll need some sort of locking mechanism, and [Redis](http://redis.io/) happens to be great at that.

If you're working in .NET (sorry, [no node.js implementation yet](https://github.com/schyntax/schyntax/issues/3)), and already have Redis in your infrastructure (which you probably should), then there is a pre-built solution for you called Schtick.Redis!

### Schtick.Redis

Available on nuget.org [![NuGet version](https://badge.fury.io/nu/Schtick.Redis.svg)](http://badge.fury.io/nu/Schtick.Redis)

> Schtick.Redis depends on [StackExchange.Redis](https://github.com/StackExchange/StackExchange.Redis) to communicate with Redis.

{% highlight csharp %}
using Schyntax;
using StackExchange.Redis;

// you should setup `schtick` and `wrapper` as singletons
var schtick = new Schtick();
var redis = ConnectionMultiplexer.Connect("localhost:6379");
var wrapper = new RedisSchtickWrapper(() => redis.GetDatabase());
{% endhighlight %}

Now, before you add a task to Schtick, you'll just wrap it:

{% highlight csharp %}
var callback = wrapper.Wrap((task, time) => DoSomething());
schtick.AddAsyncTask("task-name", "min(*)", callback);
{% endhighlight %}

That's all the code it takes to make sure the task only runs on one server per task per event time.

For wrapping asynchronous task callbacks, use `RedisSchtickWrapper.WrapAsync()`:

{% highlight csharp %}
var callback = wrapper.WrapAsync(async (task, time) => await DoSomethingAsync());
{% endhighlight %}

> Regardless of whether `.Wrap()` or `.WrapAsync()` is used, the wrapped callback is ___always___ added to Schtick via `.AddAsyncTask()`.

## Task Windows

There's one last real-world scenario to address. Imagine we have a task set to run every hour at the top of the hour, but the app re-deploys at 11:59:59 and is down for three seconds. In most cases, we'd like the task to run right away when the app comes back online, rather than waiting until the next hour.

To make this work, we need two pieces of information: 1. when did the task run last ("last run"), and 2. how long after it was _supposed_ to have run should we consider running it immediately vs waiting for the next scheduled time ("task window").

Luckily, if you're using Schtick.Redis, it already stores the last run information for you. You can retrieve it like this:

{% highlight csharp %}
var info = wrapper.GetLastRunInfo("task-name");
{% endhighlight %}

Then, pass `info.ScheduledTime` and your task window (let's use 30 minutes) to Schtick:

{% highlight csharp %}
schtick.AddAsyncTask("task-name", 
                    "hour(*)",
                    wrapper.Wrap((task, time) => DoSomething()),
                    window: TimeSpan.FromMinutes(30),
                    lastKnownRun: info.ScheduledTime);
{% endhighlight %}

> If you've never run the task before, `info.ScheduledTime` will be `default(DateTime)` which tells Schtick "I have no information to give you." Therefore, there's no need to special case the first run of a task.

The above code, in plain English, means: "run `DoSomething()` every hour, but if an event gets skipped because the app was down, then run it right away when the app finishes restarting as long as it finishes restarting within 30 minutes of the scheduled event time."

---

## Using Schyntax at Stack Overflow

We've had a scheduler (completely unrelated to Schyntax) for years. It runs as a service on a server in our primary data center and makes http requests to web servers in order to invoke tasks at defined intervals. It works great for most of our applications, including the [Q&A network](http://stackexchange.com/sites), and [Stack Overflow Careers](http://careers.stackoverflow.com/). However, for my team (Ad Server), we needed something with more precise timing, more flexible scheduling, and something which worked the same on local as it did in production without extra setup.

That's why I built Schyntax/Schtick. We use it for running data synchronizations, re-calculating multipliers for ad campaigns, taking snapshots of metrics, flushing analytics, posting scheduled messages into chat, and more. Some of our tasks run on all servers, and some only run on one at a time - all of which is easy to setup.

As an example of a scheduled task, we flush staged ad analytic data out of Redis and into SQL Server once per minute. The way our Redis memory usage smoothed out after we started using Schtick for that task really illustrates the difference in consistency between Schtick and our old scheduler. All of a sudden, the task which was supposed to flush data out of Redis once per minute was _actually_ running once per minute.

![Redis Memory Usage](http://i.stack.imgur.com/VQmd6.png)

To be clear, the memory spikes weren't actually a problem, they were simply a side-effect of the inconsistency of our old scheduler.

---

## Recap

A quick summary of all the libraries involved:

* "__Schyntax__" is the name of the DSL itself. Documentation for the language is in the root [Schyntax repo](https://github.com/schyntax/schyntax). There are fully-compatible implementations of Schyntax in [JavaScript](https://github.com/schyntax/js-schyntax) and [C#](https://github.com/schyntax/cs-schyntax).

* "__Schtick__" is the name of a task runner built on top of Schyntax. There are implementations in [JavaScript](https://github.com/schyntax/js-schtick) and [C#](https://github.com/schyntax/cs-schtick). Generally speaking, __Schtick is the library you should install__ in order to use Schyntax. However, there's no reason people couldn't build other scheduled task runners on top of Schyntax if the need arises.

* "__Schtick.Redis__" gives you distributed locking for tasks to prevent them from running on multiple servers. It's currently only available in [C#](https://github.com/schyntax/cs-schtick.redis), but there's no reason why a [JavaScript implementation](https://github.com/schyntax/schyntax/issues/3) couldn't be written.

I hope at least a few people find Schyntax useful. If you have any questions, comments, or are interested in contributing, contact me here, [on Twitter](https://twitter.com/bretcope), or [on GitHub](https://github.com/schyntax/schyntax). Thanks for reading.

← __goto__: [Part 1 of this blog post series is here](/schyntax-part-1).

---

> Schyntax was originally called "sch", because naming things is hard. You can thank [Matt Sherman](https://twitter.com/clipperhouse) for... well... naming things is hard.
>
> ![Why is my schedule not working? Looks like you're using the wrong schyntax.](http://i.stack.imgur.com/0dfWD.png)
