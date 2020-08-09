+++
author = "Serban Balamaci"
categories = ["docker"]
date = 2014-01-26T19:54:42Z
description = ""
draft = false
slug = "docker-series-intro-to-docker"
tags = ["docker"]
title = "Docker Series - Intro to Docker"

+++


### Foreword - How I stumbled upon Docker

So all this started when I was impressed after reading about <a title="Jepsen" href="http://aphyr.com/posts/281-call-me-maybe-carly-rae-jepsen-and-the-perils-of-network-partitions" rel="nofollow" target="_blank">Jepsen</a>. I like that that Kyle(the blogger) seems to have stumbled upon a generic framework for testing distributed systems behavior in cases of node failure and network partitioning etc, and how he was able to use this framework to do some nice analysis of **Cassandra**, **Ryak**, etc. So this got me thinking I wanted to do the same thing with <a title="Hazelcast" href="http://www.hazelcast.com/" rel="nofollow" target="_blank">Hazelcast</a> which I needed to see how it handles some network partition scenarios, slow network responses, etc and share my finding with the HZ dev team in hope of helping them build a perfect product.

My problem was that I'm running on an not very impressive personal laptop and I don't have a lot of resources to start up 4 individuals VMs and also do some coding and debugging on this machine. So I looked maybe at some alternatives(it's always great to be sidetracked by an interesting technology) and then I found out about Docker and got hooked on this thing which promises to hold great potential for automatization even on the development envs.


###Containers and how they compare to VMs
<p>I guess due to <strong>VMWare</strong> and <strong>VirtualBox</strong> most all of us know what a VM means and that it spins up from a disk image and that VMs will allow you to create isolated dev environments, however <strong>VMs are not small in terms of resources cost</strong>. This idea that you're running entirely isolated spaces as close as running on a different hardware means they cannot rely on reusing space and memory and thus waste much resources, along with the overhead of hardware virtualisation. Considering that an image stores OS files along with packages for each VM and they by their nature do not share this. I just need something easy to setup that will take advantage of the fact that I'm using <strong>the same damn thing</strong> 4 times, and I want it easy to set up.</p>

<p>So apart from VMs containers <strong>are extremely light weight</strong> since <strong>they all share the host's kernel</strong>:</div>
<div><img class="alignnone" alt="" src="https://www.rightscale.com/blog/sites/default/files/docker-containers-vms.png" width="400" height="300" /></p>
<small>
Sidenote: Some might be worried because containers sharing the same kernel between themselves and the host might invite the possibility of a <b>priviledge escalation</b> exploit from a "malware" process inside an untrusted container. I don't have the usecase of a multi tenancy system setup and the containers I run are the one I chose, so this is not a problem for me, however there are articles offering guidelines on mitigating this, if you interested.  
</small>
<p>"<strong>An application running inside a container will be executed directly on the operating system kernel of the host system, shielded from all other running processes in a sandbox-like environment</strong>"</p>
<ul>
<li>- "If multiple instances of a same process are running, the Linux kernel can share memory pages that are identical and unchanged across all application instances. This also applies to shared libraries that applications may use, they are generally held in memory once and mapped to multiple processes."</li>
<li>- Ideally there will be little overhead in the system if you are running a process in a container, comparable to as if you'd started multiple instances of an process.</li>
<li>- It also means the spinup time for the container should be very small.</li>
<li>- From a pragmatic point of view I guess this will translate to lower cost from resources and maybe for easier setup of multi-tenant architectures.
</ul>

<h3>LXC</h3>
<p>When talking about the Linux Containers concept we'll be talking about the implementation of this concept [LXC](http://linuxcontainers.org/") an open source project (which relies on [cgroups](https://access.redhat.com/site/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/ch01.html) - to limiting the amount of system resources allocated to users/tasks like max memory or <strong>cpu</strong>, <strong>disk</strong>, <strong>network</strong>) and <strong>namespacing</strong> for sandboxing (more explained <a title="dotclound" href="http://blog.dotcloud.com/under-the-hood-linux-kernels-on-dotcloud-part">here</a>) <small></em><strong>OpenVz</strong></em> would be another implementation of the container concept in Linux.</small></p>


<ul>
	<li>File system: a container can only access its own sandboxed filesystem (<a href="http://en.wikipedia.org/wiki/Chroot" target="_blank">chroot</a>-like), unless <a href="http://docs.docker.io/en/latest/commandline/command/run/" target="_blank">specifically mounted</a> into the container's filesystem.</li>
	<li>User namespace: a container has its own user database (i.e. the container's root does not equal the host's root account)</li>
	<li>Process namespace: within the container only the processes part of that container are visible (i.e. a very clean <tt>ps aux</tt> output).</li>
	<li>Network namespace: a container gets its own virtual network device and virtual IP (so it can bind to whatever port it likes without taking up its hosts ports).</li>
</ul>

So why you want to consider using <strong>containers</strong> more often is because they<strong> offer</strong>:

<ol>
	<li><strong>Isolation + Multitenancy:</strong>For Java consider a scenario of a SaaS application, you might think of deploying war files in the same Tomcat instance. This means that if you have a reason to restart the Tomcat all of the clients will suffer. To keep them more isolated and assure a fair share of resource usage, you can run a Tomcat and the application in a separate container for each client. That means you can take out separate instances for upgrades without impacting other running ones or run them in paralel untill you are ready to make the switch.</li>
	<li><strong>Security:</strong> If you're running a service like <em>Wordpress</em>, not the software with the best security track record. It would be nice to sandbox this application so that once hacked at least it doesn't see/impact the other running containers. On the other hand sharing the same kernel means that if something like a <em>"priviledge escalation"</em> exploit is run on a compromised container<a href="http://blog.docker.io/2013/08/containers-docker-how-secure-are-they/" rel="nofllow">might affect the host</a>. VMs are better at this isolation because of the separate guests OS from Host, but containers are still better than just running all in same host not jailed</li>
    <li><strong>Containers can run inside  VMs</strong>:</li> Docker is quite flexible and can be run inside a VM which is a great way to play with it, or just to set your mind at ease regarding the shared kernel possible security issue.
	<li><strong>Constrain resources:</strong> what happens when your app goes wild starts to take up all our CPU cycles, completely blocking other applications from doing any work or generates logs like crazy, clogging up the disk? It's easy to limit the resources through a container.</li>
	<li><strong>Elegant deployment</strong>: You could add the dependencies of the application in the container itself. This way you could ship directly a container with your application and save your customers or users the time and frustration involved in finding and installing dependencies of your application. It's neat that you can download already some projects to play around and make an idea. Also some cloud hosting providers allow you to create an instance based on your docker image.</li>
	<li><strong>Snapshoting, backing up:</strong> it would be nice, once everything is setup up successfully, to "snapshot" a system, so that the snapshot can be backed up, or even moved to a different server and started up again, or replicated to multiple servers for redundancy.</li>
	<li><strong>CI:</strong> It's good practice to automate deployment and to test a new version of a system on a test infrastructure before pushing it to production. So you can take a snapshot of a clean state of machine that's used in production with full stack of packages and configs, and have a build tool like <strong>Jenkins</strong> spin up a container and run the integration tests for your application.</li>
	<li><strong>Ease of removal:</strong> software should be easily and cleanly removable without leaving traces behind. However, as deploying an application typically requires tweaking of existing configuration files, and putting state (MySQL database data, logs) left and right, removing an application completely is not that easy.</li>
</ol>
</p>

When running containers, they can **bound their ports to ports on the host machine**(what is port 80 in the container can be exposed as port 2000 on the host), can have **host machine folders shared with them**, **can share folders with other running containers** – all in all very similar to what VMs can do.

<small>Sidenote: you don't really need **LXC** if you just want to limit a process to not hog all the cpu, memory, disk, etc and bring down a machine. You can do that with plain **cgroups**, but I'm a developer - not sysadmin- and like the high level(easier) not the "bare metal" one.

While we'll not work directly with LXC containers setting up a LXC container does not seem very hard, but as seen later, <strong>Docker</strong> will hide the "ugly" plumbing from us and let us work with a high level interface.
</small>


### Docker = LXC + CopyOnWrite FS(AuFS) + IPTABLES + Provisioning tools

Just like the VMs image, Docker is also based on a <strong>docker image</strong>, but Docker makes use of the <strong>AuFS</strong> which is a CoW(CopyOnWrite) file system which is very useful since a CoW FS means that when <strong>two(or more) processes use the same files there is no need to create copies of the files</strong>, but if one of the process needs to change the files, then it gets a separate private copy of the files it modifies and only a diff is stored. This allows <strong>a great reuse of disk space</strong> if you start more than one(image hundreds) of the same type container and work of the concept of <strong>layers</strong> which is even more interesting in the sense that you stack changes on top of one another(we'll see later).

<b>Sidenote</b>: at the time of the writing, Docker requires kernel 3.8+ and Ubuntu LTS version uses the 3.5 kernel, but you can upgrade the kernel very easy without going through a complicated process like recompiling yourself.

#### What Docker does for containers:

You start with a <strong>base image</strong>. These images live inside <a href="http://index.docker.io/" target="_blank" rel="nofollow">Docker Registry</a> (<a href="https://github.com/dotcloud/docker-registry" target="_blank" rel="nofollow">source</a>, <a href="https://index.docker.io/" target="_blank" rel="nofollow">search</a>). The registry is open source and you <a href="http://blog.docker.io/2013/07/how-to-use-your-own-registry" target="_blank" rel="nofollow">can host your own</a> inside the company. <strike>As you work with a container and continue to perform actions on it (e.g. download and install software, configure files etc.), to have it keep its state(create a savepoint), you need to “commit”, very similar to a version control system</strike>. The above was a missunderstanding from me and maybe others will think like this, but containers will keep their state, even when stopped and restarted. You can however **commit** the state of a container into a new image which you can reference and build upon.

#### Docker daemon and client
The **docker process runs as a daemon and awaits for commands** to manage all the containers and docker images. It needs to run as root so that it can have access to everything it needs.

We give commands to the **Docker daemon** through the **docker client** -command line process. 

![containers-linked](https://kjanshair.azurewebsites.net/images/blogs/docker-one/Docker-Components.png)

"By default the Docker daemon listens on _unix:///var/run/docker.sock_ and the client must have root access to interact with the daemon. If a group named "docker" exists on your system, docker applies ownership of the socket to the group."

You can change ownership of the linux socket so it's easier to invoke the **docker client** without sudo. See [here](https://balamaci.ro/docker-part-ii/) how.

There is also a REST like API through which you can use curl to give commands to the daemon - read [here](https://balamaci.ro/docker-part-ii/).

It can also be exposed and communicate using a [HTTP socket](https://docs.docker.com/engine/security/https/). 

We can fetch an image with the **docker pull** command from a remote repository. Let’s search for the <em>ubuntu</em> image on the [docker registry](https://registry.hub.docker.com/) and install it using docker
<pre>$ sudo docker search ubuntu
$ sudo docker pull ubuntu
</pre>

let's look at what is available localy now:
<pre>$ sudo docker images</pre>

<table>
<tr>
<td>REPOSITORY</td><td>TAG</td><td>IMAGE ID</td><td>CREATED</td><td>VIRTUAL SIZE</td>
</tr>
<tr>
<td>ubuntu</td><td>12.04</td><td>8dbd9e392a96</td><td>9 months ago</td><td>128 MB</td>
</tr>
<tr>
<td>ubuntu</td><td>latest</td><td>8dbd9e392a96</td><td>9 months ago</td><td>128 MB</td>
</tr>
<tr>
<td>ubuntu</td><td>12.10</td><td>b750fe79269d</td><td>10 months ago</td><td>175.3 MB</td>
</tr>
</table>
</p>
<small>Sidenote: Docker is requiring sudo for every command, and right now I'm tired of typing sudo for everything so a sidenote it's easier to enter an interactive sudo session with:
<pre>$ sudo -i</pre>
</small>
and now let's actually start the container and install Java</p>
<pre>$ docker run -t -i ubuntu /bin/bash</pre>

<small>
<strong>-t</strong> - a semiterminal, so you can see the output of commands

<strong>-i</strong> - interactive mode so that for commands needing user input, like writting the bash commands.

</small>

and you enter a bash prompt inside the container
```
# apt-get install software-properties-common
# add-apt-repository -y ppa:webupd8team/java
# apt-get update
# apt-get install -y oracle-java7-installer
```
so this container now has Java installed, but <strike>if we don't commit this new state, it will be lost</strike>(the container still keeps it's state, but we'd like to save it as an image so we can build other images base on it, or start containers from this image).

if we open up another terminal, since bash is running inside the container it also keeps the container in a running state. To list all <strong>running containers</strong>:
```
$ docker ps
```
<table>
<tr>
<td>CONTAINER ID</td><td>IMAGE</td><td>COMMAND</td><td>CREATED</td><td>STATUS</td><td>PORTS</td><td>NAMES</td>
</tr>
<tr>
<td>cae5ecb47e3a</td><td>ubuntu:12.10</td><td>/bin/bash</td><td>10 minutes ago</td><td>Up 19 minutes</td><td>focused_fermat</td>
</tr>
</table>
<small>
If we exit bash(press CTRL+D) the container will <strong>not be in a running state</strong> so <strong>$ docker ps</strong> will not show it up anymore.
To show also stopped containers type <strong>$ docker ps -l</strong>.
</small>
We could commit this container right now. 

<pre>
$ sudo docker commit $CONTAINER_ID myname/java7
</pre>

Congratulations on your first Docker container, running 
<table>
<tr>
<td>REPOSITORY</td><td>TAG</td><td>IMAGE ID</td><td>CREATED</td><td>VIRTUAL SIZE</td>
</tr>
<tr>
<td>balamaci/java7</td><td>latest</td><td>cae5ecb47e3a</td><td>1 min ago</td><td>128 MB</td>
</tr>
</table>

So we can use this container as a base for others building layer upon layer like **java7&maven3**
and on top of this have maybe two other images:

-	**java7&maven3** + **Tomcat7**
-	**java7&maven3** + **Jetty9**

Due to the usage of AuFS sharing of files, space is being reused for the shared layers.

Space is being reused because only the difference between a container instance and it's image needs to be kept. 


We can check the hierarchy, and dependencies with
<pre>
$ docker images -tree
</pre>
PS: this _tree_ command has been removed it seems from recent versions of docker in an effort to keep the core as small as possible).

<pre>
├─8dbd9e392a96 Virtual Size: 128 MB Tags: ubuntu:12.04, ubuntu:latest, ubuntu:precise
└─27cf78414709 Virtual Size: 175.3 MB
  └─b750fe79269d Virtual Size: 175.3 MB Tags: ubuntu:12.10, ubuntu:quantal
    ├─03be90cb581d Virtual Size: 745.9 MB Tags: balamaci/java7:latest
     └─7b58cf7be27d Virtual Size: 752.4 MB Tags: balamaci/mvn3:latest
       ├─96cc8ee708b6 Virtual Size: 762.4 MB Tags: balamaci/mvn+jetty9:latest
       └─3966ef6b2513 Virtual Size: 792.9 MB Tags: balamaci/mvn+tomcat7:latest
</pre>


### Dockerfile
Until now I've shown how to build the image by writting into the container _bash_ and **commit**ing, but this get tedious and not very clear, and it would be nice to just be able to write a "recipe" for how to build an image automatically, like a provisioning tool. It also makes sense to see what were the commands that created the image, because otherwise I might build upon a "java+tomcat7" public image that also contains some kind of hidden "malware" commands.  

And for that is exactly what the **Dockerfile** is, and I think it's very easy to understand while you'll have a look at one:
Write this in a text editor and save it with the name **Dockerfile** in a directory.
<pre>
FROM dockerfile/ubuntu

#Install Java7
RUN apt-get install -y software-properties-common
RUN add-apt-repository -y ppa:webupd8team/java
RUN apt-get update
RUN echo debconf shared/accepted-oracle-license-v1-1 select true | debconf-set-selections
RUN echo debconf shared/accepted-oracle-license-v1-1 seen true | debconf-set-selections
RUN apt-get install -y oracle-java7-installer

#install maven3
RUN wget -O /tmp/apache-maven-3.1.1-bin.tar.gz http://ftp.jaist.ac.jp/pub/apache/maven/maven-3/3.1.1/binaries/apache-maven-3.1.1-bin.tar.gz
RUN cd /usr/local && tar xzf /tmp/apache-maven-3.1.1-bin.tar.gz
RUN ln -s /usr/local/apache-maven-3.1.1 /usr/local/maven
RUN rm /tmp/apache-maven-3.1.1-bin.tar.gz
</pre>

pretty self explanatory right, only this is the basic form for one, but still very sufficient, we'll get into more details in part II.

To **build the image** based on the recipe file from the directory where you save the Dockerfile run and give whatever name you want throught the **-t** parameter: a pseudoconsole to see the output
**-i** parameter:
<pre>
$ docker build -t balamaci/mvn3 .
</pre>
<small>
1. the '-y' you see passed to most commands it means that it should assume 'yes' and not wait for user input when the scripts asks if you're sure you want to install.
2. 'software-properties-common' is installed so it provides the 'add-apt-repository' command which makes it easy to install a PPA(Personal Package Archive - a private package repository). We get from the PPA  'oracle-java7-installer' which grabs the JDK from the net(due to license, the PPA is not allowed to provide it in the installer. Previous commands are automated way to accept the user license.
</small>

and after it's finished it will output the new image id.
<pre>
...
Step 11 : RUN rm /tmp/apache-maven-3.1.1-bin.tar.gz
 ---> Running in 2ea26cca76f1
 ---> e85ee46afecc
Successfully built e85ee46afecc
</pre>

What docker is actually doing is **running each command in the Dockerfile** and creating a new container and **commits a new image** for each, so if we look at the new created image we'll see a chain of multiple commit that created the image. This is useful for command caching.

<pre>
  └─b750fe79269d Virtual Size: 175.3 MB Tags: ubuntu:12.10, ubuntu:quantal
    ├─c719d0cbc6dd Virtual Size: 209.7 MB
     └─537f749ee8d9 Virtual Size: 209.7 MB
       └─dbab722d3963 Virtual Size: 306.3 MB
         └─5cc0aaf9cb0e Virtual Size: 309 MB
           └─d474c9d5aad8 Virtual Size: 309.1 MB
             └─9ed6f0252815 Virtual Size: 783.9 MB
               └─eb8c6fe66b4c Virtual Size: 789.4 MB
                 └─23cfd257bc9a Virtual Size: 795.9 MB
                   └─7b8b82ad1b84 Virtual Size: 795.9 MB
                     └─e85ee46afecc Virtual Size: 795.9 MB
</pre>

#### Caching the commands

It's very nice that it only takes some time only on the first run of the build to pull the dependencies, then it **caches** the commands so that if you run them again(you rebuild using the same Dockerfile) without changing their order, it's not going to run the previous steps again, it will know to build upon to the previously committed layers. 
If, however, you do change an instruction in the Dockerfile or re-order some instructions around then Docker will only re-use the previously built containers up until the point where the Dockerfile has changed.


#### Docker Images vs Dockerfiles
To sum up again, **Dockerfiles** are **recipes** that when given to the **docker build** command result in **docker images** which are **immutable images** that docker is using to run your code.

It's also important to understand because it answers the question:
Q: "If two different people use **the same Dockerfile** to build an image, does it mean they have the same image to run on?"
A: Not necessarily, because say we both use the same "Java7" Dockerfile, but I built the image a month after you, while we both used the same Dockerfile, when my build process pulled java7 from the repository, it may be a newer minor version that what you have in your image when you built it a month ago. Consider also this for "apt packages" as well. 

So if we really need to make sure we run the same thing, rather you'd build your image and **publish** it somewhere(public or private repository). 
Then I'd pull it from there and then I can be sure we're running the exact same setup.

#### Publishing your docker image to a repository
Like **git** pushing to a remote repository is the done through **docker push**. 

**IMPORTANT** When you run <code>docker push</code> to publish an image, **it's not publishing your source code(Dockerfile)**, **it's publishing the docker image** that was built from your source code.

If you go browsing around on the docker <a href="http://index.docker.io/" target="_blank" rel="nofollow">public index</a>, you'll see lots of images listed there, but weirdly, you can't see(since "trusted builds" for some you can) the <b>Dockerfile</b> that built them. The image is an <em>opaque asset</em> that is compiled from the <em>Dockerfile</em>.

The lack of the quick info of what exactly went into a build and trust that it does not contain any other "hidden" functionality was solved by the implementation of a **"Trusted build"** functionality that you link your Github account and you publish and change your Dockerfile there and a post build commit hook triggers the building. That way the published image is guaranteed to be the exact one from the Dockerfile. 

#### Deleting the built images gotcha
I can't finish without telling you about this little gotcha that got me scratching my head.

There are two separate commands for deletion

A. remove a container - docker rm $CONTAINER\_ID
B. remove an image - docker rmi $IMAGE\_ID

And if you want to remove a certain image, you cannot remove the image unless you explicitly remove the container that was running on it. 
What is not obvious is that **even if you are not having any active containers running the image you cannot remove the image**. 

**A stopped container still exists even if stopped and it's running state preserved. By default Docker will not remove it, since it could be restarted from that state** if you wanted.

For every command in the Dockerfile when building, Docker will create an "intermediate"  container so there are a lot of "leftover" containers. 
<small>
To remove these intermediate containers when building from a Dockerfile.
<pre>
docker build -rm .
</pre>
</small>


To list all the containers.
<pre>
docker ps -a
</pre>
you can see what the container last ran command was and check it's status if it's 'Up for ...' or 'Exited'.

To save yourself the hassle, this command removes any non running containers(it will fail for running ones)
<pre>
docker rm $(docker ps -a -q)
</pre>
Then you can try again to remove the image you want. All this cleaning of stopped containers, unused images can be quite a chore. 
<pre>
# Delete all images
docker rmi $(docker images -q)
</pre>

#### Great References
<small><a href="http://kencochrane.net/blog/2013/08/the-docker-guidebook/" target="_blank" rel="nofollow">Docker guide</a>
<a href="http://www.infoq.com/articles/docker-containers" target="_blank" rel="nofollow">Docker Containers</a>
<a href="http://footyntech.wordpress.com/2013/08/23/what-docker-is/" target="_blank" rel="nofollow">Docker for Dummies</a>
<a href="https://www.digitalocean.com/community/articles/how-to-install-and-use-docker-getting-started/" target="_blank" rel="nofollow">How To Install and getting started</a>
<a href="https://blogs.atlassian.com/2013/06/deploy-java-apps-with-docker-awesome/" target="_blank" rel="nofollow">Atlasion blog - Docker</a>
</small>


Please ask any questions if you have. For now I'm preparing Part II that I'll detail some detailed explanations of Dockerfiles, best practices and gotchas of Docker.
