+++
date = '2019-06-10'
draft = false
title = 'Web Application Formal Exploiter: a formal and automated approach to exploit multiple vulnerabilities of web applications"'
tags = ["phd", "wafex", "MBT", "model-based testing", "formal analysis", "WAPT", "web application security", "multi-state attacks", "exploitation", "model-checking"]
+++

## Introduction

In this post I want to briefly present the result of my Ph.D. research. I won't go into the technical details of the project, that you can find in my [Ph.D. thesis](https://iris.univr.it/retrieve/handle/11562/979770/113719/thesis%20de%20meo.pdf), but I'll rather give an high-level overview of the work. I hope it might be of interest and inspiration to some of you.

## Where I started
When I finished my MSc I had a pretty good understanding of the main vulnerabilities of web applications. I had also played around with most of the main vulnerable by design web applications such as [WebGoat](https://www.owasp.org/index.php/Category:OWASP_WebGoat_Project), [DVWA](http://www.dvwa.co.uk) and [Gruyere](https://google-gruyere.appspot.com) however, I felt like something was missing. Although everything I had studied so far was clear to me, I found asking myself: "alright, but what would a real attacker actually do once he found an issue?". The goal of a real attacker is not to **find**, for instance, an SQLi but rather to **exploit** the SQLi to achieve something, right?? 

Imagine playing a video game. A vulnerability is a "basic move" like pressing the `K` button that results in having your character do a kick.

![step1](/images/step1.png)

But a kick is not enough, you have to press multiple buttons if you actually want to inflict some pain.

![step2](/images/step2.png)

I thus started investigating how vulnerabilities might be combined together, how they could be used in a more realistic attack scenario in order to compromise a web application. I read many different resources, three of which turned out to be very significant to me.

### The [Common Weakness Enumeration](https://cwe.mitre.org):
> CWE is a community-developed list of common software security weaknesses. It serves as a common language, a measuring stick for software security tools, and as a baseline for weakness identification, mitigation, and prevention efforts.

What I found interesting about the CWE was that each vulnerability has a section called "Common Consequences" where you can find an high-level description of the most common ways an attacker might exploit that issue to compromise the security of a web application (or software in general). This was a first step into the right direction as I could start to better understand the common result of exploitning a given vulnerability.


### The book "Chained Exploits: Advanced Hacking Attacks from Start to Finish". 

This was a very fun reading, maybe a bit old nowadays, but still it gives a interesting insight in the mind of an attacker trying to compromise a system. The book is a collection of hacks perpetrated by the main character Phoenix. You get to follow Phoenix's thoughts and reasoning process while he attempts to break into places. From a technical point of view is not too interesting but again, the way of thinking is what I found more fascinating especially for newcomers.

### Attack trees\graphs.

Let me borrow the first line from Wikipedia:
> Attack trees are conceptual diagrams showing how an asset, or target, might be attacked.

Pretty self explanatory. You basically have a means, more or less formal, to represent attacks. Attacks are divided into steps that define how to get from a starting point all the way to a goal point. I read some academical publications on the subject and it was immediately obvious that attack trees\graphs are mainly used to represent known attacks to software known to be vulnerable to those attacks.
I also found some research where steps where decorated with pre and post conditions to increase the details of the tree\graph. For example, you can use attack trees\graphs to represent your network, where you might have an ancient Windows XP machine. You can then represent the fact that this Windows XP machine is vulnerable to CVE-XYZ which will allow an attacker to access a Red Hat machine (for whatever reason).

But what if you don't know if your software is vulnerable? What if you only have the starting point and the goal point?

This is where my Ph.D. journey (sort of) began and brought me to proposing the Web Application Formal Exploiter (WAFEx), a formal approach to testing the security of web applications based on the combination of multiple steps.

## Web Application Formal Exploiter (WAFEx)
WAFEx is a [Model-Based Testing](https://en.wikipedia.org/wiki/Model-based_testing) approach which combines model-checking and testing techniques to find and exploit complex attack scenarios automatically. It does sound cool, doesn't it? To better explain what that means, allow me to set a scenario to make things funnier and easier:

You're a security expert working for a gigantic money maker corporation you don't even know what they actually do. One day, this sweet old man approaches you.

![palpatine](/images/palp.png)

You become friends and he gives you a bad ass nickname. The sweet old man sees something in you and he wants you to be a part of a secret project he's been working on. He shows you a web site and tells you that it contains the blueprints for something that will change humanity forever and, of course, he wants to make sure that no one can have access to such data.

You start poking around on the website and soon enough you find a Reflected XSS, you go to the sweet old man and tell him you found the issue. He looks at you and asks "can it be used to access the blueprint? and how would you do it?"
At this point you will probably come up with a bunch of sentences beginning with "if", such as "if condition X holds, then someone might do Y, which will result in Z which in turns will result in accessing the blueprint".

This is where WAFEx comes in, WAFEx is not meant to find security issues such as a Reflected XSS,  WAFEx is meant to be used to answer questions such as "can someone access the blueprint on the sweet old man's website?".

**DISCLAIMER:** before proceeding with the description of WAFEx, I feel like I have to make a clarification. From the example above it might, at first impression, look like that if you find an XSS that you either don't know how to exploit or cannot use to harm anything of interest, then it is not important and you don't need to fix it. This is far from being the case, whenever an issue is found it should always be planned to fix it ... eventually. This in turns might open another question: "but if I still have to fix everything, why do I need WAFEx to tell me if an issue can or cannot be used to access the blueprint?". Here some reasons:
* convincing your boss to fix it immediately,
* having a better understanding of how that particular vulnerability might be exploited on your specific target,
* identifying the minimum number of vulnerabilities to fix in order for the blueprint to be safe,

and if that is still not enough ...


![because we can](/images/tbbt.png)


## How does WAFEx work

I will now explain how WAFEx works by means of the following workflow. If you are already familiar with MBT, this picture won't be anything you haven't seen before.

![MBT workflow](/images/mbt_workflow.png)

Here is a brief explanation of what is going on:

1. A formal model of the web application has to be created. This model essentially represents the web application and how it interacts with other entities such as the file-system, the database and the end user;
2. The formal model must be analyzed in search of attacks. This means feeding the model of the web application to an attacker that will look for attacks;
3. Finally, with the result of the previous step, we test that the attacks found by the analysis can be replicated on the real web application.


### (1) Creating the formal model

The first thing is creating the formal model of the web application you want to test. The model of a web application comprises five entities: the **web attacker**, the **database** the **web application itself** and an **honest client**.

It sounds like a lot of work, doesn't it? Not to worry, as the formalisation I proposed is such that you only have to take care of one entity: the **web application**. All other entities are going to be the same for every scenario. In fact, all other entities are formalised in such a way to represent the general behaviour of the real entity they take the name from and thus are not tied to a specific scenario.

Let me briefly describe the 4 static entities:

* the **web attacker** is essentially the [Dolev-Yao (DY)](https://en.wikipedia.org/wiki/Dolev%E2%80%93Yao_model) attacker model extended with a particular constant `malicious` that gives him the ability to perform whatever action he pleases (more on this later, bear with me);
* the **file-system** defines 2 basic operations: read and write of files;
* the **database** defines 6 basic operations: query, insert, delete, update, read\write from file-system;
* the **honest client**, is basically a dummy version of an user in front of a browser interacting with the web application.

The entities described above can be used in every scenario, while you only have to take care of creating the model for the web application. This can be done by using a [Burp Proxy plug-in](https://github.com/rhaidiz/wafex-model-creator) I specifically wrote for this purpose. WAFEx model creator is a Burp Proxy extension meant to ease the process of creating the model of a web application. The way it works is that you browse the web application via Burp and collect requests. Once you have a pool of requests of your interest, you send them to the WAFEx model creator.

![WAFEx creator 1](/images/wafex_creator1.png)

The WAFEx model creator uses the requests to create a skeleton model of the web application. It will automatically generates the template for all the requests. You only need to further refine the model by including interaction with the database, the file-system and any details want to test.

![WAFEx creator 2](/images/wafex_creator2.png)

With the model ready, you now have to define the security property you want the model to hold. I focused my attention to two main security proprieties: authentication bypass and confidentiality breach. However, you can specify whatever security property you can think of as long as you can write an LTL formula of it.



### (2) Analysing the formal model
Once the formal model is ready, you give it in input to a model-checker that implements the DY attacker model and let it work its magic. Briefly, the DY attacker model is a formal model used to prove properties of network cryptographic protocols. You feed the DY intruder with a model describing the design of a network protocol and a property you want to be true over your model. The DY will try its best to prove that the given property does not hold and, if it succeeded, it generates a formula that proves that the property doesn't hold so that you can fix your model to make sure that the property holds.

Once you start the analysis, you are gonna have to wait for about 10 minutes, get a coffee, come back, wait another 10 minutes ... keep waiting until, if you have Saturn in your chart, you might eventually get an answer. (did I mention you might never get an answer?? more on that later)

I previously mentioned that with the constant `malicious` the DY attacker can perform whatever action he pleases. In fact, every entity is formalized to react upon receiving the constant `malicious` by performing any of the action they support. For instance, the database, upon receiving `malicious` might perform a delete query, or might perform a normal query and extract information. The model-checker will decide which action should be performed once the constant `malicious` is received by an entity. Given that the model-checker will explorer every possible states, it will eventually explore a path where the combination of certain specific actions are going to violate the property you have defined on your model, thus defining for you the set of actions needed to attack your model... Pretty cool eh?


### (3) Testing

If you're lucky enough to get an answer from the model-checker (again, more on this later) there's one last step to perform: **testing**. As I mentioned early on, WAFEx implements a Model-Based Testing approach, which means that, in one way or in the other, you still have to perform **testing** on your system at some point. What differs however, is that the tests that you will run are going to come from the result of formally analysing the model of the web application with a model-checker. This means that you are not going to randomly test everything everywhere, but you will perform specific tests in specific parts of the web application by following the attacks generated by the model-checker.

I have thus implemented a python engine to automate the process of testing the attacks generated by the model-checker. The python engine takes the output provided by the model-checker and runs it on the real web application. The python engine also takes advantage of external tools such as [sqlmap](http://sqlmap.org/) and [wfuzz](https://wfuzz.readthedocs.io/en/latest/) for payloads generation purposes (as you remember, WAFEx is not meant to generate payloads itself).


## Limitations

Remember the whole "if you're lucky enough to get an answer"? this is the main problem with model-checking, the model you analyze might be so complicated that generates an enormous number of states to analyze. As a result, the analysis might never terminate, might get stuck in a loop, and you'll never get an answer, at least not within a reasonable timespan. There are some tricks you can implement to avoid this states explosion to happen and to deal with them when they do happen (sometimes there is only so much you can do about it). To avoid states space explosion you want to keep your model as simple as possible. However, in case you cannot keep it simple, you can instruct the model-checker on how to behave. You might opt to specify a bound or choose between BFS vs DFS visiting algorithm. Sometimes it might help to tweak the model-checker itself however, that's quite difficult and require you to have some knowledge of how the model-checking algorithm works. Otherwise you can just go random, try them all and see if they work :-)

There are also some limitations with the kind of vulnerabilities WAFEx can deal with at the present time. This limitation, however, only depends from the fact that a Ph.D. has a limited time (mine was 4 years) and once time's up ... well ... it's done.

However, WAFEx can be extended to include more details and thus represent more vulnerabilities and more complex scenarios which, obviously, requires further research.

## Conclusions

Web Application Formal Exploiter (WAFEx) is a formal, model-based testing approach for the security analysis of web applications. Its main purpose is to find attacks to web applications that are the result of combining multiple vulnerabilities together. It is not meant, and never was, to be a point-and-click tool to find SQLi, XSS, CSRF or any other specific vulnerability.

Is WAFEx going to replace penetration testers? ... Of course not, at least not in its current form, but until then ...


<style>
@font-face {
  font-family: "Harry";
  src: url(/fonts/hp.ttf) format("truetype");
}
</style>

<div style="text-align:center;font-family:Harry;font-size:5em">mischief managed</div>
