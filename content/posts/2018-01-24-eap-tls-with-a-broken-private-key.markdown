---
author: rn.wolfe
date: 2018-01-24 13:33:50+00:00
draft: false
title: EAP-TLS with a Broken Private Key
type: post
url: /2018/01/24/eap-tls-with-a-broken-private-key/
featuredImage: /images/broken-lock.jpeg
categories:
  - Cisco ISE
  - Troubleshooting
tags:
  - 802.1X
  - aaa
  - AnyConnect
  - certificates
  - cisco
  - Cisco ISE
  - EAP
  - eap-tls
  - error
  - ise
  - native
  - radius
  - supplicant
  - troubleshooting
  - windows
  - windows 10
  - windows 7
---

When going through a standard EAP-TLS deployment recently, a seemingly new problem reared its head. We had tested it out and validated EAP-TLS on a couple of laptops using the Microsoft Native Supplicant. All of the ISE policies were good to go, and everything was working as expected.

Then came along a new laptop. This one had other plans. We plug it in and get to authenticatin'.

#### Troubleshooting

Well, it didn't seem to want to send many 802.1X requests, and the port kept falling back to MAB authentication. We validated the client's native supplicant configuration to ensure that the GPO got everything correct. All seemed to be in good order. We couldn't modify any of the administrator-level settings at this point, so we decided to move on instead of play with some of the settings.

We disabled MAB to see the problem more clearly. Most of the requests were seen on ISE using the host/anonymous identity. This clued us in a bit to the issue since that identity is the default unprotected identity string for EAP-TLS. Something was happening prior to gleaning the actual identity from the machine's certificate.

We check the machine's certificate store and see the lonely certificate. It's purposed for machine authentication as expected. We then check the trust of the ISE server certificate. That didn't seem to be the issue given how it was presenting; however, never hurts to check. The chain was present and certificate deemed valid.

Checking out the ISE live authentications, I start to see a pattern of failures between the "host/anonymous" attempts. Actual certificate identities were coming through very sparingly, though they were still failing. The error was descriptive. Unfortunately, that doesn't necessarily mean helpful.

<blockquote>Supplicant stopped responding to ISE after sending it the first EAP-TLS message</blockquote>

Okay, so that means something is happening after the client sends its initial EAP-TLS message. That's interesting. What would make that occur? The client had a good certificate, server certificate was valid, and supplicant settings were solid. Nothing seemed out of order. On to the next level of troubleshooting. Debugs.

Luckily, there was only one port being authenticated on this test switch for the time being. This made debugging a bit more digestible given the chattiness of the RADIUS, EAP, and AAA debugs. I started with the RADIUS debugs. We knew RADIUS requests were getting to ISE, so this meant that at least some level of EAP communication was happening between the client and the switch. We could see a RADIUS request being sent to ISE for the client, and ISE responding. Then we saw retransmissions of the same response from ISE.

Moving on to enabling EAP debugs, the above ISE error was only confirmed. We saw one inbound EAP frame and then a response after ISE returned the RADIUS response. The client never responded.We did see one uncommon (read as: I'd never seen it before) EAP debug message.

<blockquote>EAP-TLS-ERROR: Invalid Directive (7)</blockquote>

Some google-fu didn't exactly tell me what this meant, so I couldn't rule it out as a clue, but it didn't point me to anywhere useful, either. Perhaps ISE was responding with something that the particular client didn't like? Cue Wireshark. Comparing the troubled workstation to a working one revealed the same EAP traffic up until the point the troubled workstation stopped responding. The contents of the EAP frames were identical with the exception of the retransmissions once the client stopped working. Theory defeated.

Okay, so we knew that the supplicant received an EAP response from ISE and was not sending its own reply. Let's check out the Wired AutoConfig log in the Event Viewer. Nothing out of the ordinary. We see the communication and failures, but nothing out pointing at a problem. Perhaps something in the certificate processing was having an issue. Maybe this client had some crypto algorithm differences and wasn't able to process the server's certificate?

Headed over to the CAPI2 log in the Windows Event Viewer would allow us to check out the certificate operations on the machine and maybe identify something. Unfortunately, this log is not on by default and we weren't able to turn it on with our user permissions. Perhaps this is the universe telling us this may be where the problem lies. Perhaps not.

Fiddling for a while and getting no more enlightened, we decide to install the AnyConnect NAM. It often has a way of doing things the native supplicant fails to do. Not a very good solution to the widespread deployment, but, hopefully, it would help us find the issue. We haggle a system's administrator for her credentials and get things going. NAM installed, basic EAP-TLS profile configured. No dice. Still failing. Still seeing some of the ISE error stating EAP-TLS stopped after the first packet. We hadn't installed the DART bundle (for whatever reason) and the system admin had already scurried away.

Back to the event viewer, it is. Cisco AnyConnect has some logs it adds to the event viewer. That's where we started. We see some error messages and jump right to those. They all refer to some sort of "Internal Error 4" or "Internal Error 204" when calling a function called "acCertLoadPrivateKey()"

<blockquote>acCertLoadPrivateKey(), Internal Error 4 & 204</blockquote>

What dreaded words -- Internal Error. This didn't really tell us what exactly was happening. But, something was apparently going wrong loading the certificate's private key. Could we have finally found the issue?!

#### Resolution

We decided to delete the machine's certificate and re-issue it to test this out. After rustling back the system administrator, we were able to get it removed and a gpupdate / reboot completed. New certificate was enrolled by the client. The client was now successfully authenticating!

The failure was terribly non-graceful to say the least. It didn't give any idea what was actually going on. You would think that Windows handling a certificate with a private key that is somehow inaccessible -- corrupt, missing, who knows -- would throw some sort of error worth investigating. Or, at least, give you a red "X" over the certificate in the store.

It turned out that this was a widespread problem on their existing 802.1X deployment with Cisco ACS that we were migrating from. As they were upgrading Windows clients from Windows 7 to Windows 10, a certain model of laptop starting failing authentication. It seems something in the upgrade process was causing the private key to become broken in some fashion.

So, ultimate fix was to re-enroll all the client authentication certificates via GPO.

Hope this helps some other poor soul out there one day.
