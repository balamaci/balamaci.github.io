+++
author = "Serban Balamaci"
date = 2012-03-26T13:12:00Z
description = ""
draft = false
slug = "more-secure-than-yesterday-trying-to-prevent-brute-force-attacks"
title = "More Secure than Yesterday - Trying to prevent Brute-force attacks"

+++


#### Boring intro

I'm working in my spare time developing a little Wicket Shopping platform and these days I've been pondering how much of security considerations I've be letting on for later, such as the topic of today's post of preventing the Brute force attacks for obtaining a login in our web application.

We're all pushing for adding features to differentiate us from other products and security issues beyond user authentication is not making it in our next releases. You cannot sell a product on being the most secure, the customer takes that for granted, or at least there is no simple way to prove that your competitor is not doing it poorly. You've only have to fear making the news headlights but then it already means you are a name in the business.

So I've had some brief talks to fellow developers to pick their brains on how they do it - protecting against brute-force attacks on their sites-, but it surprised me how many of them shrug away the question on what is their mechanism of defense. But then again maybe they don't want to divulge any vulnerability of their system ;)

So this got me thinking that probably there are a large number of sites susceptible to brute-force attacks running without problems probably because of the fact that it's not really worth the time of any interested "hackers" to obtain logins for these web apps. The most damage they can do is they'll obtain some data(customer IDs and email, change some passwords), and place fake orders on the customer's behalf. Disastrous PR for Sonny and other companies, not really a problem for some of my friends with various eshops and blogs.

#### The problem with Brute Force attacks

Apart from the fact that as I've seen it, a lot of programmers only concerned themselves with providing the authentication and do not think beyond this, to consider the next step like the possibility of a brute-force attack.

Next problem is most fellow programmers wrongly think themselves safe because:
"We log ALL failed login attempts... "
Where is the safety coming from? Logging without doing any monitoring and analysis it's at most a way to know what accounts might have been compromised AFTER you become aware that some accounts were compromised. Ideally you want to prevent them from gaining access in the first place.

So there the problem we need to handle can be divided in two parts:

**Monitoring** and knowing that an ongoing brute-force attack is happening and 

 **Reacting to prevent** the attacker from succeeding.

#### Monitoring and Logging

This is actually the easiest part. To put it in Frank Herbert own words of wisdom : "The first step in avoiding a trap is knowing of it's existence".
Bottom line is we need a monitoring system that triggers our defenses when a suspect high number of failed login attempts originate from the same IP in a (generally small)fixed amount of time. Surely the definition of "suspect" leaves a lot to interpretation. One attacker might fear the existence of a monitoring system in place and think to fool it by mimicking a human behavior and try for ex. only 5 passwords per minute.

The attack can come from a single location or a distributed one if the attacker is using some proxies(a few infected computers in a botnet). I think at least for simplicity we're just going to assume we see many requests made from the same IP address.

### Reacting and defending

Now that we know we are the victims of a in progress brute force attack the biggest issue is what to do about it?

There are very interesting stories about how easy is to shoot yourself in the leg with this and the fact it's not an universal right way of handling it, we only have some means of defending, the reader must think and choose which is the right one to implement for your business scenario.

#### Blocking the attacker IP address temporarily or permanently

This should be the most effective way of dealing with the attacker. Blacklisting attacker's IP with a firewall rule at OS level, at web server or Application level. While having the ADVANTAGE of protecting your server resources by just ignoring the requests it also has a big DISADVANTAGE in the fact that you may block many of your legitimate users, consider the employees of a company or students of a campus that all access the internet under the same IP.
How it can be done for OS level for example Fail2ban is a Python solution that scans log files with regex expressions and blocks IPs by adding a rule in IPTABLES.

#### Block the attacker used username, not the attacker's IP

Usually an attacker is trying to guess the password knowing a valid system username.
We saw where the above solution of blocking the IP goes wrong by being too generic and so instead we block the User to which the attacker is trying to gain the password. We've seen this implemented in many systems the dreaded Account locked message for the legitimate user. But there's a funny story about how this turned out as a terrible idea for a known site.
You know eBay and it's bidding system? EBay had this lock user on too many failed attempts implemented and when someone bid highest, their username was shown next to the bid. Well you would take that username and try to login a few times so that user would get locked. Now the real user is locked out until he calls the Support Department to fix it and in the meantime he cannot bid against you. While the solution to this one is simple and something I'd like recommend as
Simple rule of security: Do NOT display the username or email of a user in a public area of your site, nor make it easy to obtain usernames.
EBay could have separated the user's public name from his login name and at least greatly lower the number of attackers thinking of exploiting this.

So the downside may be that you need that call to an admin that manually resets the user's status and most of the times anyway turns out it's just the user mistyping his password several times than a successful attack prevention. Ok, you may have an automated system to which the user can request a password reset, but you get the point it involves some hassle from the legitimate user.
Then again if you have a sensible threshold like for example an user typing 15 passwords per minute either that is an automated script or the user really forgot his password and needs to reset it anyway.

#### Using Captcha

Captcha is a way to check that your user is human by asking a question that supposedly only a human can answer. Thus Captcha is not only being used for authentication instead was adopted also as an additional validation, usually good for spam fighting for example when offering a Comments section you don't want automated crawlers posting garbage ads all over your site.

The Downside is:
Captcha libraries can NOT be fully trusted anymore. Making them safer means making it harder for human users also to distinguish the text in the image. Remember how many times you had to refresh the captcha image until you got something you can really read, well that's because the OCR software of the attackers are also keeping pace with the latest technology.

The library I use and recommend is ReCaptcha.
With ReCaptcha you're helping also to digitize books, you know? How it works:
ReCaptcha is a nice idea in the way that it's also an OCR software being used to digitize old paper books and newspapers. Sometimes the recognition software encounters a word it cannot make out, and so it puts it into a failed OCR attempts database and offers it instead for validation to users. The more people give the same answer to the image, the more confidence it has the image means the word most users give for an answer.
But the question that arises is how can the software validate the input from the users if it doesn't know the answer in the first place?
And the answer is that actually ReCaptcha send two images to the user for identification. For one of the two, the software knows the answer and if the user can guess that one, it assumes the answer for the other image is also correct and if more and more user answers with the same word well... "you GET the picture/image" :).

#### Internet access only
Since the challenge images come from ReCaptcha site and are not in an available database you can host or a service you can deploy, it means your users and application need to have access to the Internet, and it's not suited for closed applications with no Internet access. Since it's an external service not directly under your control theoretically it may fail. But hey, if it's availability is good enough for the likes of Google or Facebook I've got a feeling it meets your demands also.

Using timeouts between Login attempts

This works by trying to address the fact that the attacker has a lot of password combinations to try in a limited amount of time. In theory, if we start to add a waiting time between the failed attempts we greatly limit the number of tries the attacker can make per hour, day etc.
However such a simple solution of-course it has also quite a few Downsides:

Paralel requests - The attacker can initiate a few hundred(or whatever many the server can hold) simultaneous login attempts before you catch on and begin to insert your timeout
Waiting thread is still a thread, beware of helping an attacker to cause a DoS - even if you are doing just sleep() after password failed, having many such threads still alive it may mean you run out of available threads in the web server for servicing legit users
Sensible timeouts might not be enough - even a sensible timeout of 10 secs it still means that for a 100 simultaneous request it may mean the attacker could find a password
Reacting attacker - an attacker might figure out he's in a waiting timeout for password failed and terminate the connection immediately without waiting your imposed 10 seconds before starting again

#### Conclusion

Well the conclusion is that as seen there is no fit-all solution and the best option to go for might be rather a combination, an escalation of security as the pattern of events points with more confidence that a real attack is going on.
For example: After 3 failed logins maybe present the user with an additional captcha field, if he continues to fail authentication go for disabling the user and temporarily banning the IP if it turns into a full DoS attack.

Hoping I at least planted a seed of "insecurity" in the back of your mind and maybe have another look on what you have in place of defending against brute-force attacks, I await your comments and maybe get a little feedback to how you're implementing security.

Thanks for reading and good luck!
