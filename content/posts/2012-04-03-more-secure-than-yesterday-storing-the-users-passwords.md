+++
author = "Serban Balamaci"
date = 2012-04-03T21:00:00Z
description = ""
draft = false
slug = "more-secure-than-yesterday-storing-the-users-passwords"
title = "More Secure than Yesterday - Storing the Users Passwords"

+++


Today we're tackling the common problem of keeping our users passwords secure. It's going to be a recap of the best practices and reasons behind them, and I'll be introducing a very useful library for cryptography in Java to have a concrete implementation of how you can store your user's passwords in a safe way.

### About algorithms and practices
#### You should not use plaintext passwords - this is the wrong way to do it and you have no excuse for still using it. It's bad for the obvious reasons:

-	**Data theft** - sure you make think you have in place that would prevent an outsider to but on the other hand a lot of people use the same password for their email, twitter, facebook account, etc. The user entrusts you with his data, so are you REALLY, REALLY 100% confident there is absolutely NO WAY an attacker could get access to the password field in your database? No sql injection in your code, no backdoor to your server, no bug in the database.

-	**Trusting people** - in most of the cases you are not the sole person with access to the database. Even if you are the administrator, save yourself from the temptation of trying your users passwords on their provided emails.


#### What to do instead?
Well there are two way to look at the cryptography methods. There is **Two Way** (**Reversible**) encryption algorithms which means once you encrypt a text you can also decrypt it later, and **One Way** (**NOT reversible**) also known as **Hashing functions** or **Digest algorithms**. 
Among the most used hashing algorithms were(MD5) and now SHA1(Git uses SHA1):

-	**MD5** - no longer considered reliable **as a HASH function** (this is not neceseraly what's caused it's demise as the default algorithm to use for password hashing, although the missunderstood fear must have contributed a lot). 
A **colission** happens when two diferent string like *'123456'* and *'str0NG!Pass'* have the same hash. So theoretically the attacker might need to try a reduced set of dictionary words.
But hashing functions inevitable have collisions since they are limited in output. I mean you can MD5 hash a whole document and it's output will still need be 32 chars. So when you are pressing the representation of a bigger string into a smaller one, it's innevitable to have collisions, the questions is how often they occur. See [Collision Resistance](https://en.wikipedia.org/wiki/Collision_resistant) for more details.
But the real reason why MD5 fails as a password hashing mechanism it's because it's fast, as we'll see below.
-	**SHA familly** - SHA1 and SHA-2 variants: SHA-224, SHA-256, SHA-384, SHA-512

see [here](http://en.wikipedia.org/wiki/Cryptographic_hash_function) for a comparison.



You'd say there's not much use for <b>Hashing function</b> if you cannot get back the password you "encrypted" and stored because you cannot validate the user when he presents his password.
Actually it turns out a <b>Hashing function is what you want to use</b>, because you don't have to know the password of the user if you'd proceed like this:
<ol>
 <li>Store the hashed version of the password the user provided at account registration, not the plaintext password in your DB</li>
 <li><b>At authentication</b>, hash the password the user has provided with the same algorithm you used at user registration</li>
 <li>Check this hashed value obtained at (2) if it matches the hashed value you have stored in the DB of the password at (1)</li>
</ol>

If you were to store it encrypted with a reversible algorithm, the attacker(who let's say has in some way access to your users password data) would only need to know the key(which could be in a properties file on the server), the algorithm you used for encryption and he will then have access to all passwords.

There is one case in which you'd need to choose a TWO way encryption though. If you are also using some legacy service that authenticates against the plain-text password like for example a FTP account. Apart from this <b>there is little reason why you should not always use password hashing</b>.


### What if a user loses his password how can you remind it to him?
Simple answer is that you <b>cannot</b> and you should instead offer the user the possibility to reset his password to a new one which you send by email and advise the user to change it as soon as possible since the email is not a secure medium for holding a password.
<b>A site that send you back your original password today is laughed in IT business because it means it shows little respect for user privacy and is a service you should move away from</b>. <b>If you are the owner of such a site it means bad publicity for you</b> but luckily read below how you can fix this.

<h3>Dictionary based attacks</h3>
Some time ago when simply MD5 hashing was the defacto way to encode passwords, in one project a lot of the of people were using "12345" for a password, so if looked at the password field in the database you'd see lots of <i>"827ccb0eea8a706c4c34a16891f84e7b"</i> in there. Even if you did not spend any time decrypting passwords you'd see that user_3, 5 and 6 all have the same password, and that password is in fact "12345". That's how an attacker who has access to your hashed passwords will proceed. He'll search for users who's digest passwords match <i>"827ccb0eea8a706c4c34a16891f84e7b"</i> which he knows is "1234". In reality the attacker might have a huge table of precomputed hash passwords for most common used passwords(maybe words from the dictionary hence the name of <i>dictionary attacks</i>) and tries to match them against yours. This basically reduces the problem to a simple O(1) .


Luckily there is an easy way to protect against this by:

<h3>Salting the password</h3>
We've seen that are simple hashed passwords are susceptible to dictionary attacks. But what if we also added some random string to our "1234" password. It will turn out very different from a dictionary word. This is called <b>salting</b> and makes our digests not equal to the digest of only the encrypted the password alone. 
There are two types of salts:
<ol>
<li><b>Fixed salt</b> - That we try to keep hidden and attach it to every password. Problem is the attacker can use his own password(or the password of another user with a weak password) to try to brute force to find the fixed salt and once the salt is found it will compromise all passwords by generating hash("dictionary word" + fixed_salt)</li>
<li><b>Variable salt</b> - It is generated for every password individually. It's a good idea to be a random one, each stored password will be decoupled from the others, creating a stronger overall protection and highly improving safety for attacks driven against our database of passwords as a whole. But being random this means <b>we'll need to store it along with the password digest</b> to be able to reapply it at the user authentication process. While it's bad because we'll have to store it undigested it's still better because it means the attacker will just need to , he'll need to reapply the because it was 
<li><b>Combination of both types - a fixed hidden salt and a variable one, stored undigested</b> you'd get the benefits of both worlds</li>
</ol>
<b>Use a salt of at least 8 random bytes, and attach these random bytes, undigested, to the result.</b>

<table cellpadding="10">
<tr>
<td style="border:1px dashed;padding:5px;"><b>User</b></td>
<td style="border:1px dashed;padding:5px;"><b>Password digest + Salt</b></td>
</tr>
<tr>
<td style="border:1px dashed;padding:5px;">user_1</td>
<td style="border:1px dashed;padding:5px;">ee7820f711c63b3dee51778ec2a69902random_salt_1</td>
</table>

<h3>Hash iterations</h3>
Ever thought that processing too fast might be considered a bad thing? And if given the choice between two algorithms to use, you'd be wiser to pick the slower one for your application? 
This is true for a cryptographic hash function and the reason is: if the attacker needs to run a brute force generating of hash tables it would mean a much greater effort for him if for every tentative password he generates he has to wait an additional hundred of milliseconds. 
How can you make it slower? Even if the hashing algorithm is fast, by running the hash function several times(by several I mean at least 1000 times), every time feeding as input the value for the previous iteration you can make it as slow as you'd wish. 
<b>Iterating at least a few 1000s times</b> is a good practice.
For you, it means at user login an additional hundred of milliseconds which probably goes unnoticed, for the attacker who has to generate some millions of digests for each password it turns into a big problem.


<h3>Implementation</h3>
The coolest library for implementing security in Java is [Jasypt](http://www.jasypt.org/)
Maven import:
```prettyprint lang-xml
        <dependency>
            <groupId>org.jasypt</groupId>
            <artifactId>jasypt</artifactId>
            <version>1.9.0</version>
        </dependency>

        <!-- only useful for Spring usage -->
        <dependency>
            <groupId>org.jasypt</groupId>
            <artifactId>jasypt-spring3</artifactId>
            <version>1.9.0</version>
        </dependency>
```

The base class for implementing digest is <b>StandardStringDigester</b> which does all the things:
<ol>
<li><b>Use a salt and attach it to the password</b></li>
<li><b>Encrypt by applying hash</b></li>
<li><b>Iterate the hash function</b></li>
<li><b>Encode the result with BASE64</b> - because we it's safer to get the string encoded to Base64 so that it's easier to store in a DB and you're not dependent of the DB character encoding</li>
<li><b>Provide simple method to validate the password on login</b></li>
</ol>

```java
StandardStringDigester digester = new StandardStringDigester();
digester.setAlgorithm("SHA-256"); //we set the algorithm
digester.setIterations("2000");//number of iterations  
digester.setSaltSizeBytes(32); 
digester.setSaltGenerator(SaltGenerator); //if we do not set it explicit, the default is RandomSaltGenerator
...
String digest = digester.digest(userPassword); //we store the digest to persistence storage
...
//

//For authentication we just need to do:
if (digester.matches(inputPassword, storedDigest)) {
  // user authenticated
} else {
  // bad login!
}
```

But wait, where is the salt part retrieve and storage done? It's already contained in the <i>digest</i> string. Since we setSaltSizeBytes(nr_bytes) we know what part of the digest is the salt information. So while it's not stored in a different field, it can still be extracted.

Actually in a multithreaded environment you are better off using
```prettyprint lang-java
PooledStringDigester digester = new PooledStringDigester();
digester.setPoolSize(4);// This would be a good value for a 4-core system 
digester.setAlgorithm("SHA-1");
.... //all the methods used as above
```

### Spring 3.1 usage
Jasypt also makes Spring integration very easy by providing it's own <i>namespace</i>

```
<beans xmlns="http://www.springframework.org/schema/beans"
       ...
       xmlns:encryption="http://www.jasypt.org/schema/encryption"
       ...
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans-3.1.xsd
                           ...
                           http://www.jasypt.org/schema/encryption
                           http://www.jasypt.org/schema/encryption/jasypt-spring3-encryption-1.xsd
                           ...">

<!-- Construct a PooledStringDigester bean that can be referenced. All digesters and encryptors in jasypt are thread-safe -->
<encryption:string-digester id="password-digester" pool-size="4" algorithm="SHA-256" salt-size-bytes="16" iterations="50000"/>

</beans>
```


### Conclusions
 - No excuse to use cleartext passwords anymore
 - If you provide the user's lost password in an email to him it makes you services look lame.</b> Some of your users will understand that your site has a major security issue.
 - Don't try to implement your own security algorithm</b>, there are mathematical geniuses way smarter than me and you who's passion is to think about stuff like this. 
 - Use a hashing algorithm, preferably from the SHA-2 family or better PBKDF2</b>(which by design run slower).
 - Use a random salt
 - Iterate hash
 - Use library which makes security easy

Hope you got some benefit for the article. Till next time... have fun.
