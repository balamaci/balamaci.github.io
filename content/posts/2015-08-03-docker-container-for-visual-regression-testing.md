+++
author = "Serban Balamaci"
categories = ["docker", "webdriver"]
date = 2015-08-03T19:47:49Z
description = ""
draft = false
image = "/images/2015/10/docker_ui_testing.svg"
slug = "docker-container-for-visual-regression-testing"
tags = ["docker", "webdriver"]
title = "Docker containers for Visual Regression Testing setup"

+++


### Packing it all inside docker containers 
In the previous [post](http://balamaci.ro/docker-container-for-visual-regression-testing/) we looked at a possible solution for doing **Visual Regression Testing**. But stepping back and looking at it, I think there was tedious for the reader to actually try it out, because it required him to install some extra dependencies like **ImageMagick**, **phantomjs**( which in turn needs **nodejs**) to be able to run the example himself(also not considering **docker** installation). This would have gotten even more complicated if we wanted to add more browser beside phantomjs, but Docker containers can also be used as "build containers" not just to run applications. 
Doing away with extra configuration of dependencies and just run your application or tests inside a container build from a **Dockerfile** **recipe**(or a prebuilt one available from a repository) is one of the "features" of working with containers. 

Quite a few cloud services have appeared recently that offer to run docker containers. Mix them up with some git hooks and this opens the way to **run unit tests and integration tests inside the containers on each commit** or pull request before merging into your codebase. Something like what [rultor](http://www.yegor256.com/2014/07/24/rultor-automated-merging.html) is doing. What a great time to be alive :).

### Docker
We're not going to talk about what Docker is, as we already did it [here](http://balamaci.ro/docker-series-intro-to-docker/) some time ago.

### The plan

Just a little reminder of what we planned on doing. We have our web application **pippo-demo.war** file which was already built by Jenkins in another step. Part of our build setup we also have separate integration tests and visual regression tests to run for the web application.


#### Running our integration tests inside a prebuilt docker container with all the dependencies already baked in. 


We want to deploy the **pippo-demo.war** inside an "actual" Tomcat instance and run integration tests using Selenium **Webdriver** against the deployed web app.

To keep things simple we just want to be able to run our integration tests through a single command.

We needed **ImageMagick** along with **phantomjs** and **nodejs** to do the visual regression tests.

  - So we'll create a new **ui-tests** docker image with these dependencies already installed. We base this **ui-tests** image upon a simple **maven3** docker image because it's inside this image we **checkout our integration tests** suite.  

Our **integration tests are organised as a Maven project** so we can run them with just 
```
mvn clean verify
```

We have prepared a generic reusable **Tomcat7 container** into which we deploy **pippo-demo.war**.

 - **pre-integration-test** phase we use the **cargo-maven2-plugin** to deploy the application inside the **Tomcat7 container**.
 - **maven-failsafe-plugin** runs our plain JUnit tests with  Webdriver against the endpoints which we already discussed in [article](http://balamaci.ro/visual-testing-automation/).

![Build container](/assets/2015/docker_ui_testing.svg)


All the dockerfiles necessary to create the images are under [docker](https://github.com/balamaci/blog-ui-testing-container/tree/master/docker) folder.

PS: They are there as examples and for should you want to create your own dockerfiles. Be aware that there is an official docker [repo location](https://registry.hub.docker.com/_/maven/) for maven containers just know it's built upon an **open-jdk** image(might be a legal issue with distributing the oracle jdk).

That is the reason why I've not pushed the images myself and will instead maybe change the example to openjdk images.

### Docker Compose - a way of "orchestrating" containers

Sometimes when you have a more complicated setup, for ex.  your web application for which you do integration testing also requires a database, we can also pack that database inside a container. 

It makes sense to have separate services like **databases**, **web containers** defined and kept in separate containers to keep things clear, configurable(for example to be able to test  with a different database) and play around with the "big building blocks" sort of like legos to swap one component for another.

Orchestrating those containers can get messy: specifying which volume directories are shared with the host or between containers themselves, what ports are to be exposed for each container and  how they should be linked.

This is where [Docker Compose](https://github.com/docker/compose)(previously named **fig**) can help us. 
Through a file called **docker-compose.yml** you can describe the whole setup. Let's look at the file we've used for the project:

```
tomcat:
  image: balamaci/tomcat7
  volumes:
    - /tmp/tomcatlog:/opt/tomcat/logs

uitests:
  image: balamaci/ui-tests
  volumes: 
    - .:/usr/src/app 
    - ~/.m2:/root/.m2 
  links:
    - tomcat
  ports:
    - 8080:8080
```

We defined two services **tomcat** and **uitests**.

Our **uitests** container is **linked** to the **tomcat** container. This means there is an entry added into **/etc/hosts** inside the **uitests** container and to access the Tomcat web app server we can just do _http://tomcat:8080_ (from **uitests** which is our actual container in which we run the tests).

With
```
volumes: 
    - .:/usr/src/app 
```
The whole current directory - if we checked out the [project](https://github.com/balamaci/blog-ui-testing-container/) - **source files for the integration tests are being shared with the _uitests_ container** through a **docker volume**, they are not "burned" inside the image(like with the ADD command for Dockerfile). This way we can always checkout new tests and just reuse the same **uitests** container.

while with
```
  volumes:
    - ~/.m2:/root/.m2 
```

We shared the **.m2** folder from the host with the container. That is because we don't want everytime we run our integration tests to pull all of maven dependencies from the internet again and again.

For more detailed help with every command, or what else you can configure be found [here](https://docs.docker.com/compose/yml/)

If we have added in the config file a **command** line to be executed implicitly then we just could have done
```
docker-compose up
```

but instead we'll run the mvn command explicit
```
docker-compose run uitests mvn clean verify
```
which will also start the linked **tomcat** container

Stopping all the containers is done with:
```
docker-compose stop 
```
to also remove the containers so we have a clean slate:
```
docker-compose rm 
```
PS: We could have done this setup I think with all the dependencies and Tomcat server baked into a single image and container, but we kept things more clear this way and we also get a chance to play with **docker-compose**.
PS 2: **docker-compose** it's just a tool, a facilitator over  docker. The created containers are in everyway "plain" docker containers, and all docker commands still apply to them.

### The coderun

Installing docker and docker-compose:
```
#docker
wget -qO- https://get.docker.com/ | sh

#docker-compose installing
curl -L https://github.com/docker/compose/releases/download/1.3.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

clone the repo:
```
git clone git@github.com:balamaci/blog-ui-testing-container.git
```

build the images from the **Dockerfile**s
```
./build-images.sh
```

run the tests:
```
docker-compose run uitests mvn verify
```
stopping and removing the containers:
```
docker-compose stop
docker-compose rm
```


### Conclusion
**Containers are awesome**, I'm really buying into the hype and think they will definitely play a major role at least in helping reaching the next level of automation for testing that require more complex setups. 

We've managed to simply our whole setup as we no longer force the reader to run tedious steps to configure the testing environment. 

If we'd have pushed our **uitests** image to the [docker image registry](https://hub.docker.com/explore/) which offers free hosting for public repos, we could the whole would have been
**checkout the github repo with the integration tests** and run 
```
docker-compose run uitests mvn verify
```


With all the services popping up offering the possibility to deploy and run docker containers, while docker alternatives like [rocket](https://rocket.readthedocs.org/en/latest/) are being developed, it's definitely worth to keep an eye on evolution of container based packaging and deployments.


#### References
[Docker Compose](https://blog.codeship.com/orchestrate-containers-for-development-with-docker-compose/)
