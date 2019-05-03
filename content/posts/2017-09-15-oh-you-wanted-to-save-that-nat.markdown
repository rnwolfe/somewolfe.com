---
author: rn.wolfe
date: 2017-09-15 02:08:05+00:00
draft: false
title: Oh, you wanted to save that NAT?
type: post
url: /2017/09/14/oh-you-wanted-to-save-that-nat/
categories:
  - Troubleshooting
tags:
  - asa
  - cisco
  - firepower management center
  - firepower threat defense
  - fmc
  - ftd
  - gui
  - nat policy
  - ngfw
---

I recently stumbled across an "undocumented feature" on the Cisco Firepower Threat Defense managed by Firepower Management Center (FMC) that caused quite the frustration. When entering certain parts of the FMC, the "Save" and "Cancel" buttons won't show up in the top right corner. The downside of this, of course, is that I can't save whatever I was working on.

The most consistent occurrence of this was the NAT Policy. When managing the NAT Policy, about 9/10 times, or probably more, the "Save" option just wouldn't show up. If you inspect the HTML, you can find it; however, I couldn't forcibly get it to show up. The only resolution to this, as far as I could find, is to leave the page and delete you browser's cached files (sometimes multiple times). Afterwards, go back into the policy and ensure that the buttons are present before proceeding. Hopefully, you can catch this before getting too far into your changes.

You can see the area I'm talking about below enclosed in red:

![NAT Policy Buttons](/images/NAT-Policy-Save-Buttons.png)

I, personally, saw this in 6.2.1 and 6.2.2. I've also found [this bug](https://bst.cloudapps.cisco.com/bugsearch/bug/CSCvd78875/?referring_site=bugquickviewredir) which suggests it was present on same pages of 6.1.0, as well. I've also seen some Cisco presentations recently that call this out as something to be aware of. So, they are aware, but it is not yet fixed. Unfortunately, I am not sure when it will be fixed. Hopefully soon!
