---
author: rn.wolfe
date: 2017-10-13 21:37:32+00:00
draft: false
title: Allowing Mobile App Stores through a WLC Redirect ACL
type: post
url: /2017/10/13/allowing-app-stores-using-redirect-acl/
categories:
  - Cisco ISE
  - Troubleshooting
tags:
  - acl
  - airwatch
  - app store
  - apple
  - byod
  - Cisco ISE
  - google
  - google play
  - mdm
  - mobileiron
  - redirect
  - wlc
---

Cisco ISE provides the ability to [redirect users through an MDM workflow to assist in the on-boarding of mobile devices](https://www.cisco.com/c/en/us/td/docs/security/ise/2-2/admin_guide/b_ise_admin_guide_22/b_ise_admin_guide_22_chapter_01000.html#ID434). Using integration of MDMs like MobileIron or AirWatch, you can allow registered and compliant devices onto your network, while [automatically facilitating MDM enrollment](https://www.cisco.com/c/en/us/td/docs/security/ise/2-2/admin_guide/b_ise_admin_guide_22/b_ise_admin_guide_22_chapter_01000.html#ID518) for other devices. While the authorization policy for these workflows is relatively straightforward, the specification of traffic flows for redirection to the MDM portal can be somewhat challenging.

This challenge is primarily due to the need to access the Apple App Store or the Google Play Store to download a required MDM applications during on-boarding. While we may be okay with allowing people out to the internet, we still need to make sure we are capturing and redirecting web requests to the MDM enrollment portal. This is ultimately very similar to central web authentication redirect, but with more access requirements. If users can't download the needed app(s) during the on-boarding process, they will likely not be able to get fully on-boarding.

As this pretty much always takes place on wireless, the redirect ACL is limited to the feature sets of the Cisco Wireless LAN Controller platform. If we were stuck with IP-based filtering, it would be a full time job to hunt down all possible IP addresses. To make matters more difficult, we can't just restrict internet usage to certain ports because the app stores rely on TCP 80 and 443, as well as other ports, for access to their servers. Enter DNS-based Access Control Lists on the Cisco WLC.

DNS-based ACLs allow us to specify URLs in our standard access control lists as additional permit statements (aka whitelisted URLs) in addition to standard IP-based filtering. Admittedly, the feature took a bit to figure out. Initially, I was testing by applying an ACL to the WLAN using the Advanced tab settings and testing access which proved to be pretty much useless. Success finally became an option when applying it specifically as a redirect ACL via ISE authorization permissions. At that point, anything that was specified as a permit on the IP-based filter or a URL on the list would bypass the redirection.

This is a good point to differentiate between a DNS-based ACL and a URL ACL (which seems to be available beginning with AireOS 8.3):

- A **URL ACL** is a list of URLs that can be configured to act as a Whitelist \_or a _Blacklist, and then be applied to an interface, WLAN, or Local Policy configuration. More about this feature can be [found in this document](https://www.cisco.com/c/en/us/td/docs/wireless/controller/technotes/8-4/b_URL_ACL_Enhanced_Deployment_Guide.html#concept_197B3A8E799B4C2AAD4B65BF717892F6). This feature seems to mostly be for enforcement of permitting or denying access to a list of URLs. This is implemented in the **Security Tab > Access Control Lists > URL ACLs**.
- A **DNS-based ACL **is an ACL that also uses DNS lookups to permit traffic to dynamic IP addresses based on FQDN. This is implemented in the **Security Tab > Access Control Lists > Access Control Lists** using the **Add-Remove URL** setting in the blue drop down arrow as seen below:

{{< smallimg src="/images/url-redir-wlc-acls.png" alt="WLC Access Control Lists" width="70%">}}

The DNS-based ACL feature seems to have been implemented specifically for use with the redirect functionality according to the [feature information and restrictions in the documentation](http://concept_aeedd6d25578413784092b48a4636163/), which states that it can only be used with the URL Redirect feature.

<blockquote>With DNS-based ACLs, the client when in registration phase is allowed to connect to the configured URLs. The Cisco WLC is configured with the ACL name and that is returned by the AAA server for pre-authentication ACL to be applied. If the ACL name is returned by the AAA server, then the ACL is applied to the client for web-redirection.

<br /><br />At the client authentication phase, the ISE server returns the pre-authentication ACL (url-redirect-acl). The DNS snooping is performed on the AP for each client until the registration is complete and the client is in SUPPLICANT PROVISIONING state. When the ACL configured with the URLs is received on the Cisco WLC, the CAPWAP payload is sent to the AP enabling DNS snooping on the client and the URLs to be snooped.</blockquote>

This DNS-based ACL feature was introduced in AireOS 8.0, and restrictions to exist which should be reviewed for your code version in the administrative guide (link for 8.3 above). For instance, the documentation for 8.3 states that only up to 10 URLs can be configured; however, I found this to be true in 8.0, but not in 8.2+ which allowed me to enter up to 20 URLs. This is good because for the Apple App Store and Google Play to work, we got up to 19 URLs. This may just be a documentation bug, but I would generally keep the URL list as short as possible.

URL ACLs, on the other hand, seem to have been introduced in 8.3. This feature may be able to be used for redirection; however, I did not test with it as I found a solution using DNS-based ACLs instead.

### Required ACL Filters

#### Apple App Store Filters

Continuing with the DNS-based ACL for redirect during MDM on-boarding, I found the following filters allowed full access to the Apple App Store:

- Permitted IPs:

  - 23.0.0.0/8
  - 65.158.0.0/16

- Permitted URLs:
  - albert.apple.com
  - gs.apple.com
  - itunes.apple.com
  - ax.itunes.apple.com - www.apple.com

The IPs could use a bit further explanation. I started with just the URLs and had partial success (apple.com would load with stripped styling, and the app store still didn't work). So, I did some digging into the DNS of apple.com to find that it was served by a Akamai Edge CDN. I tried being a bit more restrictive, but ended up using the /8 that all of the IPs belonged to: 23.0.0.0/8. I did identify that this allowed some other sites that are served by the Akamai CDN. I noticed www.cisco.com and usaa.com we also permitted. I will look to do more restrictive ranges, if possible, in the future. But, for now, this suits my requirements as the WLAN itself is still authenticated. It is restrictive enough to prevent users from using the network for bandwidth intensive purposes (music, video, etc.). The 65.158.0.0/16 was added because I saw requests being dropped in the network while the App Store was suffering from crippled functionality. I attempted to find an FQDN that could be permitted, but the reverse lookups gave nothing and I was stuck with this range for now.

#### Google Play Store Filters

The following filters were needed to provide access to the Google Play Store:

- Permitted URLs:
  - android.clients.google.com
  - google.com
  - ggpht.com
  - play.googleapis.com
  - gvt1.com
  - www.googleapis.com
  - accounts.youtube.com
  - gstatic.com
  - .googleapis.com
  - .appspot.com
  - gggpht.com
  - android.pool.ntp.org
  - market.android.com
  - .google.co

No additional IPs were required.

Some of these may not be required, but they all got the job done for my testing. And, with all of the different flavors of Android out there, I figured - the more permissive, the better.

#### A few considerations

With the above filters in place for the redirect ACL, I did some testing on an iPhone 7 Plus running IOS 11.1 Developer Beta 2 and a Samsung Galaxy S7 running Android 7.0. During my testing I found the following:

- Apps that were permitted:

  - iMessage
  - Google Maps
  - Google Hangouts
  - Gmail (Partially) - Native Mail App to Gmail Account successfully sent mail - Google Inbox on iOS could not send mail. - Google Drive file access

- Apps that were not permitted:
  - YouTube (a video list was retrieved, but no ads, thumbnails, or actual videos were shown or played).
  - Google Duo - Apple FaceTime

### Last but not least

Of course, beyond the app store filters, you will need to permit access to your MDM resources, DHCP, DNS, and other required services to facilitate device on-boarding. Below is my final Redirect ACL where 10.202.192.0/24 is my servers submit that can represent access to my ISE servers, MDM servers, etc.

![MDM Onboarding Redirect ACL](/images/url-redir-mdm-onboard-acl.png)

![MDM Onboarding Redirect URL List](/images/url-redir-mdm-onboard-url-list.png)

This post also does not cover the configuration of MDM integration in ISE, as documentation exists (and is referenced earlier in this post). I also don't have an MDM instance in my lab to go through the full configuration outside of a production test environment. The above ACL would be applied as the Redirect ACL in a CWA MDM authorization profile.

### Further Updates

When I inevitably continue to fiddle with and further refine this, I will post updates to this post and announce via Twitter ([@somewolfe](https://twitter.com/somewolfe)).
