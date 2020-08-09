+++
author = "Serban Balamaci"
categories = ["docker"]
date = 2014-02-11T21:32:27Z
description = ""
draft = false
slug = "docker-part-ii"
tags = ["docker"]
title = "Docker Series - Docker part II"

+++


I'll be more explicit about some commands that you may want to use in your Dockerfile and explain how Docker works through them.
For introduction to Docker see <a href="http://balamaci.ro/docker-series-intro-to-docker/" title="Docker introduction">Part I</a>

**A docker container will keep running in the background as long as the initial command executed within the container is still running.**

#### CMD and ENTRYPOINT
You can define the command that will be run inside the container.
<pre>
ENTRYPOINT ["/usr/java/latest/bin/java"]
CMD ["-version"]
</pre>

So this setup defines that when we invoke 
<pre>
$ docker run above/testcontainer
</pre>

The **ENTRYPOINT** part **cannot be overriden at runtime**, while the **CMD** part supplies a **default value that can be overriden**. 

So to be usefull, this container needs some extra parameters to output something else than the java version inside it.
<pre>
$ docker run above/testcontainer -jar test.jar
</pre>
<small>
ofcourse the _test.jar_ in this case needs to be inside the container for the java process to see it.
</small>
So the container will be running for as long as the java process is running.
We can terminate it with:
<pre>
docker stop $container_id
</pre>
which will issue a SIGTERM and then a SIGKILL for the docker running process.


When building the container from the Docker file, the commands inside the Dockerfile are not run in a single(the same) container. After each command docker stops the container, writes the delta from the previous one. It then runs this newly saved container in order to execute the next command in the file. 
<small>
This explains also why you'll see no **cd** command by itself in the Dockerfile since it's efect is not saved. The **cd** commands are inlined with the actual commands.
</small>
So this raises the issue how can you run multiple processes inside a container(lets say  Tomcat and SSH). You'd want Tomcat to serve your app and ssh for you to be able to login and check say the tomcat logs from inside the container. 

One might think can just start SSH as a background service in the container and then start Tomcat something like:
<pre>
RUN /usr/bin/sshd -D
RUN service tomcat7 start
</pre>
but this wouldn’t work since when docker will be processing the _service tomcat7 start_ command, it will have shutdown the previous container in which _sshd_ was running.

So it seems there is a big problem that actually docker can only be running a single process . 
The answer is a tool like [Supervisor](http://supervisord.org/) which itself **takes care of starting other processes**, redirecting their output to log files and restarts them when they die.                     
#### Example Dockerfile
<pre>
FROM java/mvn

# We want our SSH key to be added
RUN mkdir /root/.ssh && chmod 700 /root/.ssh
ADD id_dsa.pub /root/.ssh/authorized_keys

## sshd requires this directory
RUN mkdir /var/run/sshd
RUN chmod 400 /root/.ssh/authorized_keys && chown root. /root/.ssh/authorized_keys

## we copy a config file from the host to the container
ADD supervisord.conf /etc/supervisor/conf.d/supervisord.conf

EXPOSE 22 5701

## we declare supervisord as the default command for the container
CMD ["/usr/bin/supervisord"]
</pre>
here is a simple _supervisord.conf_ example:
<pre>
[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D
stdout_logfile=/var/log/supervisor/%(program_name)s.log
redirect_stderr=true
autorestart=true

[program:hztest]
command=java -jar /root/hazelcast/java/hztest.jar
stdout_logfile=/var/log/supervisor/%(program_name)s.log
redirect_stderr=true
</pre>


#### EXPOSE
<pre>
# We declare two container internal ports as mappable from the outside
EXPOSE 8080 22
</pre>

So far as the applications running inside the container are concerned, they will be running on the container's private ports. 
You exposes a port from the container to be able to receive connections from other containers or outside. When started with the CLI run option **-p interface:host\_port:container\_exposed\_port** option we can bound that port to an interface of the host.
For example we can expose Tomcat inside the container to receive connections directly from the internet,but SSH only for internal connections
<pre>
docker run -p 0.0.0.0:9000:8080 -p 127.0.0.1:2022:22 &lt;image&gt; &lt;cmd&gt;
</pre>
and you can access **http://host_ip:9000** and the Tomcat inside the container will respond.

If you don't explicitly map the private port, a random port on the default docker interface will be mapped.

**Sidenote**
Containers when running are assigned an internal ip address something like, and you can find their ip(and a **lot other info**) by running:
<pre>
sudo docker inspect CONTAINER_ID
</pre>
to check all the running containers IPs:
<pre>
for i in $(sudo docker ps -q);
   do sudo docker inspect -format='{{.NetworkSettings.IPAddress}}' $i
done
</pre>




#### ADD
Copy some files from the HOST FS to the temporary CONTAINER and therefore be saved inside of the container image. 

**Example**: copy the user's public key inside the container authorized_keys for easier ssh login through key authentication.
<pre>
# And we want our SSH key to be added
RUN mkdir /root/.ssh && chmod 700 /root/.ssh
ADD id_dsa.pub /root/.ssh/authorized_keys
</pre>

Doesn't matter if you change the files you **ADD**ed after building the image, they are frozen inside the container image as they were at the time of the creation. For the container to see the new file version you need to rebuild the image.(For handling dynamic changing files see **VOLUME** bellow).

Remember that when rebuilding an image, Docker did **cache the commands in the Dockerfile**. Since you want the image to contain the new version, it means that the ADD command resets the cache for itself and following commands, therefore it's good practice to place the ADD clause at the foot of the Dockerfile. 
<small>
As of version 0.8 Docker has improved the caching mechanism to not reset the cache for ADD if the added files did not change(cache key is timestamp related).
</small>

#### VOLUME
A volume can be thought like a directory external to the container that can be mounted at runtime
<pre>
VOLUME ["app/conf", "/var/app/data", "/var/logs"]
</pre>

Volumes are immensely useful because:

-	They can be a good way to **"configure" at runtime** a container. Imagine for example passing different configuration directories for each container
<pre>
docker run -v /home/balamaci/project/app/conf_1:app/conf -v /home/balamaci/project/logs:/var/logs java/test_app
</pre>

through the **docker run -v** command **you can actually bind any folder inside the container, the folder doesn't necessarily have to have been defined in the Dockerfile through the VOLUME directive**.
So does the VOLUME directive has any "real" use? Yes, it does as it marks that directory with the content inside it as not going inside the image. 


-	They can be a way to share data / deliverables between containers and also the host. 
-	More convenient backups of data.

**Mounted volumes are not part of the container image**.
Changes to a data volume are made directly, without the overhead of a copy-on-write mechanism. This is good for very large files.

As a realcase **example** I also mapped my project jar / war files from the host as a volume inside the container.
That way, when I need to change something in the sources and rebuild I only need to do it once on the host and then the containers will have the new package version **without having to rebuild**  them.
The alternative of writing them(though either git pull or ADD) into the image would mean very tedious process of rebuilding and leftover images for each rebuild. Ofcourse once your files are in a stable releasable state writing them inside the image makes more sense.

Mapping a **/logs** volume for each container on the host I think it's a good ideea since it makes checking the log files and backup more convinient.

#### ENV
<pre>
#sets an environment property
ENV JAVA_HOME=/usr/java/latest
</pre>
Subsequent Dockerfile commands will see this property.

With docker you can pass different environment properties at runtime and which means added  flexibility because you can read this directly in your program or through bash scripts and for example  generate configuration files for processes.

Something like:
<pre>
#!/bin/bash

if [ "$RUN_MODE" = "production" ] ; then
sed -e "s#\${user}#$DB_USER#" -e "s#\${password}#$DB_PASSWORD#" env.template.properties > env.properties
fi
</pre>

This script uses **sed** to replace in the env.properties file constructs like bellow with the ENV value
<pre>
mysql_user=${user}
mysql_password=${password}
</pre>

and can run with
<pre>
docker run -e RUN_MODE=production -e DB_USER=john -e DB_PASSWORD=pass balamaci/test-app
</pre>


#### Storing the Docker images in a custom location.
I'm running a 64GB SSD on my laptop and I'd rather keep the images and containers on a 16GB stick.
By default docker keeps it's data in **/var/lib/docker**. 
We can change that by editing the docker config file **/etc/default/docker** to pass extra parameters to the docker daemon:
<pre>
DOCKER_OPTS="-g /mnt/my_custom_location"
</pre>
and bind mount the location:
<pre>
sudo mount --bind /mnt/my_custom_location /var/lib/docker
</pre>
Restart the docker daemon with 
<pre>
$ sudo service docker restart
</pre>

#### Docker containers IPs:
<pre>
$ sudo docker inspect -format='{{.NetworkSettings.IPAddress}}' $INSTANCE_ID
</pre>
or to show all running container's IPs:
<pre>
for i in $(sudo docker ps -q);
   do sudo docker inspect -format='{{.NetworkSettings.IPAddress}}' $i
done
</pre>

#### Run docker command without sudo 
The **docker client**(the **docker** command) communicates with the **docker daemon** through a . You can change the ownership of the . By default docker will bind the socket to the **docker** group if that group exists.


### Docker Gotchas
**Containers are NOT ephemeral**
After a container executes it's process or when you've stoped them explicitly, they still exist(that's why you cannot delete the image they're based on). The log files and state when stopped still exists inside the container, if you restart them, they will **NOT start fresh from the "clean state"** of the snapshot image. Some (including myself) thought that's the reason.

Still it can be helpful to keep separate volumes for data and logs dir so I can easily backup or check all of them.


**NOT easy to set a static IP for a container**
Docker creates a special Linux bridge network called **_docker0_** on startup. All containers are automatically connected to this bridge network and the IP subnet for all containers is randomly set by Docker. Currently, it is not possible to directly influence the particular IP address of a Docker container. Restarted containers might get a different IP(0.7 version). 

**Files like "/etc/hosts" and "/etc/resolv.conf"  are readonly inside container**
Combined with the issue above it's not very easy to reference other containers by name with an easy setup.

**UPDATE**: you can use **docker --link ** container_name, or **docker-compose** to link containers and then you can reference a host by name.
Or use [docker network](https://docs.docker.com/engine/userguide/networking/dockernetworks/) to create subnetworks for isolation.

**Layer limitations**

Dockerfile instructions are repeatable, but at present the AUFS limit of 42 layers means you’re encouraged to group similar commands where possible (i.e. combining separate apt-get install lines into one RUN command). The impact of this is that a single change to a long list of required apt-get packages means invalidating Docker’s build cache for that command and all those which follow it.


#### Docker and Ansible
Sometimes you may require extensive configuration for each container. I've seen a lot of people which recomend as a provisioning tool <a href="http://docs.ansible.com/" target="_blank" rel="nofollow">Ansible</a>. It's written in Python and seems easier to grasp than other provisioning tools like Chef or Puppet. Some suggested that you start with a basic container, have it configured through Ansible and in the end commit the result and just forget about the Dockerfile. 
<a href="http://lextoumbourou.com/blog/posts/getting-started-with-ansible/" target="_blank" rel="nofollow">Here</a>'s as a very good starting point for it.


#### References
<a href="http://docs.docker.io/en/latest/use/builder/" target="_blank" rel="nofollow">Docker guide</a>
<a href="http://www.hiddentao.com/archives/2013/12/26/automated-deployment-with-docker-lessons-learnt/" target="_blank" rel="nofollow">Automated deployment with Docker – lessons learnt</a>

