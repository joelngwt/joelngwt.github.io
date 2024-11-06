---
layout: post
title: Understanding Sidekiq's systemd Service Unit File
tags:
  - systemd
  - linux
  - ubuntu
  - daemon
  - service
  - sidekiq
---

## The Story
A few months ago, I updated a popular Ruby on Rails gem, Sidekiq, from version 5 to 6. It came with a [major breaking change](https://github.com/sidekiq/sidekiq/blob/v6.5.12/6.0-Upgrade.md) which removed the built-in daemonization that it came with, and users were asked to let the operating system handle it instead. The documentation suggested a few tools, among which were systemd, upstart, and foreman.

The problem? I had zero experience with any of these, and I didn't understand how any of these worked.

Fortunately, Sidekiq provided a [systemd service unit file](https://github.com/sidekiq/sidekiq/blob/main/examples/systemd/sidekiq.service), and it came with enough instructions in the file for me to figure out how to use it. So if you're here looking for a guide on how to set it up, the file contains enough instructions and information for you to set it up.

What I really want to explore in this post is: what does everything in the file mean?

## The Simplest systemd Service Unit File
Before I go into explaining Sidekiq's service unit file, let's learn the basics first.

At its core, the service file is a definition of three things:
- Name
- The thing to run
- When to run it

This translates to the following file:
```ini
[Unit]
Description=your-service-name

[Service]
ExecStart=/path/to/your/executable

[Install]
WantedBy=multi-user.target
```

It's really that simple! Next:
1. Create a file with the above contents and name the file `your-service-name.service`.
2. Place it in `/lib/systemd/system` (Ubuntu). Other Linux distributions might use different file paths, so just do a quick search if you're on something else.
3. Run `sudo systemctl daemon-reload` to let the system discover the newly added file.
4. Finally, run `systemctl enable your-service-name`.

That's it! Now you have a daemon that will automatically execute the program defined in `ExecStart` when your system has started.

I believe the `Description` and `ExecStart` should be pretty self-explanatory. But what does `[Unit]`, `[Service]`, `[Install]`, `WantedBy`, and `multi-user.target` mean?

## What Everything Means
### The Stuff In The Square Brackets
Anything within square brackets `[]` are the section headers of the file. Each section has their own set of keys that they accept. In both the above file and Sidekiq's file, only three headers are used. However, there are many other types of headers, such as `[Socket]`, `[Mount]`, `[Timer]`, among many others. This is a hint that systemd can be used for more than just applications - it can be used for networking, crons, managing mount points, etc.

### WantedBy
`WantedBy` is also rather self-explanatory. It means that this service is wanted by something else. So when that something else has started, this service would start as well.

It's one of the few ways to make services be a dependency of another service. The other ways would be using `Wants`, `Requires`, or `RequiredBy`. The difference between a "want" and a "require" is a soft dependency versus and a hard dependency. A soft dependency means that the dependency would try to be fulfilled, but if it fails, then it fails without affecting anything else. On the other hand, a hard dependency means that if the dependency fails, then the service that requires it fails as well.

[Documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html#WantedBy=)

### multi-user.target
This one is quite complex, and is answered very well by this [post on Unix StackExchange](https://unix.stackexchange.com/a/506374).

To summarize the vital bits:

> `multi-user.target` normally defines a system state where all network services are started up and the system will accept logins, but a local GUI is not started. This is the typical default system state for server systems, which might be rack-mounted headless systems in a remote server room.

To put it more simply, it's the state where the system is ready for use. So when you say that your service is `WantedBy` this, it means that it will spin up your service once the system has reached the `multi-user.target` state.

Without this dependency, your service will not start automatically, even if it's enabled.

### Sidekiq's Service Unit File
Sidekiq provides a service unit file [here](https://github.com/sidekiq/sidekiq/blob/main/examples/systemd/sidekiq.service), but I will copy it here for reference (with some comments removed):

```ini
[Unit]
Description=sidekiq
# start us only once the network and logging subsystems are available,
# consider adding redis-server.service if Redis is local and systemd-managed.
After=syslog.target network.target

[Service]
# As of v6.0.6, Sidekiq automatically supports systemd's `Type=notify` and watchdog service
# monitoring. If you are using an earlier version of Sidekiq, change this to `Type=simple`
# and remove the `WatchdogSec` line.
Type=notify
# If your Sidekiq process locks up, systemd's watchdog will restart it within seconds.
WatchdogSec=10

WorkingDirectory=/opt/myapp/current
ExecStart=/home/deploy/.rvm/bin/rvm in /opt/myapp/current do bundle exec sidekiq -e production

# Use `systemctl kill -s TSTP sidekiq` to quiet the Sidekiq process

# Uncomment this if you are going to use this as a system service
# if using as a user service then leave commented out, or you will get an error trying to start the service
# !!! Change this to your deploy user account if you are using this as a system service !!!
# User=deploy
# Group=deploy
# UMask=0002

# Greatly reduce Ruby memory fragmentation and heap usage
# https://www.mikeperham.com/2018/04/25/taming-rails-memory-bloat/
Environment=MALLOC_ARENA_MAX=2

# if we crash, restart
RestartSec=1
Restart=always

# output goes to /var/log/syslog (Ubuntu) or /var/log/messages (CentOS)
StandardOutput=syslog
StandardError=syslog

# This will default to "bundler" if we don't specify it
SyslogIdentifier=sidekiq

[Install]
WantedBy=multi-user.target
```

Now, let's continue with the explanations.

### After
`After=syslog.target network.target`

While "want" and "require" are service dependencies, `After` and `Before` are order dependencies. Helpfully explained in the comment in the file, Sidekiq will only start after `syslog.target` and `network.target` are ready.

The opposite direction would be `Before`. So if you could edit the syslog and network services, you could technically put `Before=sidekiq.service` there, and it would work the same.

[Documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html#Before=)

### Type
`Type=notify`

Every service has a type, and if it's not declared in the file, then it's `simple` by default. This is generally the choice, as it implies a non-forking process that runs only in the original process that systemd started.

The `notify` type that Sidekiq uses implies that Sidekiq has been coded to send notifications to systemd to let it know that it is ready. You could think of it as the readiness probe that Kubernetes has, but in the other direction, and only once. If you define your service as the `notify` type without actually sending any notifications, then your service will never transition to the "active" state.

There are other types, which can be found in the [documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html#Type=).

### WatchdogSec
`WatchdogSec=10`

The builds upon the `notify` type, turning the readiness probe into something that is sent at constant intervals. Setting `WatchdogSec=10` means that if systemd does not receive a notification in the past 10 seconds, it considers the service unresponsive. The service is then terminated, and it will not be restarted unless the `Restart` value is set properly (see the Restart section below).

[Documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html#WatchdogSec=)

### WorkingDirectory
`WorkingDirectory=/opt/myapp/current`

This changes the working directory of the service. By default, it's the root. This means that your service will work from the directory that you set here.

[Documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html#WorkingDirectory=)

### ExecStart
`ExecStart=/home/deploy/.rvm/bin/rvm in /opt/myapp/current do bundle exec sidekiq -e production`

While this might be self-explanatory, the value given might be a little more confusing.

Here, the command is telling `rvm` to switch to the `/opt/myapp/current` directory, then execute `bundle exec sidekiq -e production`.

[Documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html#ExecStart=)

### User, Group, UMask
```
# User=deploy
# Group=deploy
# UMask=0002
```

Used only if the service will be running as a system service, not a user service.

It defines the user, group, and file mode creation mask that the service will use. It's basically the permissions that will be set on the files that the service creates. The value of 0002 means that only the service will be allowed to write to it. Others will not be able to do so.

How `UMask` works can be a little complex. By default, files and directories have default permissions of 666 and 777 respectively. The mask is applied to this, resulting in the final permission value that is given to the file.

How the mask works is just a simple subtraction for each digit (so subtract the 1st digit with the 1st digit, 2nd with the 2nd, and so on. Do not just do 666-2). In this case, applying the 0002 mask to a newly created file with 0666 means that the final result would be 0664. Another example - applying a 0123 mask to 0777 would result in 0543.

[User & Group Documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html#User=)

[UMask Documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html#UMask=)

### Environment
`Environment=MALLOC_ARENA_MAX=2`

Well, this is just an environment variable for the service to use.

[Documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html#Environment=)

### RestartSec, Restart
```
RestartSec=1
Restart=always
```

`RestartSec` is the amount of time that systemd will wait before restarting a service.

[RestartSec Documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html#RestartSec=)

`Restart` defines when the service should be restarted. By default, a service will never restart on its own. Since Sidekiq sets it to `always`, it means that it will always be restarted (after waiting for 1 second) whenever it's not running for whatever reason (such as the service being terminated due to the `WatchdogSec` setting above).

Restart will also affect services that exit normally, so you might want to set it to `on-failure` if your service is expected to exit cleanly on normal usage.

[Restart Documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html#Restart=)

### StandardOutput, StandardError
```
StandardOutput=syslog
StandardError=syslog
```

This sets where the `stdout` and `stderr` is output to. In Sidekiq's case, it's output to the syslog.

[Documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html#StandardOutput=)

### SyslogIdentifier
`SyslogIdentifier=sidekiq`

Sets the prefix of the log lines, which indicates the process that sent the log line.

[Documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html#SyslogIdentifier=)

## Further Reading
I used the following articles to learn about systemd. The first one is a really good overall guide, and the second one is a summarized list of everything that can possibly be done with service unit files.
- [https://carpenoctem.dev/blog/just-enough-systemd-for-developers/](https://carpenoctem.dev/blog/just-enough-systemd-for-developers/)
- [https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files](https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files)
