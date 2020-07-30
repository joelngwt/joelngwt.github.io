---
layout: post
title: The Tale of The Mysterious 100% CPU Utilization & The Ruby on Rails Database Reaper
tags:
  - rails
---

## The Story
Two days ago I was looking at the graphs of my EC2 instances on AWS and I noticed that all of the instances in my development environment were stuck at 100% CPU utilization.

![Parameter Groups](/images/database-reaping-frequency/1.png "CPU Utilization Graph")

This was the very first time ever seeing such a thing occur, so I was at a loss as to what was going on.

I checked the EC2s in the other environments. Production was totally fine. For staging, some were fine, and some were not.

Whatever could it be?

## The Hunt
I started to check the stats of the EC2 with the usual commands and tools:

```
passenger-memory-stats
free -m
top -c
ps aux
```

The memory was totally fine, with 50% of the memory not in use. But the results of `top -c` identified what processes were causing this.

![Parameter Groups](/images/database-reaping-frequency/2.png "Output of top -c")

Whatever could this be? This told me nothing.

Maybe there was something wrong with [Passenger's process spawning](https://www.phusionpassenger.com/library/config/apache/optimization/#minimizing-process-spawning)?

Nope, my settings prevent continuous spawning and shutting down.

Maybe the size of the code base is finally becoming too big, and it takes too long to start up?

```
time bundle exec rake environment

real    0m26.020s
user    0m19.351s
sys     0m5.905s
```

Slow, but not bad enough to cause the CPU to be stuck at 100% for hours.

Maybe it was the new Apache settings I set up? I disabled them, but nothing changed.

```
PassengerMaxPoolSize 24
PassengerMinInstances 24
PassengerPreStart https://website/
PassengerHighPerformance on
```

What in the world could it be?

So I created a pull request from the development branch to the staging branch to see what differences there were. Nothing! Nothing I could see that would cause such an issue.

## The Culprit Is Found

Then I created a pull request from the staging branch to the production branch. I came across this particular setting in my `database.yml` file:

```
reaping_frequency: 0
```

I removed it from the file. The CPU utilization went back to normal.

![Parameter Groups](/images/database-reaping-frequency/3.jpg "Rage")

## Why Did This Happen?

A few weeks ago I had to downgrade my application's version of Rails from 5.2 to 5.0 due to some issues with my database's performance. I added this setting as Rails 5.2 set this value to 60 by default, while it was disabled pre-5.2. Setting it to 0 disables it in Rails 5.2.

Let's take a look at the [documentation](https://api.rubyonrails.org/v5.1/classes/ActiveRecord/ConnectionAdapters/ConnectionPool.html) for the database reaper for Rails 5.1:

>`reaping_frequency`: frequency in seconds to periodically run the Reaper, which attempts to find and recover connections from dead threads, which can occur if a programmer forgets to close a connection at the end of a thread or a thread dies unexpectedly. Regardless of this setting, the Reaper will be invoked before every blocking wait. (Default `nil`, which means don't schedule the Reaper).

The fact that my application went crazy with the setting at 0 meant that it did not disable the database reaper when the value was 0, but instead, the reaper was running _permanently_.

Excuse me? Who in the world thought that was a good idea?

Let's take a look at the [documentation](https://github.com/rails/rails/blob/v5.2.4.3/activerecord/lib/active_record/connection_adapters/abstract/connection_pool.rb#L283) for the database reaper for Rails 5.2:

>Every `frequency` seconds, the reaper will call `reap` and `flush` on `pool`. A reaper instantiated with a zero frequency will never reap the connection pool.
Configure the frequency by setting `reaping_frequency` in your database yaml file (default 60 seconds).

A far better handling of the value `0`.

## Lesson
If you are ever downgrading Rails from 5.2+ to anything lower, and you have the value of `reaping_frequency` set to `0`, make sure you remove this setting entirely! This setting is _not_ backward compatible, and will cause your CPU to be stuck at 100% for days on end.

A Reaper indeed. What apt naming.
