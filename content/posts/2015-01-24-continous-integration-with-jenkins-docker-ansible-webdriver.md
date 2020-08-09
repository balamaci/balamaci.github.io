+++
author = "Serban Balamaci"
date = 2015-01-24T20:09:22Z
description = ""
draft = false
slug = "continous-integration-with-jenkins-docker-ansible-webdriver"
title = "Part 1 - Continous Integration with Jenkins, Docker, WebDriver ..."

+++


### The plan
I'm going to start the series on Continuos Integration(CI) for **java** web projects with explaining the part related to **integration tests**, then in Part 2 talk about how we glue things together in **Jenkins**.

- Use **Jenkins** jobs for running unit tests and coverage report. Building our artifacts.

- Start up a standard **Tomcat7**(Jetty or whatever) webserver in an isolated **docker container** with limited/guarded memory and cpu power with the same base image(or the same setup of our production env). 

- **Deploy** the **war** artifact in the dockerized Tomcat container.

- Run **integration tests** using **Selenium WebDriver** tests against this Tomcat instance.

- If all tests pass, have a manual trigger to deploy on a XXXXX env. 
- Use **Ansible** to deploy to the XXXX env.

### Part 1 - Integration Tests

Following the github workflow with submitted **Pull Requests** that the developer needs to review before merging to a long lived branch like the master or develop branch(depending what git flow you adhere to). I think it's a good thing that before we merge the PR to one of the main  branches and risk poluting the project with a failing feature, we need to automatically run the tests against that PR. 
In order to reach a higher level of confidence in your builds, deploying and testing the build as a whole and not solely rely on unit test is certainly something we can do.
Luckly automatically directing a Jenkins build and test job against a github PR is also something which can be achieved and described [here](http://www.dereuromark.de/2014/03/04/continuous-integration-with-jenkins/). 

### The code first
If you first want to see the code, you can find it [here](https://github.com/balamaci/pippo-demo-webdriver-test)

### Starting with Docker
I'm not going to reiterate what docker is because I already did it [here](http://balamaci.ro/docker-series-intro-to-docker/), but just a summary on what it's about:

You can think of Docker as a very lightweight VM which doesn't bring much added overhead. Or you can relate to **a sandboxing environment** on your phone in which the phone apps don't know / see each other, can't mess with each others data. Besides the sandbox isolation you get excelent **portability**. 
Once you build say for ex, a custom Tomcat7 docker image(and you can use my supplied build file to generate one) you can reuse it anywhere to start as much containers as you'd like without much more performance overhead than say just starting the 2,3,n separate instances of Tomcat.

The container will be fully customized to have whatever is needed to run your application. You can have java, maven, git, tomcat already configured with all the required dependencies and baked into the container image, **frozen** in place with your application and ready to be run on any other machine. By deploying and running in production the same setup you use to run the tests on, it makes sense to think there should be a lower chance of surprises happening. 

One other thing in favor of portability is that you can use through **docker push**(like a **git push**) the **imutable** image to a docker repository and **docker pull** from the repository when you want to build a new container from that image. This gives us the option to think of using the docker image as the deliverable itself and deploy them on different cluster nodes through a managing platform like [Kubernetes](http://www.infoq.com/articles/scaling-docker-with-kubernetes) or [Messos](http://codefutures.com/mesos-docker-tutorial-how-to-build-your-own-framework/) dynamicaly starting/stoping more instances on the fly while the load grows or shrinks. While this sounds cool but I've yet to work with them. Still **this approach could  provide a different view on whole microservice ecosystem**. 

You can share data between the host and the containers through **volumes** which could be used for passing configuration files to an otherwise "generic" container. Volumes are also used to map the log files directory for log files created inside the container.

You can **link** containers between themselves, like having your application inside Tomcat containers talking to Mysql/Postresql containers, sort of a DependencyInjection in which the DB container would be injected by an orchestration service at runtime, linking to QA or PRELIVE databases.

#### Jenkins and Docker inside a VM - Vagrant
Now because docker containers are much lighter than a VM, because it does not bother with hardware virtualization, and it does away with the hypervisor and instead is sharing the kernel with the host system. But if you are running Windows or MacOs that would be a problem(There is an solution to this: **boot2docker** a minimal vm that allows to run docker on Windows and MacOS), so it could be a good choice to just package a configured Jenkins along with the Docker setup inside as a Vagrant machine. 

Since we also want to run the integration tests using webdriver, "driving" a **phantomjs** browser, which will allow for faster runs(I'll explain a little about webdriver later). **Phantomjs** in turn requires **nodejs** to be installed so this piles up into quite a custom setup.

### The setup in more detail
Again, the final setup can be found [here](https://github.com/balamaci/pippo-demo-webdriver-test)
I like to set up a different project for the integration test, because they usually have different dependencies(like webdriver, nodejs, phantomjs) and are running on Jenkins, not on the developer machine because most likely they will take considerable longer time to execute than the unit tests. There is no problem to organize and keep integration tests inside the project along with the unit test as shown [here](http://www.petrikainulainen.net/programming/maven/integration-testing-with-maven/)

I also have **a separate job on Jenkis running the integration tests**. This also might help if we want to run them less frequently than the unit tests.
We asume a Jenkins upstream job took care of the war packaging and we'll just **"Copy artifacts"** from the build job to bring them in our  integration job.

We're using Maven which has 3 phases in the default build scenario dedicated to integrationg tests and we'll use them according:
 
* **pre-integration-test** phase we look for the Tomcat docker image, pull it from the repository if it's not already localy stored. We start a new docker container based upon this image. Deploy our app inside the Tomcat webserver.
* **integration-test** we use **Webdriver**(to "drive" a headless browser) to **run JUnit tests** against the application deployed inside the Tomcat container.
* **post-integration-test** phase we **stop the docker container**, clean up.


### Creating a docker container.
So we want to start a Tomcat webserver in an docker container to deploy our web app.
First you need to have Docker installed. [Install docker](https://docs.docker.com/installation/ubuntulinux/)


To pull the Tomcat docker image and start the Tomcat docker container **I chose to use a bash script**. 
The base script uses my [Jdk7 + Tomcat7](https://registry.hub.docker.com/u/balamaci/tomcat7_jdk7/dockerfile/) image.
You can use the **Dockerfile** as reference if you want to build your own and publish it to the docker repository. To build an image from a Dockerfile in the current directory:

<pre>
sudo docker built -t image_name .
</pre>

There are lots of maven plugins(this [one](https://github.com/rhuss/docker-maven-plugin) looks most promissing to me) for retrieving images and starting containers. You may want to check the maven plugin approach for a more portable solution (the maven plugins use the a rest endpoint also exposed by docker, instead of the local linux socket file so it should also work with a Windows/MacOS host with boot2docker). I just want to show a more "bare metal" solution to understand there is no hidden magic happening.

```bash
docker_output_dir="${DOCKER_OUTPUT_DIR}"
docker_image_name="${DOCKER_IMAGE_NAME}"

# the directory where the Tomcat container will output it's log files
docker_tomcat_log_dir=${docker_output_dir}/log

# file where we'll store container info after we start it
docker_container_property_file=${docker_output_dir}/container.properties

## pull image from repository if not already localy available
docker images | grep $docker_image_name
if [[ $? != 0 ]]; then
   docker pull $docker_image_name
fi

mkdir -p ${docker_output_dir}
mkdir -p ${docker_tomcat_log_dir}

## start a container from the image with the log files from tomcat mapped to the host
container_id=$(docker run ${docker_run_args} -v ${docker_tomcat_log_dir}:/opt/tomcat/logs -d ${docker_image_name})
container_ip=$(docker inspect --format '{{.NetworkSettings.IPAddress}}' ${container_id})

# writing the container properties in java properties format to be read and exposed to next maven phase
echo "CONTAINER_ID=${container_id}"$'\r' > ${docker_container_property_file}
echo "CONTAINER_IP=${container_ip}" >> ${docker_container_property_file}
```

For Jenkins to be able to invoke docker commands, the user under which jenkins runs, needs to be added to the **docker** group, but also setting another owner for the docker socket file should work:
```bash
sudo nano /etc/default/docker
```
and set
**DOCKER_OPTS="--host=unix:///var/run/docker.sock -G jenkins"**

The [start-docker-tomcat.sh](https://github.com/balamaci/pippo-demo-webdriver-test/blob/master/scripts/start-docker-tomcat.sh) script first checks that we have the docker image locally available, otherwise it will pull it from the docker registry. 
  

Since we are invoking a script file, we'll be using the **maven-exec-plugin** bound to the **pre-integration-test** build phase
```xml
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
                   <environmentVariables>
                     <DOCKER_IMAGE_NAME>balamaci/tomcat7</DOCKER_IMAGE_NAME>
                     <DOCKER_OUTPUT_DIR>${project.build.directory}/docker_output_dir</DOCKER_OUTPUT_DIR>
                   </environmentVariables>
                   <executable>${project.basedir}/scripts/start-docker-tomcat.sh</executable>
              </configuration>
         </execution>
       </executions>
</plugin>
```
I'm passing as a property from maven the Tomcat7 image named **balamaci/tomcat7\_jdk7** available from the docker repository, ofcourse you can use your own.

### Deploying the war file
Right now we started the container, however docker assigned it a dynamic IP. But we've dumped them in a **container.properties** where they look like:

CONTAINER\_ID=583dba4b8f2102d040433e8133da86651ac638f2b5edfa8a72f70e55e42be291
CONTAINER\_IP=172.17.0.10


We're using another helpful maven plugin the **properties-maven-plugin** to read these values and expose them to the next build phases
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

So these properties will be available down the build phases and **we can also inject/read them in our tests**, so we can reference the tomcat IP where the application will be deployed. 

Now we proceed to deploy the war file(which is already provided by an upstream Jenkins job) through the [cargo maven plugin](http://cargo.codehaus.org/Maven2+plugin) plugin.
```xml
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
              <cargo.servlet.port>8080</cargo.servlet.port>
	      <!-- We've added the so we are guaranteed to have active the following admin user-->
              <cargo.remote.username>admin</cargo.remote.username>
              <cargo.remote.password>admin</cargo.remote.password>
           </properties>
       </configuration>
       <deployer>
          <type>remote</type>
       </deployer>
       <deployables>
          <deployable>
             <type>war</type>
             <location>${war.file.location}</location>
                <properties>
                    <context>${webapp.deploy.context}</context>
                </properties>
         </deployable>
       </deployables>
     </configuration>
     <executions>
        <execution>
          <id>war-deploy</id>
          <phase>pre-integration-test</phase>
          <goals>
             <goal>deploy</goal>
          </goals>
        </execution>
     </executions>
</plugin>
```
Worth mentioning that we also had the option of adding the .war to the base Tomcat docker image through the **ADD** command and create a new image from which we can start a container. This is how the maven docker plugins work, but I just wanted to simulate a closer approach to how I and most developers deploy web apps(as war files instead of docker images). 

### Pippo Framework
The web app I'm using as example is the demo for  [Pippo](https://github.com/decebals/pippo) a web framework designed from the begining to be highly modular(and so very light size) **jee servlet3** web framework which is great for serving template files and a REST enpoint. 
```java
POST("/contact", (request, response, chain) -> {
    String action = request.getParameter("action").toString();
    if ("save".equals(action)) {
        Contact contact = request.getEntityFromParameters(Contact.class);
        contactService.save(contact);
        response.redirect("/contacts");
    }
});

GET("/template", (request, response, chain) -> {
        Map<String, Object> model = new HashMap<>();
  model.put("greeting", "Hello my friend");
  response.render("hello", model);
});
```

### Running the [Webdriver](#webdriver) tests
Because we also have Javascript in our web application, it's not enough for the tests to just run GET/POST request against the endpoints. 
We'd rather use a real browser(or browser engine) that executes the javascript.

**Selenium Webdriver** through it's API allows us to control real browsers to perform automation. There are more concrete implementations of Webdriver destined for different browsers. **FirefoxDriver** which drives a real firefox browser is quite slow so I've used instead **Phantomjs** which is a webkit implementation of a **headless browser** faster and lower on resources. Even if it's a headless browser, phantomjs is still able to take screenshots of the web pages, and we'll use this feature when our tests fail to understand quicker where the problem was.

We first need to bring the webdriver and phantomjsdriver dependencies into our testing project.

```xml
<dependency>
  <groupId>com.github.detro</groupId>
  <artifactId>phantomjsdriver</artifactId>
  <version>1.2.0</version>
</dependency>
```

### Phantomjs
The **PhantomJsDriver** expects a property to where the **phantomjs binary** is located, so we need to install phantomjs first. But phantomjs also requires [nodejs](http://nodejs.org/) to be installed. Luckly once **node** is installed and we have created a **./node-modules** directory in our project a simple, 
**npm install phantomjs** should be enough to install phantomjs under the **./node-modules** of the current directory and we can can inject the by configuring **maven-failsafe-plugin** like below:
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <version>2.18.1</version>
    <configuration>
        <systemPropertyVariables>
            <phantomjs.binary.path>${phantomjs.binary.path}</phantomjs.binary.path>
            <container.host>${CONTAINER_IP}:8080</container.host>
            <webapp.deploy.context>${webapp.deploy.context}</webapp.deploy.context>
            <webdriver.screenshot.path>${build.directory}/screenshot</webdriver.screenshot.path>
        </systemPropertyVariables>
        <includes>
            <include>**/Test*.java</include> 
        </includes>
    </configuration>
    <executions>
        <execution>
            <id>failsafe-integration-tests</id>
            <phase>integration-test</phase>
            <goals>
                <goal>integration-test</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### The Tests
Finally we get to talk about our integration tests. I created a **BaseWebdriverTest** class shown bellow, in where we reference the properties injected from maven to create the **WebDriver** object through which we control the phantomjs driver. All our tests will extends this class and have access to the **WebDriver** object. To be noted that we are working with an interface. It will be easy to switch the driver implementation to another implementation like the Firefox driver.

```java
public abstract class BaseWebdriverTest {

    @Rule
    public TakeScreenshotOnFailedTaskRule screenShootRule =
            new TakeScreenshotOnFailedTaskRule();

    private WebDriver driver;

    private static PhantomJSDriverService phantomJSDriverService;

    protected static String serverUrl;
    protected static String appContext;

    @BeforeClass
    public static void init() throws Exception {
        serverUrl = "http://" + System.getProperty("container.host");
        appContext = System.getProperty("webapp.deploy.context");

        phantomJSDriverService = PhantomJSDriverService.createDefaultService();
        phantomJSDriverService.start();
    }

    @Before
    public void before() {
        DesiredCapabilities capabilities = DesiredCapabilities.phantomjs();
        capabilities.setJavascriptEnabled(true);
        capabilities.setCapability(CapabilityType.TAKES_SCREENSHOT, Boolean.TRUE);

        LoggingPreferences prefs = new LoggingPreferences();
        prefs.enable(LogType.BROWSER, Level.WARNING);
        prefs.enable(LogType.DRIVER, Level.INFO);
        capabilities.setCapability(CapabilityType.LOGGING_PREFS, prefs);

        driver = new PhantomJSDriver(phantomJSDriverService, capabilities);

        screenShootRule.setDriver(getDriver());
    }

    @After
    public void after() {
        driver.close();
    }

    protected WebDriver getDriver() {
        return driver;
    }

    @AfterClass
    public static void globalTearDown() {
        if (phantomJSDriverService != null) {
            phantomJSDriverService.stop();
        }
    }

    public String getBaseUrl() {
        String baseUrl = appendSlashOnRightSide(serverUrl) + appContext;
        return appendSlashOnRightSide(baseUrl);
    }
}
```
the **TakeScreenshotOnFailedTaskRule** is used to save screenshots whenever a test fails to better help us see what went wrong.

```java
public class TestCrudNgDemoApp extends BaseWebdriverTest {

    @Test public void
    forAuthorizedUserLoginSuccessful() {
        WebDriver driver = getDriver();
        driver.get(getBaseUrl() + "login");

        fillUserLoginData(driver);

        WebDriverWait wait = new WebDriverWait(driver, 5);
        wait.until(new Function<WebDriver, Boolean>() {
            @Override
            public Boolean apply(WebDriver webDriver) {
                return webDriver.getTitle().equals("Contacts");
            }
        });
    }

    private void fillUserLoginData(WebDriver driver) {
        String username = "admin";
        WebElement txtUsername = driver.findElement(By.name("username"));
        txtUsername.sendKeys(username);

        String password = "password";
        WebElement txtPassword = driver.findElement(By.name("password"));
        txtPassword.sendKeys(password);

        WebElement btnLogin = driver.findElement(By.name("login"));
        btnLogin.click();
    }
}
```

### Running
The whole setup can be simply run with
**mvn verify** which will 

 * start the docker Tomcat container through the **start-docker-tomcat.sh** script through **exec** plugin. 
 * read the container info(IP, CONTAINER_ID)
 * deploy the war through the **cargo** maven plugin
 * run the tests
 * teardown and cleanup through **stop-docker-tomcat.sh**.

### Conclusion
We have mixed different technologies into a working setup that allows us to write integration tests quickly and without being too dificult and so we added another tool to improve the quality of our builds.

### Last minute thoughts
I also mentioned  about **being able to limit the cpu and memory resources of docker containers**. You can read [here](https://goldmann.pl/blog/2014/09/11/resource-management-in-docker/) about it. Could be useful if we need to run multiple integration jobs on the same instance. It might been an interesting idea to research if we can package the webdriver setup as a separate container and have it also cpu/memory bound in case the webapp, webdriver or  phantomjs browser have memory leaks.

Hope you found it usefull and maybe look forward for Part II into which I discuss about setting up a Jenkins Pipeline.