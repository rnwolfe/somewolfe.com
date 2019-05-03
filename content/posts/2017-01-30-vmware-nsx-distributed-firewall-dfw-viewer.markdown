---
author: rn.wolfe
date: 2017-01-30 00:17:45+00:00
draft: false
title: VMware NSX Distributed Firewall (DFW) Viewer
type: post
url: /2017/01/30/vmware-nsx-distributed-firewall-dfw-viewer/
categories:
- Development
- VMware NSX
tags:
- app
- dfw
- distributed firewall
- github
- nsx
- nsx-dfw-viewer
- vmware
---

I've spoken to a few people who use VMWare NSX with the Distributed Firewall (DFW). Most of them, myself included, had some gripes about the NSX interface. While it's web-driven interface is an improvement on many firewall managers, it left something to be desired. At many points in its use, it's easy to find yourself falling deep down a menu-clicking hole while trying to check the contents of a security group, or modify some object that you're using in the policy. And, of course, it all takes time to load, navigate, and, ultimately, end up in the right place.

For some reason, this just isn't as readily accessible as one would hope. In general, the viewing and use of the policy is pretty manageable. It is a bit clunky often times and quite slow. I can certainly see it becoming easy to quickly get to a point where the size of the policy is completely unmanageable -- even with the use of sections. Ultimately, I decided that with the NSX API, I could put something together that was more snappy, useable, and solved some specific issues some colleagues were running into.

I, quite creatively, called it the [NSX DFW Viewer](https://github.com/rnwolfe/nsx-dfw-viewer). Okay, maybe it's not all that creative. However, it's a responsive web application that allows for viewing the DFW policy, and performing filtering using a simple search field. The GitHub repository has some good information provided on it to get you started.

{{< figure src="https://www.somewolfe.com/wp-content/uploads/2017/01/img_0003.png" title="NSX DFW Viewer Web App">}}

Ultimately, however, you just put the repository on a web server running PHP and you're off to the races. The only thing you need to update is the `$nsx_host` variable in the `ajax/init.php` file to reflect your NSX Manager's IP address. After that, you will be asked for credentials, on a per-session basis, when you load the page. This can be seen below, along with a couple of attempts to provide unauthorized credentials.

{{< youtube LTywf7rD4HM >}}

As you can see, it simply uses the browser-based authentication prompt using PHP's `$_SERVER['PHP_AUTH_USER']` and `$_SERVER['PHP_AUTH_PASS']` variables. It then performs a REST API call using those credentials looking for a HTTP 200 OK before granting access. If it doesn't get one, access is denied. I had previously simply used a variable in [code]init.php[/code] to store the username/password and then base64 encode it to perform HTTP Basic Auth; however, this was inherently insecure as the credentials were stored locally. Ultimately, HTTPS should be used on the server to secure the HTTP transport used to send the credentials. The REST API calls are performed over HTTPS.

Once you're in the app, you can see your firewall policy broken down by section (specifically, Layer 3 Section). By default, the sections are collapsed to allow an at-a-glimpse view of longer policies. If you have empty sections, they can be hidden using the available checkbox. Here's a simple demo:

{{< youtube ZjaCMp_Gf0Y >}}

I had some internal debate regarding the search capabilities. I wanted it to be simply implemented but still effective. I landed on it taking whatever text is entered in the search field and filtering out any section or rule information that does not contain the query. In other words, if I were to enter "quarantine," only a section containing "quarantine" or a rule containing "quarantine" in ANY of its fields would be displayed. This includes any fields that are not actually displayed such as rule ID, section ID, or object ID. This, ultimately, allows you to filter by a unique identifier, if needed. At this time, I found this to be the most useful way to implement this.

The project page is here: [https://github.com/rnwolfe/nsx-dfw-viewer](https://github.com/rnwolfe/nsx-dfw-viewer). I'm open to any feedback on the search or any part of this! I would really like for people to try it out, and let me know if I can customize it to address any specific issues you would like. Feel free to contribute via GitHub!
