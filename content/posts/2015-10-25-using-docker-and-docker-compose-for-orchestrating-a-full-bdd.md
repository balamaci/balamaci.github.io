+++
author = "Serban Balamaci"
categories = ["docker", "docker-compose", "selenium-grid"]
date = 2015-10-25T17:40:02Z
description = "We use docker and docker-compose to build a reusable setup for testing java web applications based upon containers"
draft = false
image = "/images/2015/10/flow1.svg"
slug = "using-docker-and-docker-compose-for-orchestrating-a-full-bdd"
tags = ["docker", "docker-compose", "selenium-grid"]
title = "Orchestrating a reusable BDD web testing setup with Selenium grid using docker - Part I"

+++


### Playing with lego components
#### The plan:
Part I - **We use [docker](https://dzone.com/refcardz/getting-started-with-docker-1) and docker-compose to build a reusable setup for testing java web applications based upon containers**. 

[Part II](https://balamaci.ro/orchestrating-a-reusable-bdd-web-testing-setup-part-ii/) - We introduce Cucumber BDD testing framework along with Serenity framework for providing great tests reports.

One of the strong points of using **docker** is the fact that it **gives us the option to think in terms of high level components** (database, web container, application, integration container) that we can link together or swap around, sort of like playing with lego bricks. I don't know about advising you to run your setup in production with docker, but it definitely makes sense to base your testing setup on it. Play around with containers swap in a database container with test data, or a web container is a good way of mocking dependencies.

![Lego](https://balamaci.ro/assets/2015/lego.png)

Having this option means we can have a **simplified view of things that would otherwise seem complex**. 


Add into the picture that you may reuse some of those high level components or just take advantage of some already made => this saves on time with micromanagement of getting those component working or keeping them updated.
So far this sounds just like marketing statements so let's see how we can use it in the real world scenario where 

### Selenium grid
When doing automated browser testing, I mostly relied on **phantomjs** as it being a headless browser which has a low consumption of resources. 
But if we wanted to use some real browsers one would choose something like **'selenium grid'**.
**Selenium grid** consists of a **central hub** which allows **nodes** with real browsers to **register with the hub**. When you want to test you application against a certain browser, you request instances of that browser from the hub and operate(drive) that browser through the hub, which acts like a proxy. 

![Selenium Grid](http://www.seleniumframework.com/wp-content/uploads/2015/01/Overall_GRID_architecture.png)

Using the selenium grid has the benefits of using actual browsers and if you're registering more nodes with it, **the possibility to run tests in parallel**.

While in your webdriver configuration you need only to be concerned with the selenium hub endpoint, getting the grid up and running it's if not very tedious, at least time consuming.

Would it not be cool to have this inside some docker images and ready to run with a single command so that we don't even need to spend time reading the documentation of **selenium-grid** to find out how or what you need to get it working?

### Enter 'docker-selenium'
[docker-selenium](https://github.com/SeleniumHQ/docker-selenium) is exactly what we were looking for: preexisting docker images of the hub and Firefox and Chrome nodes. Through the concept of container linking the containers for hub and nodes are made aware of each other and thus the nodes are registering with the hub. All we'd need to get them working would be:
```
#starting the hub
$ docker run -d -p 4444:4444 --name selenium-hub selenium/hub:2.48.1
```
the 'selenium-hub' container exposes port 4444 for incoming connections.

```
#starting a node with firefox
$ docker run -d --link selenium-hub:hub selenium/node-firefox:2.48.1

#starting a node with chrome
$ docker run -d --link selenium-hub:hub selenium/node-chrome:2.48.1
```


This is great because now we don't need to worry ourselves to even find out how to get Selenium grid up and running. 
Questions like: *'Do we need java installed to have it running? 
What are the commands to launch it?'* are no longer our concern. 

Also, notice the version number *'2.48.1'*, by the time you read this, very probably there will be a newer version available. This means all we'd need do to keep up to date with the latest version of [docker-selenium](https://github.com/SeleniumHQ/docker-selenium) would be to just change just a number - the version of the whole dependency-. We've gone from updating versions number of libraries in pom.xml to updating the version number for a whole packaged setup.

We just started the container images through a simple docker command(actually two commands but we'll fix even this further on when we talk about **docker-compose** :).

**At this point we at least got a selenium grid up and running through docker** ready for a webdriver client to connect to the selenium-hub container on port 4444.

Let's go even further in our lego game.


### The testing container
In a [previous post](https://balamaci.ro/docker-container-for-visual-regression-testing/) I proposed a reusable setup of running ui webdriver tests. 
We basically had a **Tomcat container** and an **ui-tests container**. The **ui-tests container** is actually just a plain java+mvn3 container(we only call it that to better differentiate it in the pictures, or to show that you are not restricted to just java & maven, and you can add any other dependencies). 

This **ui-tests**(java+mvn3) container would be used to execute **mvn** with an external supplied **pom.xml** which will take an external .war file, deploy it in the **Tomcat container**(through **maven-cargo plugin**) and **run webdriver tests against the deployed application in Tomcat container**. 


To be clear, the **ui-tests** container is immutable, an executor of maven [pom.xml](https://github.com/balamaci/blog-ui-bdd-testing/blob/master/pom.xml) - the gluecode to deploying the application inside the tomcat container and running the tests in '/src/main/java' and bring in any java dependencies needed to run the tests-.

All the dockerfiles necessary to create the images are under [docker folder](https://github.com/balamaci/blog-ui-bdd-testing/tree/master/docker). 

We're using the angular crud demo of the [pippo](https://github.com/decebals/pippo) web framework **pippo-demo.war** as example for our tests. We've also pushed it to github to be easier to run the tests without having to build it yourselves.

![Containers](https://balamaci.ro/assets/2015/flow1.svg)


### Putting the bricks(containers) together
It only makes sense now to put together the selenium-grid(selenium-hub and how many selenium-nodes containers)
together with the **Tomcat** and **ui-tests** containers with **the scope of running the webdriver tests through the selenium-hub** 

![containers-linked](/assets/2015/containers_together.svg)

The flow:

 - **ui-tests** container is sharing a volume directory with the host through which it receives the **pippo-demo.war**(**pippo-demo.war** could be generated by another jenkins job which passes down the build .war artifact to our ui-tests jenkins job), **pom.xml** and the sources for the tests(which we placed them in '/src/main/java'). 
 - **ui-tests** container deploys the .war file through the 'cargo plugin' into the **tomcat** container.
 - **ui-tests** through the shared directory with the host begins executing our **cucumber-serenity**(discussed in [Part II](https://balamaci.ro/orchestrating-a-reusable-bdd-web-testing-setup-part-ii/)) that it finds there.
 - **selenium-hub** container acts as a proxy for the webdriver commands it receives from the webdriver tests executed inside **ui-tests** container.
 - **selenium-node** container(s) act as browsers opening our application deployed inside the **tomcat container** executing the webdriver commands received through the hub.

Through the shared volume directory we also pass the /src/main/java into which we'll have our **cucumber-serenity** tests(discussed in part II).


### docker-compose - simplified orchestration
As the number of container increases, the rules on how they depend on one another, how they link together, what ports they expose and what they share together with the host, this information can get messy. This is where [docker-compose](https://blog.codeship.com/orchestrate-containers-for-development-with-docker-compose/) can help us with describing all those details in a **docker-compose.yml** file then with the use of a single command 
```
docker-compose run mvn clean verify
```
I've also used docker-compose in the [previous post](https://balamaci.ro/docker-container-for-visual-regression-testing/) and it seem I can't get enough :).

Let's see how we this file looks for our setup:
```
tomcat:
  image: balamaci/tomcat7-jdk8
  ports:
    - 8080:8080
  volumes:
    - ./out/tomcat:/opt/tomcat/logs

hub:
  image: selenium/hub:2.48.1
  expose:
    - 4444:4444
  links:
    - tomcat

nodeff:
  image: selenium/node-firefox:2.48.1
  volumes:
    - ./out/ff/e2e/uploads:/e2e/uploads
  links:
    - hub
    - tomcat

uitests:
  image: balamaci/ui-tests
  volumes:
    - .:/usr/src/app
    - ~/.m2:/root/.m2
  links:
    - tomcat
    - hub
    - nodeff #forces instantiation of nodeff
```

Sources can be be downloaded from [github](https://github.com/balamaci/blog-ui-bdd-testing)

I think it's pretty self evident what each line is about, however some may need some explaining: 

   - **ui-tests** in theory you don't need link with the **nodeff** firefox grid node, but if not referenced from any other container, docker-compose will see no reason to start it.
   - ui-tests volumes:
    - **'.:/usr/src/app'** through this we share the **pom.xml** and the **src/** folder where the tests are. We also get the **target** folder as output 
    - **'- ~/.m2:/root/.m2'** is just a speedup optimization  that because we destroy and recreate the containers with every run, we don't want maven to download the dependencies again and again for every run, instead we want the new containers to reuse them, that's why we share our home dir maven - could be any other directory mapped to the container's */root/.m2* folder.

all we need now is to run 
```
docker-compose run mvn clean verify
``` 

this will invoke **maven-failsafe-plugin** in our [pom.xml](https://github.com/balamaci/blog-ui-bdd-testing/blob/master/pom.xml) which will to run the **Cucumber tests** against our **pippo-demo.war** application deployed inside the Tomcat7 container and generate the **Serenity reports** of which we talk about in [Part II](https://balamaci.ro/orchestrating-a-reusable-bdd-web-testing-setup-part-ii/)


### Conclusion
Structuring our workflow with **docker** and **docker-compose** has freed us from the tedious details of setting up a selenium-hub or keeping it updated. 

We have now have a blueprint setup to use with ui testing for all our java web application.
We saw in the **docker-compose.yml** that it's not hard to have a group of containers talk to each other. This opens the way to add and swap in more containers for mocking dependencies, databases for testing, etc.
Take into considerations the cloud providers offering for deploying docker containers and it might just mean we no longer need to concern ourselves with infrastructure even for running testing with actual browsers.

We're instead free to focus on writing the actual webdriver tests for which we'll introduce in the [second part](https://balamaci.ro/orchestrating-a-reusable-bdd-web-testing-setup-part-ii/) - **Cucumber** and **Serenity** two great frameworks that simplify our work and produce the following end result - **clear test reports along with printescreens** that give a real understanding through natural language of the business domain to what has been covered by automated tests, what is working and what is not:


[![BDD preview](/assets/2015/bdd_preview.png)](https://balamaci.ro/static/serenity/index.html)
