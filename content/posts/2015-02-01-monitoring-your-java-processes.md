+++
author = "Serban Balamaci"
date = 2015-02-01T22:16:18Z
description = "Monitoring your Java processes with monit"
draft = false
slug = "monitoring-your-java-processes"
title = "Monitoring your Java processes"

+++


I was looking for a complete and lightweight monitoring solution for my java processes when I found the perfect match **[Monit](http://mmonit.com/monit/documentation/monit.html)**.
I prefer many small 512MB memory Digital Ocean hosted instances to host my apps and therefore I was looking for a **lightweight free opensource** solution that is able to monitor a process, tries to **restart it in case it stops**, or **emails me in case it fails**. Digital ocean already provides graphs for cpu usage and I did not need fancy graphs therefore more complex solutions like **nagios** seemed like overkill.

I really like **monit** as I found it easy to setup and reading the config file for the service monitoring feels natural and self explanatory.

It's also surprisingly more powerful as it can monitor system resource usage or make network calls to check response on remote services and I think it can prove quite sufficient for most cases.

## Config
Install in Ubuntu is as simple as:
```
sudo apt-get install monit
```

and edit the config in **/etc/monit/monitrc** 
```
set daemon 120 #sets cycle=120secs

set logfile /var/log/monit/monit.log
set httpd port 2812 address localhost
    allow localhost
set mailserver smtp.gmail.com port 587
    using tlsv1 with timeout 30 seconds
    username "username" password "password"
set alert user@gmail.com with reminder on 15 cycles
```

Now Monit runs continuously as a daemon, but periodically(the period which we set to 120 is called a **cycle**) it wakes up and evaluates each service entry test defined in its configuration file. 

Lets see how a test looks like: 
```
check process buyit with pidfile /home/sbalamaci/projects/buyit/process.pid
  start program = "/home/sbalamaci/projects/buyit/start.sh" as uid="www"
  stop program = "/home/sbalamaci/projects/buyit/stop.sh"
if 3 restarts within 3 cycles then alert
```

I think the setup above is pretty self explanatory already, just to sum up this 
**Monit** is checking that the process called **buyit**(we just give it a name for **monit** to know how to refer to it) with id taken from the **pid** file is running. I always output and store the pid file for my java processes, if you do not have your own script, I suggest you take a look at mine <a href="https://gist.github.com/balamaci/42e649ac8e787f9b64dd">here</a>. 

If it's not running, it tries to start it under the user **www**. However the process might die again and may continue to do so every time monit tries to restart it.(Imagine a runtime exception shortly after startup, or a bad property, an OutOfMemory, etc that prevents the process from remaining up and running between the cycles). 
Rember that I said we can configure the **cycle** time, say we set it at 2 minutes. So **monit** is continually trying to start the failed process every cycle(2 min), but if it reaches our threshold it also notifies us(it will still continually try to restart it, see below other actions that prevent it).

Alerting is not the only action to take. We could also use: 

 - **exec** - executes a command and trigger the alert
 - **stop** - runs the service "stop command" and alerts but does not try anymore to restart it
 - **unmonitor** - alerts but no longer watches the service for state changes.
```
if 2 restarts within 3 cycles then exec "/my/cleanup/script"
```

Now it also means we also have another way of starting/stoping the services through monit like:
```
sudo monit stop buyit
sudo monit start buyit
```

and also a quick summary of the services status with
```
sudo monit status
sudo monit summary
```


### Checking a http response 
If you have a web app, while the proces might be active it does not mean it's serving http request. A more would be to also check the response to a http request, for ex in case of a REST service we can do something like:
```
check process buyit with pidfile /home/sbalamaci/projects/buyit/process.pid
  start program = "/home/sbalamaci/projects/buyit/start.sh" as uid="www"
  stop program = "/home/sbalamaci/projects/buyit/stop.sh"
  if failed port 8080
      with protocol http request "/some/path"
      with timeout 5 seconds
      3 times within 5 cycles
  then alert
```


#### Monitoring resource usage for whole system
But monit is even more powerfull and you can see already commented out example in the configuration file:
- Check general system resources such as load average, cpu and memory
```
  check system myserver
    if loadavg (1min) > 4 then alert
    if loadavg (5min) > 2 then alert
    if memory usage > 75% then alert
    if swap usage > 25% then alert
    if cpu usage (user) > 70% then alert
    if cpu usage (system) > 30% then alert
    if cpu usage (wait) > 20% then alert
```

#### Monitoring remote host accesibility
```
check host mmonit.com with address mmonit.com
    if failed
       port 80 protocol http
       with http headers [Host: mmonit.com, Cache-Control: no-cache,
         Cookie: csrftoken=nj1bI3CnMCaiNv4beqo8ZaCfAQQvpgLH]
       and request /monit/ with content = "Monit [0-9.]+"
    then alert
```

#### Monitoring disk space
```
check filesystem rootfs with path /dev/sda1
  if space usage > 85% for 3 cycles then alert
```

#### Other ex of capabilities
- Check contents of file for a pattern
- Run a script and check the return status:
```
 check program myscript with path /usr/local/bin/myscript.sh
       if status != 0 then alert
```

Many of this scenarios can be found <a href="http://mmonit.com/wiki/Monit/ConfigurationExamples" rel="nofollow">here</a> and I'll not reiterate them.

There are multiple times of **alerts** emited and you can set filters for which one you want to be notified by email.



### Monit as root while our applications deploy user is not root.
This was actualy my problem with using this setup fr deploying my java apps.**During deploys I don't want to be notified that the process is down**, and I don't want the monitoring to try to start it, since it might be in an inconsistent state. Basically **I needed a way to tell the monitoring process to unmonitor while the new version for the service is deploying**, but after I finish the deploy, I want to start monitoring the new instance. 

Since I installed and ran monit out of the box, it runs under root priviledges and we cannot issue commands like:
```
sudo monit stop buyit
sudo monit start buyit
```

unless we'd give the deploy user sudo rights, which was not something I wanted to do. 
There is also an option to build and run monit under a separate user than root, but I rather want the simpler "out of the box" setup and did not investigate this further.
 
Initially I found others with the same problem <a href="https://coderwall.com/p/ev8zzg/restarting-a-service-through-monit-with-a-restart-txt-file">here</a>. Bassicaly the proposed solution was to have a set of files named 'stop-proces1', 'start-process1' which would be created by a normal user and **we'd use monit itself to watch for these files** and invoke the 'monit start proces1; rm start_proces1'. It was an ok solution, but I was not 100% fond of this because:

1. I had to 'polute' my monitoring with the 2 extra service config for file watches for each service which was pretty tedious and prone to errors from copy&paste. 
2. Monitoring was also showing in it's /var/log/monit file each time it ran and did not find any of the above files which was 99% of the time. 
3. I needed to wait for monit to run it's cycle of 2 min before it figured out I created the 'star_proces1' file and start the service. This was making the total time of deploy quite high, and obtaining feedback that there was a problem with the build. 
4. I suspect it was a bug(since the 'apt-get' version is not one of the latest), but it sometimes did not seem to react on these file creations. 

#### My Solution - use curl with the embedded web server
- Use the local provided web server and **curl** to start/stop the services.

In the **Monit** config file you can uncomment and start a small web server which by default is available only for local connections, through which you see the status and can stop/start monitored services. 
```
 set httpd port 2812 and
    use address localhost  # only accept connection from localhost
    allow localhost        # allow localhost to connect to the server and
```

The url for the http commands for starting and stoping are pretty straight forward, and we can easily create a curl command that mimics them like this:
```
curl http://localhost:2812/buyit -d "action=start"
curl http://localhost:2812/buyit -d "action=stop"
```

Since the webserver accepts only local connections and I already connect through ssh to the machine there is no security problem and we don't have any issues with this approach. 

Hope I gave you an ideea to use in your projects and make your life easier.

### References
[Monit documentation](http://mmonit.com/monit/documentation/monit.html)
[Monit examples](http://mmonit.com/wiki/Monit/ConfigurationExamples)
[Monit primer](http://omgitsmgp.com/2013/09/07/a-monit-primer/)
