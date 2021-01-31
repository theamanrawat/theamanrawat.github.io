---
layout: post
title:  "How I hacked my Gym"
description: In this writeup I will show you that How I database.
---

Hi, today I will show that How I hacked my Gym. I joined Gym in january and they sent me their android app with username and password on email. You know what will happen when you give an app or a website to any Hacker :)

![image](../../../assets/images/gym-hacking-1.png){: .imagePopup}

I setup Burp Suite with my rooted android device that I mostly use for android application testing. I installed application on my phone and then trying to capture the request but due to ssl pinning I was not able to capture the requests but bypassing ssl pinning was easy in that application. I used SSLUnpinning to bypass it and Now I was able to capture all the requests.

I loggedin into my account and start navigating the application to capture all the request in Burp Suite. In login page I added `'` in my username and I got SQL error.

![image](../../../assets/images/gym-hacking-2.png){: .imagePopup}

I used sqlmap and I was able to dump the database. Now I want RCE (Remote code execution) so I tried to enumerate the database user's permission but the user didn't has privileges that would allow me to upload files into system. I found few password hashes which includes the admin password. I ran hashcat to crack the password hash.

```
hashcat -m 400 -a 0 hash.txt rockyou.txt
```

But didn't get anything. I could only view database and the data was useless for me then I decided to go further and I started testing their website :) 

I created an account on their website where I had a functionality to add clients and trainers and that took my attention because we can also upload profile picture so I added a trainer

![image](../../../assets/images/gym-hacking-3.png){: .imagePopup}

I edited the trainer's profile picture and tried to upload php file because the application was build on PHP so I changed the file extension to `.php` and content-type to `application/x-httpd-php`


![image](../../../assets/images/gym-hacking-4.png){: .imagePopup}

Opened image in new tab that I uploaded and I got the id of that system.

![image](../../../assets/images/gym-hacking-5.png){: .imagePopup}

and then used [weevely](https://github.com/epinna/weevely3) to get shell and now I was able to read and write any files so I changed admin panel file where it was asking for password and now I can login into any account by just knowing their email address. I had all the email addresses which I got from sqlmap and I loggedin into my GYM's admin dashboard where I can renew my membership. 

It was very easy and fun to hack them.