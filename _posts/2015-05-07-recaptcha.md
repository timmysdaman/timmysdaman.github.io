---
title: reCAPTCHA on a Salesforce Marketing Cloud (SFMC) Landing Page
---
Have you ever been annoyed at CAPTCHA technology on a page?  And then later, much to your chagrin been the person implementing CAPTCHA?  I know your pain... this post comes from exactly that. 

The following is an example of using Google's reCAPTCHA on SFMC landing pages.  Although this is tailored landing page solution, it should be easily adapted for use in other environments. It pains me a little to say this, but Google did a really nice job making this easy to implement.  Good documentation was a big plus as well. 

### Prerequisites
1. Set up <a href='http://help.exacttarget.com/en/documentation/exacttarget/content/microsites'>microsites</a>  in your SFMC account, with 3 landing pages of type HTML.
2.  <a href='http://www.google.com/recaptcha/intro/index.html'>Register.</a>
   * Get your keys, both public and private.
   * Register a domain there (since my landing page code is on pages.exacttarget.com, this is what I registered).
3. (Optional) Make sure that the page you want reCAPTCHA on has some decent client side validation on it.  Passing bad data is a pretty good way to mess yourself up before you even get started.
4. A basic understanding of SFMC AMPscript, SFMC SSJS, SFMC landing pages, javascript and html would help a lot.

<br>

### First Page
The first page we need is going to be the form which we're going to post from.  (For the sake of this demo, I'm not actually posting the data.  Do remember these are pages within the SFMC).   For now, I'm going to start with a basic, seemingly inconsequential form:
![Test](../img/simpleForm.png "Simple Form")

<br>After reCaptcha:

![Test](../img/simpleFormReCaptcha.png "Simple Form")

We're going to paste in the following script block into the header:

    <script type="text/javascript">

    function validateCaptcha(){
        var xmlhttp;
        var c = Recaptcha.get_challenge();
        var r = Recaptcha.get_response();
        var param = "challenge="+c+"&response="+r;
	    	    
        if (!r) {
            alert("Please enter the Image Verification (Captcha) code.");
            return (false);
        }
        if (window.XMLHttpRequest) {// code for IE7+, Firefox, Chrome, Opera, Safari
            xmlhttp=new XMLHttpRequest();
        }
        else {// code for IE6, IE5
            xmlhttp=new ActiveXObject("Microsoft.XMLHTTP");
        }
        xmlhttp.onreadystatechange=function() {
            if (xmlhttp.readyState==4 && xmlhttp.status==200) {
                var sindex = xmlhttp.responseText.indexOf('<resVld>');
                var eindex = xmlhttp.responseText.indexOf('</resVld>');
                var keyStr=xmlhttp.responseText.substring(sindex+8,eindex);
		    
                if(keyStr==="true"){
                    document.forms[0].submit();
                }
                else{
                    alert("Please enter the valid Image Verification (Captcha) code.");
                    Recaptcha.reload();
                    Recaptcha.focus_response_field();
                }
            }
        }
        xmlhttp.open("POST","LOCATION_OF_PROCESSING_PAGE",true);
        xmlhttp.setRequestHeader("Content-type","application/x-www-form-urlencoded");
        xmlhttp.send(param);
        return false;
    }
    </script>
    
All posting is essentially disabled via javascript, except in the line `if(keyStr==="true")`.  The javascript needs to be wired up to the onsubmit method in the form tag.  

Next, we need to put some javascript somewhere in the body tags, to actually put the reCAPTCHA widget onto the page.  Note there are two separate scripts:


    <script type="text/javascript" src="http://www.google.com/recaptcha/api/js/recaptcha_ajax.js"></script>
    <script type="text/javascript">
	Recaptcha.create("PUBLIC_KEY_GOES_HERE", 'captchadiv', {
	    tabindex: 1,
	    theme: "clean"
        });
    </script>

"captchadiv" is the name of some div that you have on your page.  My page has the following line on it:

    <div name="captchadiv" id="captchadiv"></div>

###The Processing Page
Next up, we have the page where the magic happens, the processing page.  Although it may at first glance look like this page isn't necessary, this is where the private key lives (and it lives server side for security purposes).   On the second page, put the following code:

    %%[
        SET @c = RequestParameter("challenge")
        SET @r = RequestParameter("response") 
    ]%%
    <script type="text/javascript" runat="server">
        Platform.Load("core", "1");
        var response = [0];
        var responseDetails = {};
        var challenge = Variable.GetValue("@c");
        var reCaptchaResponse = Variable.GetValue("@r");
        var headerNames = ["MyTestHeader1", "MyTestHeader2"];
        var headerValues = ["MyTestValue1", "MyTestValue2"];
        var contentType = 'application/x-www-form-urlencoded';
        var url = 'https://www.google.com/recaptcha/api/verify';
        var privateKey = 'PRIVATE_KEY_GOES_HERE';
        var remoteIP = "192.168.1.1"
        var payload = 'privatekey=' + privateKey + '&remoteip=' + remoteIP + '&challenge=' + challenge + '&response=' + reCaptchaResponse;
        responseDetails.StatusCode = Platform.Function.HTTPPost(url, contentType, payload, headerNames, headerValues, response);             
        responseDetails.Response = response;
        var res=responseDetails.Response[0].substring(0,4);
        var resp='<resVld>'+res+'</resVld>';
        Write(resp);
    </script>

No need to change the IP address, or the header names and values, this are just necessary for the POST.  After you create this page, make sure to put it's URL back on the original page.  


###Last Page - Success!
The last page to create is mainly a place holder for this example.  Mine is just an HTML page that says "Hooray! Captcha worked!", and yours can say whatever.  The URL of this page should go in the action attribute in the form tag on the first page.  

Congratulations! In just a few minutes, you were able to set up reCAPTCHA on a landing page.


