---
title: "security advice for the average joe"
date: "2015-11-13"
slug: "2015/11/13/security-advice-for-the-average-joe"
categories:
  - networking
  - security
cover: "/images/password2.jpg"
---

Introduction
============

A computer lets you make more mistakes faster than any invention in human history - with the possible exception of handguns and tequila. --Mitch Ratliff

The Internet has gone through a massive transformation since it's inception. From a tool used mostly by academics, it has come to be a pervasive tool used by just about everyone to communicate, shop, pay bills, invest, and entertain.

While the use cases never cease to increase, one aspect of Internet usage that is rather problematic is educating the public about the risks involved in living a connected life, and what are the ways people can defend against attacks.

Here is a non-exhaustive list of threat vectors to be cognizant of:

<!-- more -->

* Privacy: Companies, governments, and criminals are potential entities which may have an interest in collecing information about you without your knowledge. This information can be collected for benign reasons (personalizing a news feed based on inference of your online behaviour), yet can possibly be used against you (blackmail, discrimination, and so on).
* Fraud: Criminals are always looking for new ways to swindle, the online world has made it much easier for large scale scamming operations as the barrier to reaching out to millions of people is now very low.
* Theft: A whole new landscape has emerged on ways for criminals to steal things. If a criminal can gain access to your bank account, they can empty your life savings in seconds.

In this post I will review various ways that can help keep your online life safe.

Passwords, now entering the  DANGER ZONE!!!
===========================================

![DangerZone](/images/dangerzone.png)

Fact #1 
-------

The majority of the population is terrible at choosing good passwords.

See your password here?

![Password fail](/images/password2.jpg)

Fact #2
-------

Bad actors love to break into things by attacking password based authentication systems. Why? See Fact #1

Brute force
-----------

Here's some math to help everyone understand. 

Let's imagine the following password: **"aA4$ 6z"**

To break this password, we need to calculate all possible permutations:

* 7 Characters
* 96 possible character combinations
* 75 trillion possible combinations

Now that may seem like alot, but an average home PC can calculate about 4 billion combinations per second, so this hard to remember password will take about 5 hours to break. Ok, so that's bad, however that's not to say that this type of attack is feasible against all systems. After all, first the bad guys need to somehow grab a list of password hashes then try to crack them using brute force. Which leads us to...

Dictionary attacks
------------------

Remember Fact #1? Well here's the thing. Since people are terrible at choosing passwords, the bad guys often don't have to steal a database of passwords. By simply trying commonly used passwords, it's often possisble to break into systems. To make things worse, over the years even hard to guess passwords have been broken by brute force techniques, and databases of passwords that are re-used have been created. So your 15 character password which could take a billion years to break using brute force, may already be part of a password dictionary attack that could break into your account in seconds.

Recent data breaches have made this easy to see. In some cases up to 20% of passwords in recent breaches are cracked within hours of the breach. Using the top 100 re-used passwords often leads to hundreds of thousands of compromised accounts. And these numbers don't include the individual accounts broken into via targetted attacks that never make the news.

This is both an indication of a lack of complexity as well as password re-use.

Rules for passwords that don't suck
-----------------------------------

1. The best passwords are randomly generated. Here's a good one: **"SD#$5fdgxvn w5436 sw345ZDCVQ#$%SEW$Dgfhy4q5oretuzkfdjgnzksjdhl(iw3q45ypa4rtz"**. Don't use this one.
2. Use as many different types of characters as possible: Uppercase, lowercase, symbols, numbers. If you can input symbols in other languages do that too.
3. Make them as long as is allowed. If you can enter a 100 character password, that's how long it should be.
4. Don't ever re-use a password. If one password gets compromised, others are still safe from dictionary attacks.
5. Don't maintain a list of passwords unless you're encrypting the list.

Ok, most people throw up their hands at this point with the following observations:

* There is no way to remember these passwords
* Typing these passwords correctly is next to impossible.
* I don't know what encryption is or how to do it. How can I keep my list of passwords safe?

Password Managers to the Rescue
-------------------------------

It's simply not possible to follow all of the above rules without using a tool to manage passwords. The gigantic advantage of using a password manager is that it frees you from having to remember *ALL BUT ONE* password.

Password managers I've used:

* [LastPass](https://www.lastpass.com): Free and paid versions
* [KeePass](http://keepass.info/): Free app
* [1Password](https://agilebits.com/onepassword): Free and paid versions

A password manager takes care of generating passwords for you, and saving them. Some support synchronizing your password database between devices automatically (PC, laptop, phone, tablet, etc...)

My current favorite is LastPass, which I've bought the paid version for (12$/year). It works great, has a browser add-in and works on Android, IOS, PC, Mac and syncs between all devices.

I simply cannot stress this enough, your first line of defense in online security is using strong passwords, and password managers are the best way of doing this.

* Stop reading now. 
* Go install a password manager.
* Take an hour or two and change ALL your passwords.
* Keep reading.

Beyond Passwords: Multi-factor authentication
=============================================

Not just for spies anymore!
---------------------------

Even the safest, most impossible to decrypt password in the universe is not immune to programming errors. On average, developers typically have at best rudimentary knowledge of security concepts. Coupled with the fact the software development is an extremely challenging profession to master, it comes as no surprise that bugs happen. Security bugs may completely compromise or disclose a password.

All hope is not lost however, as some sites/systems implement a second layer of defense during authentication. This second layer is typically called multi-factor authentication, or two-factor authentication (2FA).

The idea behind multi-factor authentication is pretty simple. Combine

* Something you know (eg: a password), with something you are (eg: a fingerprint, iris scan, heart rate signature, facial recognition, etc..)
* Something you are (eg: iris scan), with somewhere you are (eg: GPS coordinates, IP geolocation, etc...)
* Something you know (eg: passphrase), with something you have (eg: token, mobile phone)
* And so on...

In the case of multi-factor authentication, the compromise of a single factor (eg: your password is leaked online) does not allow an attacker to gain access to your account (eg: they would need the second factor, for example your mobile phone).

A weaker form of multi-factor authentication is using the same factor twice. For example two things you know (a password, and a security question), two things you are (fingerprint, iris scan). Arguably that last one is pretty hard to fake...

The most common implementations I've come across are

* One time passwords via hardware or software tokens
* SMS/phone calls

I'm sold! How can I get this?
-----------------------------

Easy peasy. Go [here](https://twofactorauth.org/). It's a website that tracks sites that have implemented two-factor authentication.

* Stop reading this page. Click the link above.
* Go setup 2FA everywhere you can (gmail/facebook/linkedin/yahoo/hotmail/your banks/online trading/etc...)
* Keep reading.

The list may not be exhaustive, so if you use other services not listed (basically anything you have a password for), contact their technical support to find out of they support a 2FA scheme, and if so get it setup.

Multi-factor authentication is the second strongest possible protection against attackers breaking into your life.

Just as a side note, many of these sites will allow you to setup a recovery email/phone number, please do so. This will allow you to reset things if someone steals your phone/token.

Mobile Security
===============

There is a ton of sensitive information on your phone that could be used against you or is valuable to you.

* Your contacts
* Your personal email
* Apps that connnect to your bank account
* Your work email
* Social media apps
* Pictures, videos, etc...
* etc...

You should setup whatever protections you can to prevent unauthorized people from using your phone. At the very least, enable the pin screen locking mechanism. Even better if your phone supports a fingerprint reader.

If possible, setup a remote find/wipe application that will allow you to remotely wipe all the information on your phone in the case where it is stolen or lost. Another thing to configure is to setup automatic wipe of the phone if a pin/password is incorrectly entered too many times. Keep in mind that if you have kids/cats they might inadvertently wipe your phone while trying to guess your password or mistyping it.

USB drives
==========

These great little portable devices are a great way of propagating viruses, malware or even worse!

Let's take a look at this harmless looing device:

![usb](/images/usb1.png)

Looks harmless enough right? So it _could_ be a USB drive. The files on it that could potentially be infected with viruses or malware. Some of these types of infections can take over your computer by simply plugging them in (without even opening any files).

With a bit of luck, your anti-virus software will keep you _safe-ish_.

However it could even be nastier! How about this:

![usb](/images/usb2.png)

This is a miniature computer embedded into a USB stick. When plugged into your computer, it'll behave like a normal USB stick, however while it's plugged in it will start trying to attack your computer by sending commands as if it was a connected keyboard. Within seconds it can be running programs that allow remote attackers to gain full access to your computer.

My advice is that if someone gives you a promotional USB stick, or you happen to find one lying on the ground, throw it away.

In fact, as a general rule, it is not always safe to assume that even a factory sealed device may be free of threats as there have been several cases of devices being shipped with malware onboard. Break out your tin hats!


Basic Internet Hygiene
======================

Email
-----

Don't do this:

![email](/images/openemails.png)

There are several different classes of attacks that use e-mail as the initial vector of exploitation. 

There are a some really important things everyone should understand about e-mail:

* Opening up an e-mail is sometimes sufficient to exploit and attack your computer. 
* Email attachments can and often do contain files infected by viruses.
* People blindly trust people they know, even though their contacts may have compromised systems and can be inadvertently attacking them.

Here are tips on using e-mail safely:

* If you don't know the sender, you can simply delete the email without opening it.
* If it looks like it might be spam, you can simply delete the e-mail without opening it.
* If an email is not directly addressed to you by name, delete it.
* Check the email address domain against information in the email. For example, if the email claims to be from your bank but the sender of the email is something like yourbankname@fastscam.tz, it's fake. If things don't line up, delete it.
* Don't ever click on links from untrusted sources.
* Don't click on hidden links like [THIS AWESOME SITE HAS NAKED PICS OF OBAMA](http://goo.gl/50Qodi)
* Don't click on shortened links like this one: http://goo.gl/50Qodi shortened links make them easier to type or tweet, however it's a tool often used by attackers to make links look harmless. If a link looks like one of these, don't click: goo.gl, bit.ly, ow.ly, t.co,tr.im, tinyurl.com
* Ensure that the mouse-over link (in the bottom of your browser) is the same as the link text. For example [http://www.google.com](http://goo.gl/50Qodi)
* Never respond to emails that ask you for information or promise you a reward. Same thing for those emails with ONE LAST STEP BEFORE YOU WIN THIS AWESOME PRIZE. They are fake. Delete them.
* In general, avoid links sent by e-mail even from senders you trust. If you do not recognize the link destination, don't click on it.

Operating system patching
-------------------------

Keep your operating system up to date with patches from the vendor. Over time software vendors patch their systems for known security holes, and if you aren't keeping everything up to date, you're exposing your system to the risk of exploitation.

Anti-Virus
----------

Install a decent anti-virus. Almost all vendors offer free versions for home users. I use [BitDefender](http://www.bitdefender.com/) which appears to have a decent detection rate.

Firewall
--------

Configure your host-based firewall. 

For Windows: 

```
Start -> Control Panel -> Administrative Tools -> Windows Firewall with Advanced Security
```

Or
```
windowskey+R, then type
wf.msc
```

For MacOS: [https://support.apple.com/en-ca/HT201642](https://support.apple.com/en-ca/HT201642)

Secure your communications using HTTPS
--------------------------------------

Go through your bookmarks, try all of them using https instead of http. If it works, you should edit your bookmark to only connect via https. There is even some browser extensions that try to do this for you.

For example: [https://www.eff.org/https-everywhere](https://www.eff.org/https-everywhere)

