---
layout: post
title: "GovTech Stack the Flags CTF 2022 Writeup 1: Top Secret Admin Panel"
date: 2022-12-10
author: xxbaemaxx
tags: govtech stf22 ctf
excerpt: 'The challenge starts by providing us with a APK file, in which I immediately installed it onto my Genymotion Android VM Emulator (just use the free edition will do) with the adb tool. Upon loading the app, it presents itself with a login interface, prompting me to key in the username and password...'
---
## Walkthrough

The challenge starts by providing us with a APK file, in which I immediately installed it onto my Genymotion Android VM Emulator (just use the free edition will do) with the adb tool. Upon loading the app, it presents itself with a login interface, prompting me to key in the username and password:

<p align="center">
  <img src="/assets/images/2022-12-10-STF22-1-TopSecretAdminPanel/photo1.png" style="width: 25%; height: 25%">
</p>

Next, as part of my mobile app testing framework, I will decompile the mobile app using two tools:
  - apktool (android decompilation to smali code)
  ```bash
apktool d challenge.apk
  ```
  - jadx (decompile to smali code and map it to Java code)
  ```bash
jadx -d $(PWD)/jadx-output challenge.apk
  ```
<p align="center">
![Smali Code with apktool](/assets/images/2022-12-10-STF22-1-TopSecretAdminPanel/photo2.png)
![Java code using jadx](/assets/images/2022-12-10-STF22-1-TopSecretAdminPanel/photo3.png)
</p>

I'd like to take this opportunity to say something to the challenge developer:

Thank you for NOT obfuscating the APK file.

It was honestly brutal when I had to go through obfuscated apps over my years as a security consultant.

As we were presented with a login page, the first instinctive thing to do is to figure out the login functionality and how it works. A quick grep of the login keyword within the jadx output and it directs the user to this file (put filename here):

![Login Function in MainActivity.java](/assets/images/2022-12-10-STF22-1-TopSecretAdminPanel/photo4.png)

First thing that came up to my mind was: "what the hell is Amplify"? A quick Google search and what I got was this:

```bash
https://docs.amplify.aws/
```
![Amplify Framework](/assets/images/2022-12-10-STF22-1-TopSecretAdminPanel/photo17.png)

OOOOOOO~~~ So it's actually a backend framework for mobile/web applications, and it's powered by Amazon Web Services. :D

Not gonna lie, ever since I left consultancy and went to teaching, I have never come across mobile apps that leverage on AWS Identity Access Management as the backbone of user authentication. So this would definitely be a first time for me.

The login mechanism relies heavily on AWS, and this can be seen by the captured web traffic via Burp Suite when attempting to log in with bogus credentials:

![Burp Proxy Traffic for Login Request](/assets/images/2022-12-10-STF22-1-TopSecretAdminPanel/photo6.png)

From here it was pretty much a dead end when I can't proceed past this login page without valid credentials. All hope was lost, until I saw this within the Android XML resource files:

![strings.xml](/assets/images/2022-12-10-STF22-1-TopSecretAdminPanel/photo7.png)


Wait, seriously? -_-"

Normally resources within the strings.xml file contain text values that are eventually used as part of the Android application. But this, is **HIGHLY** unusual. Wonder if it works...

Logging in with the creds             |  Success!!!
:-------------------------:|:-------------------------:
<img src="/assets/images/2022-12-10-STF22-1-TopSecretAdminPanel/photo8.png">  |  <img src="/assets/images/2022-12-10-STF22-1-TopSecretAdminPanel/photo16.png">

WELL WHADDYA KNOW? A classic case of a typical mistake done by developers to clear out previous user credentials :)

Now after logging in, the hint provided was a really obvious one. 3 repetitions to the letter S, and in addition to that, we also have prior knowledge that the mobile app utilises the AWS infrastructure. SOOO.... Amazon + 3 'S' characters = Amazon S3?

Anyway, for us to access any authenticated S3 storage buckets, we would need these information:
- AWS Access ID
- AWS Access Secret Key
- AWS Region

If we look at our Burp Proxy traffic, the following HTTP response was received when we successfully logged into the mobile application:

![AWS creds given to me, me likey :))](/assets/images/2022-12-10-STF22-1-TopSecretAdminPanel/photo9.png)

All you need now is to use the AWS S3 CLI (if you're on Linux, setting this up is easy as hell) by following these steps:

1) Install AWS CLI (Check this website for more info: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
2) Configure an AWS CLI profile with the access ID, key/token and region (I created a separate profile as I too have other AWS keys to manage):
```bash
aws --profile stf22 configure
```
<p align="center">
  <img src="/assets/images/2022-12-10-STF22-1-TopSecretAdminPanel/photo10.png">
</p>

3) if all configurations are correct and well, run the following command and you should be able to see multiple S3 buckets:
```bash
aws --profile stf22 s3 ls
```
<p align="center">
  <img src="/assets/images/2022-12-10-STF22-1-TopSecretAdminPanel/photo11.png">
</p>

HMMMM. Seems like it's not working T_T. After reading the documentation, it turns out that i also need to include the AWS session token as well. For this you will need to find the 'credentials' file and add it yourself:

<p align="center">
  <img src="/assets/images/2022-12-10-STF22-1-TopSecretAdminPanel/photo12.png">
</p>

<p align="center">
  <img src="/assets/images/2022-12-10-STF22-1-TopSecretAdminPanel/photo13.png">
</p>


After further analysis of listing the buckets, only one contained the flag.

<p align="center">
  <img src="/assets/images/2022-12-10-STF22-1-TopSecretAdminPanel/photo14.png">
</p>

Fetching the flag was relatively easy:
```bash
aws s3 --profile stf22 cp s3://stfflag2/flag2.txt flag2.txt
```

Simply view the contents of the file and voila, you have the flag:

<p align="center">
  <img src="/assets/images/2022-12-10-STF22-1-TopSecretAdminPanel/photo15.png">
</p>

## Thoughts about the challenge

I can totally understand how this challenge is considered to be "easy", provided that you know about these two concepts:
- Android App Analysis (Static and Dynamic)
- Familiarisation with Amazon Web Services and its various cloud services (AWS IAM, S3)

I was expecting this challenge to have some form of Dynamic Binary Instrumentation (DBI) stuff like using Frida to perform method overriding, but I guess sometimes the best way to solve it is when the solution is simple and straightforward. No need to be so fancy.

I guess the only thing I can brag about this challenge is that my group was the one that got first blood, but i didn't take photo evidence of it. UGH!
