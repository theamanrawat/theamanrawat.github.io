---
layout: post
title:  "Hackershala's CTF walkthrough"
description: This is the wackthrough of CTF conducted by Hackershala.
---
Hi today I am gonna do writeup of hackershala hackathon 2020 so lets start

![image](../../../assets/images/hackershala-ctf-1.png){: .imagePopup}

we got two challenges to solve so first is the web challenge and in second challenge we had to do privilege escalation to get the flag and both challenges were related to each others so we had to solve web challenge first.

# Challenge-1

We got a URL and after opening the URL I got the apache default page.

![image](../../../assets/images/hackershala-ctf-2.png){: .imagePopup}


so  I did directory search  and I got one valid directory - `/adminmaster`

![image](../../../assets/images/hackershala-ctf-3.png){: .imagePopup}

and adminmaster has nothing more than a static web page and when I saw the source code of that website I got the password but the password was encrypted to protect it from hackers but still I hacked it  😉

![image](../../../assets/images/hackershala-ctf-4.png){: .imagePopup}

so now I had an encrypted password and I had to decrypt it so I tried different methods to decrypt it but I failed and then I through that maybe this is cipher encryption so I tried to decrypt it from cipher and luckily I was able to decrypt it as it was using ROT47 encryption and I got the password `h4cK3rsh414`

![image](../../../assets/images/hackershala-ctf-5.png){: .imagePopup}

`Hi webmaster, I am giving you the password, but don't forget it again. I have encoded it just to ensure safety from those cunning hackers out there!`

and after reading the above line I was able to guess the username : **webmaster** and I got the password : **h4cK3rsh414**.

# Challenge-2

So for this challenge we got a terminal where we had to do our task so I run command netdiscover to find the IP addresses on my network and I found two IP address first IP was of my network and another one was the target IP and then I did port scanning for that IP and only one port was open.

![image](../../../assets/images/hackershala-ctf-6.png){: .imagePopup}

Now port 22 is open for 192.168.147.3 so I logged in into ssh using username : webmaster and password : h4cK3rsh414. After logged in through ssh I found a file (magicfile.txt) that contain some encrypted strings.

```
==jnzuTMugzMbEJLeMTMmS2n
z5JqxM2ouM2ox5Jqu92LuyJqzyUM1gJL
55zMxIKLgE2o1AKL112LxAao5SJqbAJq4yJL
==NMiS2LiEJLi12L1S2owMTMecJL
==tMxSJozc2ogS2ocMJnu9zMc9TMuAzqbc3n
wAKLi1zMxS2ow9znxSJni12LdyTMfyJL
=DzneAKLfcTMmkJL
==tGCyRHASRFQ9IDZSRFGWIEYAHDV9IEFS0KI9HJsAUquW3Mh92D
ucTMfg2pdSzMdEJLfczMfSznxk2nzcTMugTozc2nxkJLdMToeAKL
```

and then I tried to decode it using base64 from end to start but no luck 🙁

so now its time to do privilege escalation so I ran `sudo -l` to get the sudoers permission for current users and I found this

![image](../../../assets/images/hackershala-ctf-7.png){: .imagePopup}

the above image means that we can run `sudo /home/webmaster/shell` without password so I ran this command and got the root shell.

![image](../../../assets/images/hackershala-ctf-8.png){: .imagePopup}

Now its time to go in /root directory and in this directory I found python file named as decryptor.py and after reading the code of that file I got to know that this script takes string as an input and then decode that string from rot13 first and then again decode it from base64 so I ran that script with strings in magicfile.txt as an input and I got this message.

![image](../../../assets/images/hackershala-ctf-9.png){: .imagePopup}

and here both the challenges were solved successfully.