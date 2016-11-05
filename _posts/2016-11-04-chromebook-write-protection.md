---
layout: post
title:  "Disable write-protect on Samsung Chromebook 3 (CELES) XE500C13-K02US"
date:   2016-11-04 22:00:00 -0700
categories: Linux
---
It involves removing a screw on the *back* of the motherboard. Yes, after opening the laptop's case from the back, we have to dismount the screws on the motherboard and flip it over before we can see that actual screw which controls write protection.  

A [description](https://voat.co/v/Linux/comments/1262683) on where the screw is:  
**Anyways, the write protector is on the back of the motherboard. It looks different than all other write protectors I've seen online so I wasn't sure about it when I removed it. It is the only screw with an arrow pointing towards it on the back of the motherboard. There is also a littler metal under the screw.**

A [tutorial](https://www.reddit.com/r/chromeos/comments/3xjp07/i_cant_find_the_writeprotect_screw_on_my_brand/) on that for Samsung Chromebook 2 may be useful.  

Here are some pictures:  

![Celes-screw-1](/assets/celes_screw_1.jpg "celes-screw-1")
This is what it looks like after we open the case. Reds are screws to dismount to flip the motherboard; greens are cables to undo.   

![Celes-screw-2](/assets/celes_screw_2.jpg "celes-screw-2")
The backside of the motherboard; just remove that screw in the yellow square!  

After that, you can run the following commands in the full `shell` of Chrome OS to test if it works:  

```
$ sudo -i
$ flashrom --wp-status
$ flashrom --wp-disable
```