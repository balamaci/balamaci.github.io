+++
author = "Serban Balamaci"
categories = ["Wicket"]
date = 2011-11-02T11:40:00Z
description = "Apache Wicket is a great stateful component based framework which cultivates a sound architecture that appeals to Java developers"
draft = false
slug = "why-choose-apache-wicket-framework"
tags = ["Wicket"]
title = "Why I'd choose Apache Wicket as a web application framework"

+++


In a short description <a href="http://wicket.apache.org/" title="Apache Wicket">Apache Wicket</a> is a great *stateful component based framework* which cultivates a sound architecture that appeals to Java developers. This is just going to be a general summary of the points of why you might want to have a look at the framework. 
Let's begin:

#### Clean separation of Java code from HTML
One of the best and most talked about Wicket feature is of course the clear separation of HTML from Java code.
<b>No more Java code inside HTML</b>(yes that's you JSP). In Wicket we have pages composed of panels with components and for every page/panel we get two files: a <b>.java</b> file and one <b>.html</b> file which sit next to each other in a java package structure(this could be changed and have separate locations for html and java files) and both get packed into the final war.
Java components are matched to html by adding a wicket:id="javaComponentName" attribute for a html tag.

All the logic resides in Java so this makes the code compile time safe, also it means that the code can be debugged and unit tests created for it.

#### No custom tags other than HTML
<b>No custom tags</b> other than plain HTML are used nor any templating syntax to remember. HTML and CSS is the web language, we don't need to learn more vendor specific tags thank you. <b>No XML files</b> used for configuration either.

#### Share the work with a frontend developer
Both separated logic and no custom tags lead to a <i>.html</i> file which is now fully standards compliant and you can delegate the work of the UI layout to a specialised front-end developer instead of doing it yourself and are pretty safe of not getting on each other toes. Being standard html the frontend developer can use his favourite tool to edit the file.

All in all that means it is faster to get from PSD files => HTML => Page => Component.

### Easy refactoring
Sometimes the requirements and business specifications for a project change or we'd like to improve on our code as we advance in the project, so refactoring the code should be easy in a web framework(for many it is not). Due to the nature of the components as Java objects it means that refactoring is not very different from what you would do for a non-web Java application. This means, you as a developer have a low response time to changes of the requirements.

#### Component based framework with good concepts that appeals to developers
But the best thing for me is that Wicket really feels like a true Object Oriented web framework. All your components, pages, panels are Java objects(with a .html). Every web component extends a base <b>Component</b> class and can in turn be extended. Another way of creating components in Wicket is through composition of other web components inside a <b>Panel</b>. You can pack your components inside a jar file and bring them into another project and ideally a common library of components in your organisation might emerge. A place with generic open source components is the <a href="https://github.com/wicketstuff/core" title="Wicket Stuff project">Wicket Stuff project</a> or the <a href="http://code.google.com/p/visural-wicket/" title="Visural Wicket project" target="_blank">Visural Wicket project</a>

Wicket also comes with built in <i>Validators</i> and <i>Converters</i> or interfaces to implement your own.

Wicket cultivates a nice architectural pattern by imposing that every Component be backed up by a <b>Model</b>. It is the Model for the Component which <b>handles the data retrieval</b>. Same component can be used with different Model implementations that provide the data from different sources or ways. 
Think also of building and reusing generic Models like one that knows how to retrieve say through  Spring-Data an entity by ID.

Wicket provides a series of predefined Models, and a good understanding of what Models are and how/when each should be used is a very important thing to pay special attention when learning Wicket.

<b>Behaviours</b> are another concept that has been separated from Component. The reason for this is that even how a component "behaves" may be similar to another component's behaviour and such can be a candidate for reuse or extension. Ajax is an example of being one such cross component behaviour.

In Wicket you can also use <b>Visitors</b>(based on the <i>Visitor pattern</i>) for transversal of components in a Page.

Wicket from version 1.5 provides an <b>EventBus</b> mechanism through which components can communicate.

Because Wicket has been designed from the beginning as being <b>stateful</b> you can pass around Java objects(a good habit to get into is to actually pass Models) from one Page instance to another. Apart from the clearer intent and easier way to follow which pages use a particular Model, this may also be seen as an inherit security bonus. For example a user cannot force access to a Checkout page with url parameters but needs to create the Model object by passing through all the steps of the flow until checkout. Also that state will be tied to a particular page instance and you do not have to explicitly handle cleanup.

#### Good integration with Spring
Worth mention is the good integration with Dependency Injection frameworks like Spring/Guice. You can easily inject Spring beans inside any of your pages,panels,components,models by only providing the <b>@SpringBean</b> annotation.
Think of an example of a DetachableModel in which you injected a GeneralDAO that looks up and loads an entity by class and id, and you have an example of a reusable model component that knows how to populate itself when needed. Extend the idea to a html table repeater and you have a component's model to which you only have to pass the entity class and provides you data retrieval with pagination. (As a side note, I work with <a href="http://code.google.com/p/hibernate-generic-dao/" title="hibernate-generic-dao">hibernate-generic-dao</a> to build a GenericTableModel on this idea that also takes a <a href="http://code.google.com/p/hibernate-generic-dao/wiki/Search" title="Filter">Filter</a> object as parameter and reuse it in all my projects as it covers most all of my tables usage).

### Your Session is a POJO
In Wicket, you do not directly access HttpSession, instead you create your own Plain Java Bean Object as a WebSession. This helps with refactoring, because now instead of the Map structure of HttpSession, you have a Java object who's properties and methods you can track to where they are used in your project. But it is good practice to use the Session to store only global application information like user context, and use Models for storing the business logic data.

#### About Wicket and JS
Even from the beginning Wicket had Ajax capabilities built-in, that could be called from Java. This is done by simply adding the Java component to an ajax context object, the <b>AjaxRequestTarget</b>. Of course you can invoke any JavaScript code before or after the Ajax call.

Wicket does not generate any JS files from Java, unlike GWT frameworks and tries to keep things simple and clean so that it feels like you are in control of what HTML and JS gets generated by the framework. 

#### WicketTester enables and makes writing Unit Tests easy
Again Wicket developers kept testability in mind when they created Wicket and created the WicketTester class that exposes a rich API, for easy writing Unit Tests for Pages and components. Using WicketTester you can check that pages and components render without errors, forms submit and page navigation and test ajax behaviours without problems in a headless environment(non GUI).
I like also that you do not modify your application Pages and components in any way to integrates Spring it allows and that I can test the full flow of the application without doing any special modifications.

#### Security
Wicket offers support for component level security. Entire panels or components can be automatically be shown or hidden if the user has a certain security level. Of course it's still easy to provide the standard page level authorisation model most projects require. 
Wicket can also be easily integrated with <b>Spring Security</b>.

#### Internalisation support.
Wicket has good internationalisation support by using property files, or can be customised to handle any situation like for example strings loaded from the database.

#### Wicket is free and open source.
No need to say more.

#### Does not impose a Single Page Application architecture.
You could build a single page application in Wicket with no problems by having your panels replaced by ajax but you are not forced to do it. Other framework either impose it, or make it harder to have you application multi page. (SPA usually have issues with SEO)

<h3>Wicket has an active community</h3>
Bugs and questions are promptly responded through the <a href="http://apache-wicket.1842946.n4.nabble.com/" title="Wicket mailing list" target="_blank">Wicket mailing list</a>. 
Wicket is being actively developed with the 7.0 version very soon.

There are many Wicket tutorials and blog entries to get you started, best of being or you could check the source code for the <a href="http://wicketstuff.org/wicket/index.html" title="Wicket examples" target="_blank"></a> or the <a href="http://code.google.com/p/visural-wicket/" title="Visural Wicket project" target="_blank"></a>, and at least two books worth mention: <a href="http://wicketinaction.com/" title="Wicket in Action" target="_blank">Wicket in Action</a> and <a href="http://www.packtpub.com/apache-wicket-cookbook/book" title="Wicket Cookbook">Wicket Cookbook</a> for the more advanced.


## Framework minus points
To be fair I have to say something about what I think are the current weakness of the framework:
1. Building a stateless application requires extra work and knowledge.
Remember that Wicket has been developed for stateful applications. But there are a number of reasons why you'd want to opt for a stateless application.
Because if your application is stateless it means that requests can be processed by any of the web servers in your cluster, because all the parameters a page needs to reproduce a certain state are provided in the url. In a stateful application you either have to turn on session replication for the cluster(and infer a performance penalty on memory and network usage to keep sessions replicated on all machines) or more simply turn on sticky sessions to make sure that subsequent requests land on the same server where the user's session was created and contains the data.

Another reason could be that for a stateful application a http session must be created, which is kept in the url as the parameter "jsessionid" when indexed by <i>search engines</i>, and this might not look nice for a user seeing your page url returned as a search result.

But if you insist of keeping your application stateless is not really that hard though, and it may seem hard because you will be doing things that Wicket was doing for you and took for granted, and practically you'll just have to face the same thing that developers using stateless frameworks do everyday. 
The community provides support on this topic and replacement stateless components have been developed.

2. Because currently Wicket has it's own JS Ajax engine, when changing the DOM structure through Wicket Ajax(replacing a panel for ex), your JQuery selectors have to run again on the new DOM structure. This can easily be bypassed by packaging your selectors in a JS method and call it after doing the ajax operation.
As stated above, using JQuery for doing Wicket ajax is on the roadmap, and this little inconvenience may dissapear on it's own.

3. With Wicket, the session size tends to get big if you do not know how to detach your models. So this might hurt you in a way that it's not evident from the start but when your userbase grows. Having to know the framework I suppose it does go without saying and it's a must for just any framework, but I just wanted to reiterate that I advise you as a new wicketer to take some time to really understand Models as they are an important concept in Wicket, especially <i>DetachableModels</i>.

4. It has a higher learning curve because it's harder to find good object oriented programmers. Some people even learn to develop more Object Oriented and clean because of Wicket. Even if you are a Java programmer but come from a procedural way of programming JSP, Struts, Stripes you might get frustrated of having to instantiate components with models instead of the quick&amp;dirty way of just throwing in some code in html. Don't get me wrong, there are situations when the quick and dirty way might be the way to go, as stated for conclusions, Wicket like any framework should be used where it brings value, not as the proverbial hammer with everything looking as nails.

5. Performance testing done with tools which do not simulate real clicking on web components, but rather build URLs with parameters, much like JMeter does, requires an analysis of how Wicket generates the urls and how to match them, by the people doing the performance testing(because Wicket is stateful a page version parameter is encoded in the url, and also study the url on how to target specific Ajax <i>Behaviours</i> in the page).

As the closing statement I think and let me point out that I'm also a true believer of using the right tool for the right job and that in some cases Wicket might prove overkill. For example if I'd want a read-only site like say a blog, then I'd probably go for a light PHP CMS solution like Wordpress(or Ghost since I've imported this post recently into it).

Or if you have a desktop like application and you require out of the box heavy JS components that look good and don't need customisation then maybe I'd use [Vaadin](http://vaadin.com). Whatever the case I'd say if you are looking for a component based and you like you should at least give Wicket a try and so to set you on the right path just use the following Maven Wicket Quickstart archetype to generate and run yourself a very simple Wicket project:

```
mvn archetype:generate -DarchetypeGroupId=org.apache.wicket -DarchetypeArtifactId=wicket-archetype-quickstart -DarchetypeVersion=1.5-SNAPSHOT -DgroupId=com.mycompany -DartifactId=myproject -DarchetypeRepository=https://repository.apache.org/content/repositories/snapshots/ -DinteractiveMode=false
```
