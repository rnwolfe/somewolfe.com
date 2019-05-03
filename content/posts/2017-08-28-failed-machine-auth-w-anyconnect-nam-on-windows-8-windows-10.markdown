---
author: rn.wolfe
date: 2017-08-28 21:00:13+00:00
draft: false
title: Failed Machine Authentication with AnyConnect NAM on Windows 8+
type: post
url: /2017/08/28/failed-machine-auth-w-anyconnect-nam-on-windows-8-windows-10/
categories:
- Cisco ISE
- Troubleshooting
tags:
- AnyConnect
- authentication
- Cisco ISE
- eap chaining
- eap-fast
- failed
- machine
- nam
- peap
- windows 10
- windows 8
---

Starting with Windows 8, Microsoft changed a default security setting that only allows third party software to access the machine's domain password in an **encrypted** format. This results in a third party supplicant sending an encrypted password string to the domain that then compares it against an unencrypted password string. The authentication then fails with the error "Machine authentication against Active Directory failed due to wrong password." If you look in the authentication steps, you will see the `24344 RPC Logon request failed - STATUS_WRONG_PASSWORD, ERROR_INVALID_PASSWORD,pclthp10156.domain.com` message. This _traditionally_ indicates that the machine account has expired, or the machine password has otherwise fallen out of synchronization with the domain. It could be easily be fixed by rejoining the machine to the domain.

In this case, however, it is due to the fact that ISE sends the provided password string to the domain unchanged, and when the domain compares the encrypted password string to the its copy of the password that is unencrypted, it does not match up. The string you provided does not match up with the string I have? That's an invalid password! Well, that may technically be true, but it's also misleading.

The issue is that the AnyConnect Network Access Manager (NAM) is a third party agent, so it's using that encrypted password format. A bug covering this behavior can be [found here](https://bst.cloudapps.cisco.com/bugsearch/bug/CSCuc13862). The only workaround for this right now is a registry edit that allows third party agents to access the password in an unencrypted format. The following registry edit will make the change:

1. Open `regedit.exe`.
2. Go to `HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Lsa`.
3. Add a new DWORD(32-bit) Value with the name `LsaAllowReturningUnencryptedSecrets`.
4. Modify the new key to set it to `1`.
5. Click OK.

Unfortunately, the bug is listed as "Fixed" despite it still being a problem. It may not be Cisco's problem, but what's the real resolution besides allowing all third party applications from accessing this value in it's native hash format? This remains unclear. As more and more enterprises move to Window 10, this will likely cause more problems. Another resolution is to use the Windows native supplicant wherever possible, unless your use case explicitly requires the AnyConnect NAM as a supplicant, e.g. EAP Chaining, advanced management of wired and wireless networks, etc. The native supplicant will correctly handle the password without a registry change.

That being said, anyone using the NAM to perform a machine-based authentication in any way -- such as a PEAP, or a PEAP inner method for EAP-FAST -- you are going to want a plan for this moving forward. At this time you only have two options: native supplicant, or registry change. Hopefully, a behavior change to the NAM will fix this in the future, but with this existing since AnyConnect 3.x and, with 4.5 just having been released, I don't see it as a near term fix. Though, to be fair, this could very likely be because of the change being something that the NAM can't work around and Cisco is stuck in this situation.
