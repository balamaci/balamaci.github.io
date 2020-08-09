+++
author = "Serban Balamaci"
date = 2012-03-29T14:43:00Z
description = ""
draft = false
slug = "more-secure-than-yesterday-using-recaptcha-with-wicket"
title = "More Secure than Yesterday - using ReCaptcha with Wicket"

+++


We're going for some concrete implementation of a captcha login using <a title="ReCaptcha" href="http://www.google.com/recaptcha">ReCaptcha</a> using with the <a title="Wicket" href="http://wicket.apache.org/">Wicket</a> framework.

### ReCaptcha explained
#### Short Intro
The problem with current captcha images is that there is software that can make out with pretty good precision the letters in a pretty obfuscated image of text. The more you make it harder for that software to distinguish the letters, the more you make it harder also for the human user to read the text.
But the idea behind <b>ReCaptcha</b> is at least an interesting one:
It start with the fact that <b>ReCaptcha</b> it’s also an OCR software being used to digitise old paper books and newspapers. So the OCR software parses the image scans of the old books in the process of pulling out the text from the image and sometimes encounters a word it cannot make out, and so it puts it into a failed OCR attempts register.
Those failed attempts are then used as captcha images for the human user to validate against.
So what makes it interesting is that images offered for user validation, are already the ones that give a hard time to good OCR algorithms in the first place.

#### By implementing ReCaptcha you’re helping also to digitize books
That is the statement on their site at least...
The explanation should be obvious considering what I've already said about the fact that users will have to try to give answers to some distorted images which gave top OCR algorithms a hard time figuring out what the hell was written in those old books Google is trying to digitise and make publicly available for posterity. The more users responses gathered, that point to the same answer the more certain the software is that was the true word the image represented. Add also a validation that puts the word into context of the entire sentence and you've got a pretty sound way to process those old books.
<h4>Prerequisites for usage</h4>
If you choose to read further please take into account the fact that you can use ReCaptcha if:
<ul>
	<li><b>Internet access for your users and server</b>
ReCaptcha is actually a web service that provides the images to the user and you validate on the server the user's response by making a request to the ReCaptcha endpoint for validation. So it's usage is only suitable if both your <b>users and server</b> can reach(have access) to the. Also since you have to <b>register your site</b> to receive a <b>public</b> and <b>private key</b> this means it's not suited for closed intranet applications. It that's the case you'd have to find another captcha solution.</li>
	<li><b>Service not under your control</b>
The service is not hosted by you, so in theory it may happen that the captcha service is down while your application is up. Since Facebook and Google deemed it safe for usage by their own services I think you should not loose any sleep over it going down and rather worry how your services might be taken down.</li>
</ul>


#### Implementation
While I'm a fan of using framework and libraries in the case of the <i>net.tanesha.recaptcha4j</i> library I think we can do a little better on our own and save this additional Maven import. Not to bash the library, but all it does is return two lines of script and make a POST request. My biggest issue with the library is that I would want to use ![Commons HttpClient](http://hc.apache.org/httpcomponents-client-ga/index.html") with a connection pool instead <i>URLConnection</i>(used by the library) to make the user-response validation checks requests, so there is not much use for the library remaining.

Even if you are not using Wicket framework all that what you're supposed to do to add the following inside your User/Password form

```html 
<script type="text/javascript" src="http://www.google.com/recaptcha/api/challenge?k=your_public_key"/>

<!-- this is used when no JS available but it provides the same as above -->
  <noscript>
     <iframe src="http://www.google.com/recaptcha/api/noscript?k=your_public_key" height="300" width="500" frameborder="0"></iframe>
     <br>
     
     <textarea name="recaptcha_challenge_field" rows="3" cols="40"></textarea>
     <input type="hidden" name="recaptcha_response_field" value="manual_challenge">
  </noscript>
```

The effect of the above code is to generate a html structure that will contain the
```prettyprint lang-java
<div style="" id="recaptcha_widget_div" class=" recaptcha_nothad_incorrect_sol recaptcha_isnot_showing_audio">
<div id="recaptcha_area">
<table class="recaptchatable recaptcha_theme_white" id="recaptcha_table">...
   <tr><div id="recaptcha_image" style="width: 300px; height: 57px;"><img width="300" height="57" src="http://www.google.com/recaptcha/api/image?c=03AHJ_Vus.."></div>...
   </tr>
   <tr>...
      <input type="hidden" value="03AHJ_VusIYkHVoO60mM3bF92Wbjc-Ib2..." id="recaptcha_challenge_field"/>
   </tr>
   <tr>
   <input type="text" id="recaptcha_response_field" name="recaptcha_response_field">
   </tr>...
</table>
...
```

So basically you will now have also two additional input fields to your form **recaptcha_challenge_field** and **recaptcha_response_field** generated by the above JS.

These two parameters can be retrieved on form Submit where we need to verify them against the <b>ReCaptcha</b> service endpoint

So the java code for a <b>CaptchaPanel</b> would look like:

```java
public class CaptchaPanel extends FormComponentPanel {

    private static final String RECAPTCHA_ENDPOINT = "http://www.google.com/recaptcha/api/";

    @SpringBean
    private ExternalService externalService; //this is my service for validating the Captcha

    public CaptchaPanel(String id) {
        super(id);

        WebMarkupContainer scriptLabel = new WebMarkupContainer("script") {

            @Override
            protected void onComponentTag(ComponentTag tag) {
                tag.put("src", RECAPTCHA_ENDPOINT + "challenge?k=" + MY_PUBLIC_KEY);
            }
        };
        add(scriptLabel);

        WebMarkupContainer noScriptLabel = new WebMarkupContainer("noscript") {
            @Override
            protected void onComponentTag(ComponentTag tag) {
                tag.put("src", RECAPTCHA_ENDPOINT + "noscript?k=" + publicKey);
            }
        };
        add(noScriptLabel);
    }

    @Override
    public void validate() {
        String challenge = getRequest().getRequestParameters().
                getParameterValue("recaptcha_challenge_field").toString();

        String userResponse = getRequest().getRequestParameters().
                getParameterValue("recaptcha_response_field").toString();

        String remoteIp = NetworkUtil.getRemoteIp((HttpServletRequest) getRequest().getContainerRequest()); //utility class for getting the remote ip, keeps account of

        boolean validatedCaptcha = externalService.
                isCaptchaValid(remoteIp, challenge, userResponse);
        if(! validatedCaptcha) {
            error("Captcha validation failed");
        }
    }

    //this part is only used for customization of ReCaptcha(see Customization)
    @Override
    public void renderHead(IHeaderResponse response) {
        super.renderHead(response);

        PackageTextTemplate optionsRecaptcha = new PackageTextTemplate(CaptchaPanel.class, "recaptcha.options.js");

        response.renderJavaScript(optionsRecaptcha.getString(), "recaptcha");
    }
}
```


the html is just the script part added, nevertheless here it is again <b>CaptchaPanel.html</b>:


also the verification of the values against the ReCaptcha service:

```prettyprint lang-java
    public boolean isCaptchaValid(String remoteIp, String challenge, String userResponse) {
        HttpPost postMethod = new HttpPost(RECAPTCHA_VALIDATE_ENDPOINT);
        List<NameValuePair> nameValuePairs = new ArrayList<NameValuePair>(4);

        String privateKey = StoreWebRequest.get().
                getSettingValue(SettingKey.RECAPTCHA_PRIVATE_KEY);

        nameValuePairs.add(new BasicNameValuePair("privatekey", privateKey));
        nameValuePairs.add(new BasicNameValuePair("remoteip", remoteIp));
        nameValuePairs.add(new BasicNameValuePair("challenge", challenge));
        nameValuePairs.add(new BasicNameValuePair("response", userResponse));

        try {
            postMethod.setEntity(new UrlEncodedFormEntity(nameValuePairs, HTTP.UTF_8));
        } catch (UnsupportedEncodingException e) {
            //log
        }

        HttpResponse response = null;
        try {
            //HttpClient in my case obtained from connection pool of type ThreadSafeClientConnManager
            response = httpClient.execute(postMethod);
            if(response.getStatusLine().getStatusCode() != 200) {
                return false;
            }

            String responseString = IOUtils.toString(response.getEntity().getContent());
            if(responseString.startsWith("true")) {
                return true;
            }
        } catch (IOException e) {
            log.error(e.toString(), e);
            return false;
        } finally {
           if(response != null) {
            try {
                EntityUtils.consume(response.getEntity());
            } catch (IOException e) {
                log.warn("can't consume entity", e);
            }
           }
        }
        return false;
    }
```

#### Customization
Customization relies on the fact that you can construct a <b>RecaptchaOptions</b> object in Javascript before the code that . As seen in the <b>CaptchaPanel</b> above we also add the in the response:

```prettyprint lang-js
var RecaptchaOptions = {
    lang : 'ro',
    theme : '${theme_name}'
 };
```

Full list of what you can customize <a title="Customizing ReCaptcha" href="https://developers.google.com/recaptcha/docs/customization">here</a>

Interesting that you can even fully customize the layout of the fields see <b>Custom Theming</b> in above link.

#### Conclusion
So you've seen how simple it is to do it in any framework not just Wicket, why not use it in places where you need to make sure there is a (apart from the fact that you'll give your users a hard time distinguishing the word in the images:) ?

Cheers!
