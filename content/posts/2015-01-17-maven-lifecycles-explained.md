+++
author = "Serban Balamaci"
categories = ["maven"]
date = 2015-01-17T11:42:14Z
description = ""
draft = false
slug = "maven-lifecycles-explained"
tags = ["maven"]
title = "Maven lifecycles and plugins explained"

+++


Maven is much more than just a simple dependency management and packaging tool. Through a set of **plugins** it becomes a powerful tool for handling the "backbone" of CI setup that might require generating new source files, compile, package, execute bash scripts that for ex. start a docker container, start integration tests, before finaly pushing the deliverables to a repository. 

And heck, **combined with Jenkins** it becomes so powerful it can even fight crime. Ok maybe not that powerful. Or at least so powerful it might make you think again if you really need **Gradle** to implement what you need.

Many java devs don't really pay it enough attention besides using it as a tool to build a package, and because of that I think it was necesary to write a short explanation and gather some usecase before we proceed with more interesting things like a CI series in the next articles.


## Maven default lifecycle
One of the reasons people are not satisfied with Maven is that it's got this **default lifecycle**(also called the build cycle) that actually consist of a set of **phases**(steps) that pretty much should cover the general steps of releasing a software project but may not fit always the needs of a project build and release. I'm just going to add a few words to those who are not self explanatory by their name.

- <span style="font-family: courier">[validate]</span> - Validates the project is correct and all necessary information is available to complete a build
- <span style="font-family: courier">[initialize]</span> - create directories, sets properties
- <span style="font-family: courier">[generate-sources]</span> - generate extra source files for example the jooql plugin is generating classes from the database metadata.
- <span style="font-family: courier">[process-sources]</span> - additional processing of the sources like rewriting, adding extra code, filtering files, etc.
- <span style="font-family: courier">[generate-resources]</span>
- <span style="font-family: courier">[process-resources]</span> - copy the resources to the target directory.
- <span style="font-family: courier">[compile]</span>
- <span style="font-family: courier">[process-classes]</span> - post-process the generated files from compilation, for example to do bytecode enhancement on Java classes
- <span style="font-family: courier">[generate-test-sources]</span> - like their generate-source/resource counterparts but for test classes
- <span style="font-family: courier">[process-test-sources]</span>
- <span style="font-family: courier">[generate-test-resources]</span>
- <span style="font-family: courier">[process-test-resources]</span>
- <span style="font-family: courier">[test-compile]</span> 
- <span style="font-family: courier">[process-test-classes]</span>
- <span style="font-family: courier">**[test]**</span> - runs the unit tests and generates the test report, this is the job of the **Maven Surefire** Plugin.
- <span style="font-family: courier">prepare-package</span> 
- <span style="font-family: courier">**[package]**</span> - packages the application.
- <span style="font-family: courier">[pre-integration-test]</span> - runs before the integration tests, I use this for ex. to start a Tomcat container into which to deploy the war package built above.
- <span style="font-family: courier">[integration-test]</span> - performs the integration tests, this is the job of the **Maven Failsafe** plugin.
- <span style="font-family: courier">[post-integration-test]</span> - cleans up after the integration tests, like stoping the container
- <span style="font-family: courier">**[verify]**</span> - checks that the final package meets the quality criterias - checkstyle plugin for ex is bound to this phase. 
- <span style="font-family: courier">**[install]**</span> - Install the jar archive into the local Maven repository(the .m2/repository directory).
- <span style="font-family: courier">**[deploy]**</span> - Install the jar archive into a remote  Maven repository.

These build phases are executed sequentially in **the given order**. 
Calling one phase of the lifecycle results in the execution of all phases up to and including that phase.
To do all the steps, you only need to call the last build phase to be executed, in this case  **mvn deploy**. Calling **mvn verify** will do the all the steps except **install** and **deploy**. 


### Goals and Plugins
Now we've seen that Maven defines this steps(phases), but the ideea is that **you can attach one or more  actions** to them as they get stepped through. These tasks or units of work are called **goals**. I like to think of them as the "public methods" of the **plugins** into which goals are bundled to provide some highlevel functionality. 
Plugins themselves can be referenced in the project pom.xml in the **plugins** section. 
```prettyprint lang-xml
<plugins>
...
</plugins>
```

They will be resolved and downloaded by maven from the repository and their functionality can be bound to the build phases(thought the plugin goals) so that the **goals gets executed if that phase is reached**.
As an example I'm attaching to the **pre-integration-test** phase the **exec** goal of the **exec-maven-plugin** which is supposed to execute the specified executable - in this case starting a docker container which contains a Tomcat server into which we plan to deploy the packaged application-.

```prettyprint lang-xml
<plugins> 
...           
   <plugin>
          <groupId>org.codehaus.mojo</groupId>
          <artifactId>exec-maven-plugin</artifactId>
          <version>1.2.1</version>
          <executions>
               <execution>
                    <id>start-tomcat-docker</id>
                    <goals>
                        <goal>exec</goal>
                    </goals>
                    <phase>pre-integration-test</phase>
                    <configuration>
                        <environmentVariables>                                                <DOCKER_IMG>balamaci/tomcat7</DOCKER_IMG>
                        </environmentVariables>
                        <executable>${project.basedir}/scripts/start-docker-tomcat.sh</executable>
                        </configuration>
                </execution>
           </executions>
   </plugin>
</plugins>
```

You can **invoke a maven goal separate from the build cycle** on it's own. 
<pre>
mvn exec:exec
</pre>

You can bind **more than one goal of the same or to different plugins to a build phase**, and they will be executed in the order in which they apear in the pom.xml


### Other lifecycles
Now you might have asked yourself where probably the most used command **mvn clean** fits into the equation since it's not mentioned as a lifecycle phase.
Well that is because **clean is a lifecycle** with it's list of phases.

- pre-clean
- clean
- post-clean

**mvn site** is another lifecycle used to generate documentation.

Also in theory you can also create your own custom  lifecycle along with customized phases, but I've never had to, so I cannot comment on this...

### Default bindings  
Maven, even if you don't explicitly add something in the **plugins** section to bind any **goal->phase** appears to be doing quite a lot of things itself when you invoke a build phase like **mvn package**. 
This is because maven has **by default bounded some plugin goals to the default build lifecycle**. Exactly what plugin goals are bound **differs depending on the type of packaging**: pom, jar, war, ejb, etc). 

You can override and customize the default configuration of these plugins by adding them to the **plugins** section and passing in your required configuration.   

### Maven Profiles
Build profiles are helpful to further customize and run different build setups. 

You might want this for different reason, most likely go through a speedier build setup on your development machine vers what needs to be done when running on Jenkins. 
Or another realcase example in case of a web app which contains a lot of js files, you might want to combine and minify the files when releasing a production version, but you might want to skip this step when just running the application in development.

Profiles are defined like:
```
<profiles>
  <profile>
    <id>development</id>
      <activation>
         <activeByDefault>true</activeByDefault>
      </activation>
      
      <!-- we can define properties here... -->
      <properties>
          <skip.integration.tests>true</skip.integration.tests>
      </properties>
      <!-- add different dependencies -->
      <!-- add or configure plugins differently-->
      <build>
         <plugins>
           <plugin>
              <groupId>com.mysema.maven</groupId>
              <artifactId>apt-maven-plugin</artifactId>
              <version>${mysema.apt.version}</version>
              <dependencies>
                  <dependency>
                    <groupId>com.mysema.querydsl</groupId>
                    <artifactId>querydsl-apt</artifactId>
                    <version>${querydsl.version}</version>
                  </dependency>
              </dependencies>
              <executions>
                <execution>
                  <phase>generate-sources</phase>
                  <goals>
                     <goal>process</goal>
                  </goals>
                  <configuration>
                     <outputDirectory>target/generated-sources/queries</outputDirectory>
                     <processor>com.mysema.query.apt.jpa.JPAAnnotationProcessor</processor>
                  </configuration>
                </execution>
              </executions>
        </plugin>
       </plugins>
  </profile>
  <profile>
    <id>production</id>
    <!-- ...and here we can define other sets of properties-->
    <!-- ...other dependencies and plugins-->
  </profile>
</profiles>
```
We see that by default 'dev' is active, we can easily use the production profile by doing **mvn -P production**


### Usecases and the plugins for them


#### Releasing a new version
For performing a new release of a project I use the [maven-release-plugin](https://maven.apache.org/maven-release/maven-release-plugin/examples/prepare-release.html). The plugin itself does a lot of stuff under the hood 

- **mvn release:prepare**
- **mvn release:perform**
- **mvn release:stage** 


### Packaging your app as a "fat" jar
Sometimes you just want to handle a single deliverable to be easier to deploy and run a java application. The problem is that your .jar app file is depending on other jars, but the good news is you can rebundle the dependencies classes inside your jar, obtaining a "fat" jar through **maven-shade-plugin** explained [here](http://www.mkyong.com/maven/create-a-fat-jar-file-maven-shade-plugin/), and while it's a better option than the **maven-assembly** plugin. 

**Spring boot** does this "fat jar" thing with the their [spring-boot-maven-plugin](http://docs.spring.io/spring-boot/docs/current/reference/html/build-tool-plugins-maven-plugin.html) 

For other ways of packaging your java application might have a look at [capsule](https://github.com/puniverse/capsule) to see what it does.

### Getting code coverage reports with Jacoco
Already a great explanation [here](http://www.petrikainulainen.net/programming/maven/creating-code-coverage-reports-for-unit-and-integration-tests-with-the-jacoco-maven-plugin/).

### Executing scripts
Sometimes in your build you might want to execute a script file.
Already gave an example of the **exec-maven-plugin** plugin.


### Reading properties from files.
Maybe you need to provide the conection information for a plugin which needs a database connection.
Or you need to read into your build some value that was generated externally like from a bash script like the case bellow where I read the started docker container IP(which was outputed in a properties file).
```prettyprint lang-xml
<plugin>
   <groupId>org.codehaus.mojo</groupId>
   <artifactId>properties-maven-plugin</artifactId>
   <version>1.0-alpha-2</version>
   <executions>
       <execution>
          <id>read-container-properties</id>
          <phase>pre-integration-test</phase>
          <goals>
             <goal>read-project-properties</goal>
          </goals>
              <configuration>
                 <files>
                    <file>${docker.output.dir}/container.properties</file>
                 </files>
              </configuration>
      </execution>
   </executions>
</plugin>
```

### Doing integration tests apart from unit tests
While in Unit Tests you'd mock the response of external dependencies and test components/classes of you project in isolation, Integration tests are the one  covering the system as a whole, the components glued together even maybe with some actual real external dependencies with the idea of having less surprises in production system not working the same as their mocked versions. 
Now because integration tests have a higher cost to run than the unit tests, you're going to want to execute them separate from the unit tests and maybe less often and on a dedicated build system like **Jenkins**.

You also maybe want to be able to execute the whole build lifecycle without running the costly integration tests - see **Profiles** section above how it can be easily done.

Running of the unit tests are handled by the separate plugins again:

- **surefire** -> unit tests
- **failsafe** -> integration tests
and we can configure these two plugins separately as to use different convetion to distingu like classes starting with **Test\*** are unit tests and **ITest\*** could be the integration tests.


This I see after writting this :) has been improved upon [here](http://www.petrikainulainen.net/programming/maven/integration-testing-with-maven/) by keeping the integration classes in a different directory from the unit tests.
Just to mention here the [Build Helper Plugin](http://mojo.codehaus.org/build-helper-maven-plugin/) which is used to add extra source/resource dirs into the build, like the /src/main/integration-tests directory.  

I myself tend to keep the sources of the integration tests in a different repo because I don't run them localy, but through jenkins.


See [here](http://www.balamaci.ro) my next article about doing Integration tests with the help of Jenkins, Docker, Webdriver.

### Deploying a war file to a running app container
For this usecase the [maven-cargo](http://cargo.codehaus.org/Maven2+plugin) plugin can be used. For ex. I'm deploying the packaged app inside a test container as part of the **integration-test**
```prettyprint lang-xml
<plugin>
  <groupId>org.codehaus.cargo</groupId>
  <artifactId>cargo-maven2-plugin</artifactId>
  <version>1.4.11</version>
  <configuration>
     <container>
        <containerId>tomcat7x</containerId>
        <type>remote</type>
     </container>
     <configuration>
          <type>runtime</type>
          <properties>
            <cargo.hostname>${CONTAINER_IP}</cargo.hostname>
                <cargo.servlet.port>${TOMCAT_PORT}</cargo.servlet.port>
                <cargo.remote.username>${TOMCAT_ADMIN_USER}</cargo.remote.username>
                <cargo.remote.password>${TOMCAT_ADMIN_PASS}</cargo.remote.password>
          </properties>
     </configuration>
     <deployer>
         <type>remote</type>
     </deployer>
     <deployables>
        <deployable>
            <type>war</type>
            <location>${project.build.directory}/pippo-demo.war</location>
            <properties>
                 <context>pippo</context>
            </properties>
        </deployable>
     </deployables>
  </configuration>
  <executions>
     <execution>
       <id>war-deploy</id>
       <phase>integration-test</phase>
       <goals>
          <goal>deploy</goal>
       </goals>
     </execution>
  </executions>
</plugin>
```
