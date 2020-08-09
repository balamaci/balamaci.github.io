+++
author = "Serban Balamaci"
categories = ["webdriver"]
date = 2015-07-28T21:10:00Z
description = ""
draft = false
slug = "visual-testing-automation"
tags = ["webdriver"]
title = "A Solution for Automation of Visual Regression Testing"

+++


### What's the problem with visual regression testing automation
When you need to test a web application and not just an API, you need to check also that the changes in the **DOM** order and / or **.css** files **did not mess something up with the way the page renders for the user**. **Usually this is done manually by a tester** which gives his approval that after the changes **"the app still looks ok"**. 

Visual testing is something that gets tedious without even considering that there are more browsers to test and whole bunch of resolutions - now with responsive design-.

To give **an example a small change in the .css style file** might trigger in some place the **"Submit" button to become very small or even invisible**. Now the button is still in the DOM so **a check that the next page after calling ('#submit').click() will pass**, but **in reality the user will not easily be able to submit the form** and advance in the flow.

### A possible solution
The first time I stumbled upon a solution was [Huxley](https://github.com/facebookarchive/huxley) which was used by Instagram in the early days.

**The idea** is  
- We can **use Webdriver** to command **a series of browsers** and use it's capabilities to **take printscreens.** 

- We **take a series of printscreens before / after** every action that matters(changes the UI) during a Webdriver flow. This we consider the "correct / right" way of how the application must look to the user and we **use these images as reference**

- When we run the tests, we **take new printscreens and fail on any difference from the above mentioned "base" printscreens.**

We use a **diff tool for images** which will help **outline what has changed**. 

The **Huxley** way involved doing this in python, I'm just more comfortable to implement roughly the same idea but in Java to be able to integrate it in our test setup.


### Some drawbacks and disclaimers
I want you to be aware firsthand and not waste time reading on, as you may consider them deal breakers.
   
  - **If there is dynamic data in your tests they will fail** because the printscreens will be different.
  Example of dynamic data could be the **date / time** so if you display some entries with creation/update date, you'd need to adjust your mocks to show the same time. 
  
  - **Animations(gifs / carousels)** that are not predictable that could interfere with the printscreens and may trigger false tests fails.
  
  - After changes that you approve of, you need to recreate the base printscreens from which we judge that this is the way it's supposed to look. Since we can automate this, it's not really a problem I think.
  
#### Good news 
  - You can still **use this method to test components in isolation** something in the idea of **unit tests** but for visual components, vs doing integration tests for the whole application.

  - You can still **have an automatic way to generate printscreens of all the pages and panels in the application**. It's still quite nice that after running the automation of the printscreen generation you can take a quick glance how the whole app looks.

### Implementation
I've already talked about how to setup Webdriver and possible way of deploying our web app inside a docker container, but how can we determine that two printscreen images are different and once deemed different can we highlight the differences?

1. Deeming **two images equal** is the easier part. For example **we can do a fast hash function of the content(say md5)**.
2. For **highlighting the differences** well, we can use the all powerful **ImageMagick** program which allows for a command line input of image manipulation. 

Extra praise for **ImageMagick** which you can use as a Swiss army knife of image manipulation and command it quite easily from Java which by itself is not very versatile at . So, little advice to **consider it should you find yourself in need of doing image manipulation from java**. 

One of the many functions available in  **ImageMagick** is **image diff** so we can use this to highlight differences between printscreens.

And we can use **im4java** library which is just a wrapper through which command line parameters are passed to **ImageMagick**, which means **we must have ImageMagick separately installed on the machine where we run the tests**.
Luckly this is just very simple for Ubuntu.

```
sudo apt-get install imagemagick
```

and import the dependency in **pom.xml**
```xml
<dependency>
  <groupId>org.im4java</groupId>
  <artifactId>im4java</artifactId>
  <version>1.4.0</version>
</dependency>
```

The end result example
![Original image](https://raw.githubusercontent.com/balamaci/blog-ui-testing/master/printscreens/ContactsPage_800_600.ref.png)
after playing around with the css file, can you already spot it?
![Changed image](https://raw.githubusercontent.com/balamaci/blog-ui-testing/master/printscreens/ContactsPage_800_600.png)
Tadaaa here is **the diff image**:
![Diff image](https://raw.githubusercontent.com/balamaci/blog-ui-testing/master/printscreens/ContactsPage_800_600.diff.png)



### The code
Whole example project can be found [here](https://github.com/balamaci/blog-ui-testing)
**Update** In the next article [Docker containers for Visual Regression Testing setup](http://balamaci.ro/docker-container-for-visual-regression-testing/) I've improved on the solution by using **docker** to package the whole setup  (imagemagick with phantomjs) inside a container and with **docker-compose** to start this container along with a separate Tomcat container.

We start with **CrudNgDemoAppTest** which contains some simple tests for the Login Page.

```java
@RunWith(Parameterized.class)
public abstract class CrudNgDemoAppTest extends BaseUIWebdriverTest {

    @Parameterized.Parameters
    public static Collection<Object[]> getDimensions() {
        return Arrays.asList(new Object[][] {
                {DIMENSION_800x600}, {DIMENSION_1024x768}});
    }

    public CrudNgDemoAppTest(Dimension dimension) {
        super(dimension);
    }


    @Test public void
    loginPage() throws Exception {
        navigateTo("login");
        waitForElementWithName("username");

        takeScreenshotCompareAndCollectDiff("LoginPage");
    }
...
```
#### Passing parameters to JUnit tests
**@RunWith(Parameterized.class)** allows us to pass in different screen resolutions which are returned by _getDimensions_ method which is marked as the supplier of values with the **@Parameterized.Parameters** annotation.

Each _Dimension_ parameter get passed to the **public CrudNgDemoAppTest(Dimension dimension)** constructor. 

The _getDimensions_ method must return a **Collection&lt;Object[]&gt;** because the Test class constructor might take more than one parameter(although in our particular case it's just a single parameter).

The result is that we'll have **a quick automated way to see  how the application looks for different screen resolutions** which helps with testing the look on different mobile devices / desktop. And we can **add more resolutions at any time**. 


**BaseUIWebdriverTest** is our base class from which all our UI tests will inherit. It will contain the logic to take printscreens and helpers method for Webdriver commands(while writing this I stumbled upon [Selenide](http://www.methodsandtools.com/tools/selenide.php) which I plan to give it a try in the future).

```java
public abstract class BaseUIWebdriverTest {

    private Browser browser;
    private Dimension dimension;
    private static Boolean flagUpdateReferenceScreenshots;

    public static Dimension DIMENSION_800x600 = new Dimension(800, 600);
    public static Dimension DIMENSION_1024x768 = new Dimension(1024, 768);

    @Rule
    public ThrowErrorOnDiffScreensRule throwErrorOnDiffScreensRule = new ThrowErrorOnDiffScreensRule();

    public BaseUIWebdriverTest(Dimension dimension) {
        this.dimension = dimension;
    }

    protected abstract Browser initBrowser();

    @Before
    public void before() {
        browser = initBrowser();

        browser.changeWindowSize(dimension);
    }

    protected void takeScreenshotCompareAndCollectDiff(String scenarioName) {
        takeScreenshotAndCompare(scenarioName).ifPresent(throwErrorOnDiffScreensRule::addScreenDiff);
    }

    private Path takeScreenshot(String scenarioName, boolean isReference) {
        Path currentScreenshotPath;
        if (isReference) {
            currentScreenshotPath = getScreenshotReferencePath(scenarioName);
        } else {
            currentScreenshotPath = getCurrentScreenshotPath(scenarioName);
        }

        Path screen = browser.getDriver().getScreenshotAs(OutputType.FILE).toPath();
        try {
            Files.copy(screen, currentScreenshotPath, StandardCopyOption.REPLACE_EXISTING);
        } catch (IOException e) {
            throw new RuntimeException(String.format("Error saving image path=%s", currentScreenshotPath.toString()), e);
        }
        return currentScreenshotPath;
    }
```

The idea was to **not throw an immediate Exception whenever a difference in printscreens is encountered**, because for a single **@Test** we may want to take multiple printscreens after different actions that change the UI in the whole flow.

Therefore the method **takeScreenshotCompareAndCollectDiff** **just adds the possible record of an image diff to a List&lt;ScreenshotDiff&gt;** and does not throw an Exception.
We use _flagUpdateReferenceScreenshots_(which we get from maven properties or overriden in the command line) to decide if we need to update the reference screens or do the actual comparison of the new screenshots with the references.

we can invoke just a rebuild of the reference images by using the 
```
mvn -DflagUpdateReferenceScreenshots=true clean verify
```
or the actual screen comparison tests with
```
mvn clean verify
```
So we need a way to **signal a test failure if there are image diffs at the end of the test** to get a hint what screens differ.

### Using Junit "Rules" to take actions Before and After a test runs
We don't want to add boilerplate code to explicitly check at the end of every test if there were any differences between the printscreens. 

JUnit provides a way through **TestRules** to indicate that it needs some work done before and after a test's execution.


This is the class **ThrowErrorOnDiffScreensRule** that we injected into our test class through the **@Rule** annotation

```java
public class ThrowErrorOnDiffScreensRule extends TestWatcher {

    private List<ScreenshotDiff> differentScreen = Lists.newArrayList();

    @Override
    protected void starting(Description description) {
        differentScreen.clear();
    }

    @Override
    protected void finished(Description description) {
        if(! differentScreen.isEmpty()) {
            throw new ScreenshotDiffsException(differentScreen);
        }
    }

    public void addScreenDiff(ScreenshotDiff screenDif) {
        differentScreen.add(screenDif);
    }

}
```  

It's actually simpler to extend JUnit **TestWatcher** abstract class as a way to do something around @Before and @After methods of a test.

A link better explaining [JUnit TestRules](http://www.alexecollins.com/tutorial-junit-rule/) and why you should use them and their roles in setting up and cleaning up after a test. 

BTW you might also be interested this is a good way to have a printscreen taken whenever a Webdriver Test fails to see how the page looked when it failed, you can have a look [here](https://github.com/balamaci/pippo-demo-webdriver-test/blob/master/src/test/java/ro/fortsoft/pippo/demo/integration/rule/TakeScreenshotOnFailedTaskRule.java).



PS: We could actually have achieved the same by implementing the check for in the **@After** method of **BaseUIWebdriverTest**, but this way it's more decoupled I think and also gave us a chance to advertise the **TestWatcher** class.

#### PhantomJS as WebDriver implementation
So we programmed the tests to run with the generic WebDriver interface. However we need an "actual browser" implementation to "drive". I've already mentioned [here](http://balamaci.ro/continous-integration-with-jenkins-docker-ansible-webdriver/#webdriver) that I like the speed and easy install of the PhantomJs **headless browser**.

```
public class PhantomJsCrudNgDemoAppUITest extends CrudNgDemoAppTest {

    public PhantomJsCrudNgDemoAppUITest(Dimension dimension) {
        super(dimension);
    }

    @Override
    protected Browser initBrowser() {
        return new PhantomJsBrowser();
    }
}
```
While it is not included in the project the idea would be to have more than one browser tested. 
Right now I'm  we'd simply create some other classes like **FirefoxCrudNgDemoAppUITest**, **ChromeCrudNgDemoAppUITest** which will just all extend the same scenarios defined in  **CrudNgDemoAppTest**, and just have different code for Browser object initialization. 
We'll also probably need to add the name of the browser to the generated image files to know which is which, but all this seems pretty doable at least in theory.


Finally the **ImageUtil** class just exposes methods for comparing the two image files by hashing the content to determine if same or not 
**public static boolean isEqual(Path img1, Path img2)**

and the one that uses **im4java** to create the diff image.
**public static void createImageDiff(Path firstImage, Path secondImage, Path diffImage)**


#### Pippo web framework 
The web application we tested in the example is the AngularJS [CRUD demo app](https://github.com/decebals/pippo-demo/tree/master/pippo-demo-crudng) of the [Pippo web framework](http://www.pippo.ro/doc/getting-started.html). 

I very much like Pippo framework as it's **the most lightweight jee servlet3 web framework**, a perfect fit for the smallest VPS instance or even on RaspberryPi to have it running. I suggest you give it a look.

While **pippo is very lightweight**(~200kb) itself and being built from the start with the concept of modules you can tailor it to just fit your exact needs(different template engines, codahale metrics, etc), still the inclusion of the **webjars** for angularjs(which accounts for 60% of the size) and bootstrap and font-awesome makes up for almost all the rest of the .war file for the AngularJS CRUD demo.

#### Final thoughts
We just might have added another piece in the great puzzle of automatization of the build & testing process.
At the very least it's an idea for a way to generate all the printscreens of the application along test cases.

Along the way we also shared some info about **parametization of the Junit testcases** by injecting different screen resolutions.

We also touched a little on the ways to use **JUnit @Rule** to setup pretest conditions or cleanup after the test.

PS: In the next article [Docker containers for Visual Regression Testing setup](http://balamaci.ro/docker-container-for-visual-regression-testing/) I've revised the running process to package the dependencies inside a docker container. 
