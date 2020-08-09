+++
author = "Serban Balamaci"
categories = ["docker", "bdd", "cucumber", "serenity"]
date = 2015-11-02T10:08:00Z
description = ""
draft = false
slug = "orchestrating-a-reusable-bdd-web-testing-setup-part-ii"
tags = ["docker", "bdd", "cucumber", "serenity"]
title = "BDD web testing setup - Docker, Cucumber and Serenity - Part II"

+++


### Overview 
In [Part I](https://balamaci.ro/using-docker-and-docker-compose-for-orchestrating-a-full-bdd/) we looked at setting up a reusable testing setup by **thinking in high level components through the use of docker containers**. We added **selenium-grid** containers  and saw how **docker-compose** makes it easy to play around with more containers and still make it look simple and clear. This sets the blueprint for adding and swapping around other containers(like db containers, other web containers) that could help with setting up mocked service dependencies, test databases.

In this part we also talk about writing BDD tests in **Cucumber** and how it's great to pair it with **Serenity** for some great tests reports. 


### Crash course BDD
I'm not going to reiterate what BDD is in detail, as this has been better explained [here](http://thucydides.info/docs/serenity-staging/) which I encourage to read anyway.
In fact we're just going fast over and rather presenting some code of how it all fits together to get you started quickly.

#### There is a language mismatch between the stakeholders in a software project.
Usually when working on a software project there might be problems communicating between people involved(some which might use  to get to the same level of understanding to what the application should do exactly and how it should behave.
Jokingly or not there is this image where developers and business people speak totally different languages and it's very easy to misunderstand one another.

#### Ambiguity is a problem
I think we can all agree that it's dangerous to leave things to interpretations. We've all gone through the pain of having to rework parts of the application after we thought them finished, as it turned out it was something different what the client meant.

Having some user story that says **'Users should not be allowed to enter invalid phone numbers when creating a Contact'** might look ok but it's actually damn **vague**.
What does it mean for a phone number to be valid?
- Should there be a minimum/maximum number of digits?  
- Should there be only digits or is '+' at the beginning also valid?  
- Are international phone numbers allowed?  
- Are grouping spaces like '+49 234 46535465' ok?  

I leave to your interpretation what the simpler 'User should not be allowed' could mean.


#### Concrete examples help solve ambiguity.
One way to fight ambiguity is to give some **concrete examples** on how the system should behave.
To counter the above ambiguous scenario of *'Users should not be allowed to enter invalid phone'* a way to having it written better would be:
*When an user enters as a Contact's phone a number with less than 3 digits and tries to press 'Save' he should remain in the edit form and see a warning message about the length of the number*


One good way to express the behavior, is to structure the test case into a formal **'Given', 'When', 'Then'** setup.
**'Given'** to provide some context of the state the system is in, **'When'** to describe an action or event and **'Then'** to verify the outcome.

Let's see an example of how we could describe the "Editing a contact's phone feature":
Now we can better describe how exactly the application should behave as we write more and more scenarios:

**Scenario**: Contact phone cannot have less than 4 digits  
  **Given** I edit a Contact's Phone Number  
  **When** I enter a Phone Number '+493'  
  **Then** I should see 'Phone number too small' warning  

**Scenario**: Contact phone should only contain digits or whitespaces.  
  **Given** I edit a Contact's Phone Number  
  **When** I enter a Phone Number '+49a5657878'  
  **Then** I should see 'Invalid phone number(only +,digits or  spaces allowed)' warning.  

**Scenario**: Contact phone containing spaces is valid, and the spaces are removed.  
  **Given** I edit a Contact's Phone Number  
  **When** I enter a Phone Number '+49 7722 533 280'  
  **Then** I should see 'Saved contact successfull' message  
  **And** I should see the '+497722533280'  

Structuring the requirements under this for is better because:
  
  - There is a smaller chance for misunderstanding between all parties when developing the requirements also when testings them.
  - **Better overview of and regression testing reports**. By implementing automated tests for each step in the scenarios and by generating tests reports everyone can have the broad picture to know exactly what has been tested, what is working and what is not with any release.
  - **Living documentation** - instead of having separate documentation to how the system should behave we can use the scenarios with the concrete examples as documentation.
  It's called **living** since we have the scenario steps related to actual automated tests that are continuously being maintained and added and do not get out of sync from the application as people forget to also update the documentation.



### Cucumber and gherkin "language"
Cucumber is the framework that relates this natural language description to running actual automated testing code.

In Cucumber each test case is called a **'Scenario'**.
Multiple **Scenarios** can be grouped together into a **'Feature'** and stored into files with the *.feature* extension.
You write the **'.feature'** files with natural language in what they actually call **gherkin** language.

Every scenario consists of a list of **steps**, which must start with one of the keywords **Given, When, Then, But** or **And**

For ex:
**Feature**: Login functionality  
  Users should be able to login into the application.

 **Scenario**: Entering valid credentials I am able to login and see 'Contacts' page   
   **Given** I open the application  
   **And** I see the Login page  
   **When** I login with user 'admin' and with password 'admin'  
   **Then** I should see the 'Contacts' page  

  **Scenario**: Entering empty password triggers a warning message  
    **Given** I open the application  
    **And** I see the Login page  
    **When** I login with user 'admin' and with password ''  
    **Then** I should see the warning message with 'password-cannot-be-null' key  

Notice the words in bold, they are actual gherkin 'keywords'.  

Until now we've just covered the documentation part of the steps in the gherkin language. We now need to relate them to actual code so that whenever Cucumber parses a step in the gherkin text, code that performs the action gets executed. 

**The actual code associated with a Step is called the "Step definition".**

Using something similar to regular expression we can associate a gherkin Step with a Step Definition.

Let's look at an example to see how this would look like:

```
public class LoginStepDefs {

    @Given("^I open the application$")
    public void open_application() {
        throw new PendingException();
    }

    @When("^I login with user '(.*)' and with password '(.*)'$")
    public void login(String username, String password) {
        throw new PendingException();
    }
....
}
```

So we associate a java method through an annotation like **@Given, @When, @Then, @And, @But** with a Step in gherkin Scenario through the regex like expression.

We grouped the methods inside a single java class *LoginStepDefs* but we could organize them however we'd like(cucumber is looking for the method annotations), still it probably makes sense to have a separate class for each domain entity.
We just need to tell Cucumber the package where to look for the step definitions methods.

Talking about regex patterns, do not be confused by **^** and **$**. They only mark the beginning and the end as to match exactly with the gherkin text.
For example omitting the **$** at the end
```
 @Given("^I open the application")
 public void open_application() {
```
would make the java *open_application()* match also text like "I open the application under test".

We also have an example of a simple regular expression to indicate dynamic parameters that can be passed into the methods.
```
@When("^I login with user '(.*)' and with password '(.*)'$")
public void login(String username, String password)
```



### Serenity framework
Right now Cucumber helped with relating the business scenarios to actual generic code, but since we're doing testing ui testing for a web application we can pair it with **Serenity** which **provides a higher level wrapper for WebDriver to remove parts of the boilerplate associated with it**.

Apart from that **Serenity helps with providing great tests reports** that helps seeing the bigger picture of the status of each scenario and how they fit in the overall set of product requirements, what parts of the requirements have been covered by tests and if those tests have passed or not.

![](/assets/2015/serenity-aggregate-report.png)


  

But what I particularly like is the **possibility to have also printscreens taken for each step executed in the test reports**.
![](/assets/2015/printscreen_test_report.png)


### Page Objects with Serenity
The concept of **Page Objects** is a **Software Pattern**, not something limited to using Serenity. It goes to say that for doing actions on a certain page it's better to encapsulate the Webdriver workings inside a class, hide the actual the low level details of Webdriver interactions with the  components on the page, and instead expose higher level natural language concepts that the user could do on the page. 

For example in a **LoginPage** we could have components for a password text field and username. Instead of having methods like *public WebElement getPasswordTextfield()*, it makes more sense to just expose a *login(String username, String password)*  or *typeUsername()* or *typePassword()*.

The idea would be that should the components of the web page would change, we could limit the need for refactor just to the inside of the Page Object class.

Serenity provides it's own helper class **PageObject** and we should extend it. We get the benefits of automatic creation and injection of Webdriver inside the PageObject and also access to a range of helper methods: **find(By.name(..))**, **typeInto**, **clickOn**, **waitFor**, etc.

```
@DefaultUrl("http://localhost/login") 
public class LoginPage extends PageObject {

    public void enterUsernameAndPassword(String username, String password) {
        WebElement txtUsername = find(By.name(Locators.getValue("login.txtUsername.name")));
        WebElement txtPassword = find(By.name(Locators.getValue("login.txtPassword.name")));

        typeInto(txtUsername, username);
        typeInto(txtPassword, password);
    }
}
```
*@DefaultUrl("http://localhost/login")* is useful when doing loginPage.open() to get directly to that url. We'll see further on how this is passed dynamically through serenity.properties .


### Serenity Step class
While we looked at the fact that in Cucumber through **@Given, @When, @Then** we supply Step definition, Serenity also offers **ScenarioSteps** class for extension. 
This extra layer of indirection helps with keeping the "what" separate from the "how", it acts as an adapter from the business language in Cucumber to the more technical low level of implementation. Without it step definitions would tend to become rather technical, which limits reuse and makes them harder to understand and maintain. 
For ex. a Cucumber step would say "I login with 'user' and 'password'" => Serenity Steps: "User types 'admin' in 'username' textfield" and "User types '123456' in 'password'  field" and "User clicks 'Submit'"

```
public class LoginSteps extends ScenarioSteps {

    private LoginPage loginPage; //automatically instantiated

    @Step
    public void enterUsernameAndPassword(String username, String password) {
        loginPage.enterUsernameAndPassword(username, password);
    }

    @Step
    public void clickOnSubmit() {
        loginPage.clickOnSubmit();
    }
}
```

Each **@Step** annotation method that gets called inside the Cucumber step definition method, makes the method appear inside the test report under the Cucumber step.

The **LoginPage** object gets automatically instantiated and the Webdriver preconfigured instance injected inside. 

"Both **scenario step libraries** (annotated with the @Steps annotation) and **Page Objects** that are declared inside the Cucumber step definition classes will be automatically instantiated."
```
public class LoginStepDefs { //Cucumber Step definitions

    @Steps
    private LoginSteps loginSteps; //Serenity Steps

    //private LoginPage loginPage; - could have used the PageObject directly

    @When("^I login with user '(.*)' with password '(.*)'$")
    public void login_user(String username, String password) throws Exception {
        loginSteps.enterUsernameAndPassword(username, password);
        loginSteps.clickOnSubmit();
    }

    @Given("^I am logged in as admin$")
    public void login_admin_user() {
        loginSteps.openLoginPage();
        loginSteps.enterUsernameAndPassword("admin", "admin");
        loginSteps.clickOnSubmit();
    }
}
```

Notice we could have used the directly the **LoginPage** PageObject, and not have to bother with the extra layer of indirection in the Serenity **LoginSteps**, but that would have meant that we would not have those @Step method in the final Serenity test report along with the printscreens.

### The code
Let's now look at what you need to setup to start with your own Cucumber Serenity tests.
The source code can be downloaded from [here](https://github.com/balamaci/blog-ui-bdd-testing).

First of we'll need a special test runner **CucumberWithSerenity**

```
@RunWith(CucumberWithSerenity.class)
@CucumberOptions(
        format = { "pretty", "html:target/pippo", "json:target/cucumber.json" },
        features = {"src/main/resources/features/"},
        glue = "ro.fortsoft.pippo.demo.bdd.cucumber")
public class PippoCrudDemoRunner {
}
```
through which we specify where the *.feature* files are located and the package under which the Cucumber step definition classes are.  
There is also **serenity.properties** that allows customization of Webdriver, and the generated test reports.
```
webdriver.remote.driver = firefox

webdriver.base.url=http://tomcat:8080/pippo/
webdriver.remote.url = http://hub:4444/wd/hub

serenity.use.unique.browser = true
serenity.take.screenshots = BEFORE_AND_AFTER_EACH_STEP
```

Notice in the above props with url-s we reference the containers by the name we configured them in **docker-compose.yml**(**'tomcat'** container and **'hub'** container of the selenium-grid).

   - webdriver.remote.url = http://hub:4444/wd/hub - the address of the selenium grid hub
   - webdriver.base.url=http://tomcat:8080/pippo/ - this replaces the 'host' part in
    ```
    @DefaultUrl("http://localhost/login")
    public class LoginPage extends PageObject {
    ```
    we mentioned before, and which would mean 'http://tomcat:8080/pippo/login'

### Running the tests with maven
We're references the **CucumberWithSerenity** by including the package where it resides from the **maven-failsafe-plugin** which performs integration tests.

```
<plugin>
		<artifactId>maven-failsafe-plugin</artifactId>
		<version>2.18</version>
		<configuration>
				<includes>
						<include>**/runner/*.java</include>
					</includes>
					<reuseForks>true</reuseForks>
		</configuration>
		<executions>
			<execution>
				<goals>
					<goal>integration-test</goal>
					<goal>verify</goal>
				</goals>
			</execution>
		</executions>
</plugin>
```

### The application to run the ui tests against
We're using as example in our BDD testing setup the demo app [pippo-crud-ng](https://github.com/decebals/pippo-demo/tree/master/pippo-demo-crudng) of the [pippo](http://www.pippo.ro/) web framework.
I always take opportunity to present other interesting frameworks and such is **[pippo](https://github.com/decebals/pippo) java web framework** which I like as it's very lightweight and build with the concept of modularity and the small footprint makes it ideal for IoT applications. If you have not already, please take a look and maybe consider it as a possible **alternative to other frameworks like spark, dropwizzard, ninja**.

I also have pushed **pippo-demo.war** in the blog repo just to make it easier for the reader to run the whole setup.
```
git clone git@github.com:balamaci/blog-ui-bdd-testing.git

./build.sh

./start.sh
```

the Serenity report gets generated and can be found and opened at **./target/site/serenity/index.html**

This is the result 
[![BDD preview](/assets/2015/bdd_preview.png)](https://balamaci.ro/static/serenity/index.html)

### Extra tips and tricks

#### Cucumber tags
Sometimes there may be long running tests we might want to just run once per night not with every build. Or we'd like to separate ui tests from batch tests. This is where **cucumber tags** come into play: we can add one or more **@tagname** at each Scenario or Feature in the .feature files to mark them.

For example:
```
@ui
Feature: Login functionality

  @security
  Scenario: Entering valid credentials I am able to login and see 'Contacts' page
    When I login ...
```

We can tell then Serenity to just run scenarios with a certain tag(s)
```
mvn clean verify -Dcucumber.options="--tags @long-running,security"
```

#### Cucumber hooks
Cucumber hooks help with pre-setting test conditions.

For example we usually want to make sure to have a clean state when firing up the browser to clear any set cookies - like user's session-.

We can do this through the **@Before** or **@After** annotations from Cucumber. These annotations can also take a tags as parameters: **@Before("@security, @ui")**

```
public class CucumberHooks {

    @ManagedPages
    private Pages pages;

    @Before //@Before("@ui")
    public void openBrowser() {
        pages.getDriver().manage().deleteAllCookies();
        pages.getDriver().manage().window().maximize();
/*
        pages.getConfiguration().getEnvironmentVariables()
            .setProperty(ThucydidesSystemProperty.WEBDRIVER_BASE_URL.getPropertyName(), "http://tomcat:8080/pippo/");
*/
    }
```


### Conclusion
**Cucumber combined with Serenity are two great tools** that go well hand in hand. I just got to say I love the test reports Serenity generates, especially the print screens seem very useful.
They can be used in any Java project. It may not bear mentioning but **they are not dependent in anyway to run with docker**, it was just an example to do some complex setup, you are free to start using them right now in your projects.

Through this blog post I hope to have made you aware and curios to read further on about them. 

Thank you!

### References
   1. [Serenity Documentation](http://thucydides.info/docs/serenity-staging/) - good general BDD information
   2. [Cucumber for Java Book](https://pragprog.com/book/srjcuc/the-cucumber-for-java-book)
   3. [Cucumber Docs](https://cucumber.io/docs/reference)  
   4. [Docker - getting started](https://dzone.com/refcardz/getting-started-with-docker-1)
