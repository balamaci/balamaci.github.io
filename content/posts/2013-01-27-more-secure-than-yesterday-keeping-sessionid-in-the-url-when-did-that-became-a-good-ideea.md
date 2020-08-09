+++
author = "Serban Balamaci"
date = 2013-01-27T18:56:00Z
description = ""
draft = false
slug = "more-secure-than-yesterday-keeping-sessionid-in-the-url-when-did-that-became-a-good-ideea"
title = "More Secure than Yesterday - Keeping sessionid in the url - When did that became a Good ideea?"

+++


Now you may have heard that <b>keeping the session id in the url is bad practice</b>. Might even have read it in the <a href="https://balamaci.wordpress.com/2012/05/08/preventing-session-fixation-type-of-attacks/" target="_blank">previous post</a> and I'm just going to reassure you it still holds true as long as you don't take other precautions. 
<h3>A short full Recapitulation</h3>
Http Protocol is Stateless - Meaning you need to pass some sort of additional info into each request if you want the server to know that those requests are related(part of the same conversation if you want). This is where the sessionId comes into play most commonly used after the user logs in an used to identify requests corresponding to the same user. So by now you understand that you have big responsibility to not let this information fall into the wrong hands.

There are 2 most common possibilities for keeping the Session Ids:
<ul>
 <li>Either directly added into the URL</li>
 <li>Through a Cookie - since cookies are passed along with every request to the server-</li>
</ul>
<ul>


However the first approach of <b>passing it into the URL has some bad reputation</b> because:
 - Session ids in the url might be passed by mistake through copy-paste to others. Without knowing the user has exposed his account and privacy to that person.
 - Session ids may be leaked to 3rd party application through the <b>Referer</b> header or end up in server logs, Proxy logs
 - URLs remain in the browser history
 - Session ids passed through URL instead can be read by an [XSS](https://owasp.org/www-community/attacks/xss/) vulnerability (the URL on the other hand is readable through JS while an Http-Only cookie should be inaccessible to JS).

But <b>using JSESSIONID as a HttpOnly Cookie</b> also has a </b>big usability downside</b> that if the user opens another tab with you application, <b>all the tabs share the same session now since you just have one JSESSIONID Cookie that is common to all the tabs opening you application site</b>.

Depending on the nature of your application that might be ok, but you may also want to keep them independent. 

Storing the sessionid in a Cookie also had another downside in that <b>without additional protection leaves you open to a possible [CSRF attack](https://owasp.org/www-community/attacks/csrf)</b>.


### Now think of the fact that the user wants to be logged into your application with different users in different tabs.

<b>IDEAL: We'd want the user be able to use multiple sessionids into different tabs</b>
Meaning we'd want the "jsessionid" in the url but we also need to make extra sure there is also something else required that prevents others who have the user's browser url from just using it to open his account. What is a low level of security is to have an additional check that the User-Agent value matches. The User-Agent as is pretty low probability of two users having the exact User-Agent.

### Poor Solution
Keep the User-Agent on the serverside on the session and recheck it's the same one stored. This should be ok itself but still remains the fact that through an XSS vulnerability in your site the User-Agent can be read through JS and passed to a malicious user along with the sessionid so they can fake it. We need to do better:

### Better Solution - Take from both worlds!
Use the jsessionid in the URL <b>+</b>
1. Set a <b>HttpOnly</b> Cookie = <b>Hash(UserAgent)</b>. Actually do <b>=Hash(SALT + UserAgent)</b> to prevent smart attackers from figuring what the algorithm is used to get the cookie value and reproducing that value from the known UserAgent.
2. On the server check that  <b>Hash(SALT + Session.getUserAgent()) = TokenCookie.getValue()</b>

This means no longer can a 3rd party use just the url, also we've taken care of the XSS attack as even if an attacker gains knowledge of the User-Agent + sessionId it cannot reproduce the value of HttpOnly Cookie neither can he read it.
Bonus: The dynamic jsessionid in the URL acts as a token which also <b>nullifies any possible CSRF attack</b>(actually just by not using the cookie for the session solved that issue).

Till next time hope this info helps make the web a little more secure.
Thank you for reading.
