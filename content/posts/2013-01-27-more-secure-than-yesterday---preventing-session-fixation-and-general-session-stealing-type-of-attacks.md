+++
author = "Serban Balamaci"
categories = ["security"]
date = 2013-01-27T18:11:00Z
description = ""
draft = false
slug = "more-secure-than-yesterday---preventing-session-fixation-and-general-session-stealing-type-of-attacks"
tags = ["security"]
title = "More Secure than Yesterday – Preventing \"Session stealing\" type of attacks."

+++


### Explaining Session Fixation attack
The session fixation type of attack is very simple to understand and relies on the fact that the attacker convinces a victim to use a session already fixated(provided) by the attacker to your site. 

Most usecases involves the fact that the attacker would pass an unauthenticated session to the victim which authenticates to your site using his valid credentials. Since the attacker provided the session, he has access to the victim's session which will be an authenticated one and thus all the information and actions available to the user will also be available to him. 

Well you may ask "How exactly this can be done?", so here is a valid scenario:

 - <b>The attacker obtains a session on your site. </b>
Some sites create a session even before the user logs in or after a failed login attempt. For example by default Wicket web framework intercepts an unauthenticated user's call to a protected page and redirects the user to a Login page but with a session created in which it stores the original page request and parameters the user made. After succesful authentication the user is redirected to that original page he tried to access -which is available in the web session-. (This is not necessarily a bad thing once you are aware how you can protect from session fixation attacks).
The attacker will see a session was created and keep the session's id for example JSESSIONID=1234.
 - <b>The attacker tricks a user to access the site with the session id provided by him </b> http://www.yoursite.com/home;jsessionid=1234. For example he can send him an email where he appends the sessionid in the url. Even if the user checks that it's your site legit's url he is clicking and not a phishing site, he might not consider anything wrong with the presence of a parameter in the url that he may think to be a security token for even enhanced security.
 - The attacker checks periodically(and thus even keeping alive the session the sent in the email) that the victim took the bait and authenticated in the site. 

## Means of Defense
Fortunately the way to protecting one's self is easy in this case and usually it's done by
### Regenerate the Session after Login
Meaning that after a succesful user login, if there is an existing session, that session is invalidated and a new session is created. 
<ol>
<li>User enters his credentials and they are validated. If passed continue to step 2.</li>
<li>Some information can be retained from the old session and since we're going to destroy the old session keep it temporarely. (Remember that for example <b>Wicket</b> stores in the session the initial user request). </li>
<li>Destroy the old session( call <i>HttpSession#invalidate()</i> method)</li>
<li>Create a new session with a new session id.</li>
<li>Populate the new session with the data temporarily held.</li>
<li>Continue with the normal flow after authentication like directing the user to the home page or his initial requested page.</li>
</ol>
This solves the Session fixation problem because <b>even if the user came to your site with a preset(fixed) session, after he authenticates, that old session will be invalid and only the user will receive the new session's id that the attacker will not know</b>. 

In <b>Wicket</b> btw there is actually a <b>Session#replaceSession()</b> method which basically does the invalidation and recreation of the web session and was created exactly for solving the session fixation scenario.

### Additional useful counter measures for Session based attacks
These are some additional general session related good practices you might want to consider that would not only make a session fixation attack harder, but are general good thing to have to protect against session stealing/hijacking. 
Here they are:

#### 1. Store Session IDs only in cookies instead DO NOT pass them in URLs
If the session id is passed around in the URL it could be saved in a number of locations including the browser history, proxy server logs, referrer logs, web logs, etc. This of course make the application more vulnerable to session stealing/hijacking attacks. Users might post the urls containing their sessionid or message them to "friends".

Instead by storing the JSESSIONID in a cookie (and also good practice is to set the Secure flag to it), the session will not appear in the above mentioned logs neither would it be possible to set it through the fake email.
The <b>Servlet 3.0</b> specifications let you configure this in the <b>web.xml</b>

```
<session-config>   
   <tracking-mode>COOKIE</tracking-mode>
</session-config> 
```



#### 2. Accept only Session IDs from cookies and not in URL

By calling <b>HttpServletRequest#isRequestedSessionIdFromURL()</b> or <b>HttpServletRequest#isRequestedSessionIdFromCookie()</b> you can establish is a session in a request was passed from the url or not. This would also mean that it would have been impossible for a session fixation attack to have come from the user clicking on an url in an email for example.


#### 3. Consider not allowing <b>same Session id</b> with requests coming from different IPs or with <b> different User Agent</b>
Normally it doesn't make sense other than the case when an attacker hijacked another user's session for requests with the same session id to come from a different IP than the one that created it. 

There are however some cases where this could happen for example: Some users behind proxies might be directed to different gateways and so will appear with different source IPs, although it's not that common and we can even reduce this possibility by taking into consideration only the first two parts of an IPV4 address when looking for different IPs.

Well this would not be the case when looking that requests have the same <b>User-Agent</b>. Having requests with a different User Agent for the same session is highly suspicious. However for an attacker faking an <b>User Agent</b> is trivial, but the attacker would first need to know the victim's browser, not a hard thing to do, but still something which lowers the success rate of such an attack. 
Even to this there is an exception that legitimate users can for example switch their IE browser to <i>Compatibility view</i> which actually send a different User-Agent also. But think that probably the users will not have a problem with having to relogin after they perform such action.

The best we can do is to look for suspicious signs like the ones mentioned above and get the user to re-authenticate by destroying the session if you suspect at any point they're not the same user. 

#### 4. Don't set long session timeouts, keep track or invalidate session that were alive for a suspicious long time
For better user experience some people would want you to set your session with long spans of time until expiration thus assuring that the user is not bothered with having to relogin even if he forgot about your site in a browser tab for a few hours. 
Having a long time before session expiration is bad because you give a larger time frame for an attacker to act in. 

A session can be kept from expiring by an attacker by regularly initiating requests to the server. So even if you do set a session timeout of 30 mins, the lifetime of the session will be extended by 30 min with every call and doing regular calls one can extend it practically indefinitely. 

By using also at the creation time of the <b>HttpSession#getCreationTime()</b> you can find out the session that were kept alive for a long time. Destroying those session and forcing a relogin should be a good idea. You can easily implement this behavior by using a Servlet Filter for example.

#### 5. Set <b>HttpOnly</b> attribute on the JSESSIONID Cookie
If a <b>HttpOnly</b> flag (optional) is included in the HTTP response header that creates the cookie, that cookie cannot be accessed through client side script (most all today's browsers support <a href="http://www.browserscope.org/?category=security" title="Browsers that support HttpOnly">this</a>) but instead they are only to be available to the server, any attempt to access them from Javascript through <i>document.cookie</i> will not return anything. Since we advised to store the Session ID in a cookie it's a good candidate for setting the <b>HttpOnly</b> flag because most <b>XSS attacks</b> target the SESSIONID cookie which they send back to the attacker. As you know, once an attacker will hijack the session he has the keys to the kingdom(see above for lowering success rate of this by checking IP and User-Agent).

Again the <b>Servlet 3.0</b> permits setting this in the <b>web.xml</b>:
```
<session-config>
 <cookie-config>
  <http-only>true</http-only>
 </cookie-config>
<session-config>
```

So this is a very easy quick win.
One should also notice that the <b>HttpOnly</b> is available to be set for any cookie, not just the JSESSIONID, like for ex.:
```java
response.setHeader("SET-COOKIE", "TOKEN=" + userToken + "; HttpOnly");
```
