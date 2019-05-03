---
author: rn.wolfe
date: 2018-01-23 12:28:30+00:00
draft: false
title: Setting up an iPerf3 Server on a Raspberry Pi
type: post
url: /2018/01/23/setting-up-an-iperf3-server-on-a-raspberry-pi/
categories:
- How To
tags:
- iperf
- iperf3
- networking
- raspberry pi
- rpi
- speed tests
---

I was recently battling for server time when doing to internet-based performance testing against one of the [publicly listed iperf3 servers](https://iperf.fr/iperf-servers.php). Unfortunately, iperf3 only supports one test at a time. This makes sense in order to provide full resources to an individual test for reliable results. In any case, I didn't want to wait when I'm smack in the middle of some heart throbbing troubleshooting. So, I decided to set my own up from my home and let internet traffic in to use it.

#### Updating the Raspberry Pi
First things first, I had barely touched my RPi in a while, so I needed to update Raspbian to the most recent distro -- Pixel. This was easy enough, though time consuming. The RPi package servers apparently don't download all that quickly. It took around a couple of hours all in all. [This is a straightforward guide for it](https://www.cnet.com/how-to/update-your-raspberry-pi-to-raspbian-pixel-for-fancy-new-features/). But, it essentially comes down to:

    sudo apt-get update
    sudo apt-get dist-upgrade

The article also suggests these for some beautification and common use packages:

    sudo apt-get install -y rpi-chromium-mods
    sudo apt-get install -y python-sense-emu python3-sense-emu
    sudo apt-get install -y python-sense-emu-doc realvnc-vnc-viewer

#### Install iperf3
Next, let's get to iperf3. This is pretty simple to install, as most linux packages are.

    sudo apt-get update
    sudo apt-get install iperf3 -y

Now, let's set it to boot at system start:

    sudo nano /etc/rc.local

Add this line somewhere before the exit 0.

    /usr/bin/iperf3 -s &

Since this file could have been customized on your RPi already, it could look like anything so you should add it where appropriate. Mine was still default so pretty much anywhere would do except for after the exit is called. If you need to add any additional flags to your server instance, do it here.

    reboot

    ps -A | grep iperf
    561 ? 00:00:00 iperf3

#### Network Changes
If you are running behind a firewall, or ISP-provided modem. You will have to setup some NATs and security policy to address the needed iperf flows. In my case, it was inbound TCP/5201 and UDP/5201; however, you can customize these ports if needed. It would require some additional flags to be specified on the server and client to change the default port.

The only thing that may be non-standard is UDP flows coming out from the iperf server after a client initiates a UDP test. The server seems to send a separate UDP flow on some ephemeral ports. This seems similar to the behavior of active mode FTP, but the firewall doesn't inspect and compensate for it. So, you may need to rules to allow the outbound UDP flows separately from the rest. For what it's worth, I'm using a Palo Alto Networks firewall and it doesn't seem to match the iperf App-ID on the initial outbound UDP flow, but it does for everything else. So, consider that if you're having issues with your App-based rule approach.

#### Test
Of course, final step is to test the thing out. So, get iperf3 on a client. This can be Windows, macOS, Linux, etc. Distros are [available here](https://iperf.fr/iperf-download.php) for all OS types.

After that, you'll want to run the following command to test it out using TCP. This example is on Windows.

    iperf3.exe -c [your-server]
    Connecting to host [your-server], port 5201
    [  4] local 10.202.192.100 port 61586 connected to [your-server-ip] port 5201
    [ ID] Interval           Transfer     Bandwidth
    [  4]   0.00-1.00   sec  8.00 MBytes  67.1 Mbits/sec
    [  4]   1.00-2.00   sec  8.00 MBytes  67.1 Mbits/sec
    [  4]   2.00-3.00   sec  8.62 MBytes  72.4 Mbits/sec
    [  4]   3.00-4.00   sec  9.12 MBytes  76.5 Mbits/sec
    [  4]   4.00-5.00   sec  9.25 MBytes  77.3 Mbits/sec
    [  4]   5.00-6.00   sec  9.50 MBytes  79.8 Mbits/sec
    [  4]   6.00-7.00   sec  9.38 MBytes  78.5 Mbits/sec
    [  4]   7.00-8.01   sec  9.50 MBytes  79.5 Mbits/sec
    [  4]   8.01-9.00   sec  9.38 MBytes  79.0 Mbits/sec
    [  4]   9.00-10.00  sec  9.50 MBytes  79.5 Mbits/sec
    - - - - - - - - - - - - - - - - - - - - - - - - -
    [ ID] Interval           Transfer     Bandwidth
    [  4]   0.00-10.00  sec  90.2 MBytes  75.7 Mbits/sec                  sender
    [  4]   0.00-10.00  sec  90.1 MBytes  75.6 Mbits/sec                  receiver

    iperf Done.

Add a flag for UDP, and another to set target bandwidth over 1Mbps which is default in iperf3's setup.

    iperf3.exe -c [your-server] -u -b 500000000 -f m
    Connecting to host [your-server], port 5201
    [  4] local 10.202.192.100 port 65527 connected to [your-server-ip] port 5201
    [ ID] Interval           Transfer     Bandwidth       Total Datagrams
    [  4]   0.00-1.00   sec  55.4 MBytes   464 Mbits/sec  7095
    [  4]   1.00-2.00   sec  59.9 MBytes   503 Mbits/sec  7670
    [  4]   2.00-3.00   sec  59.7 MBytes   500 Mbits/sec  7636
    [  4]   3.00-4.00   sec  59.1 MBytes   495 Mbits/sec  7567
    [  4]   4.00-5.00   sec  58.7 MBytes   495 Mbits/sec  7510
    [  4]   5.00-6.00   sec  60.7 MBytes   509 Mbits/sec  7774
    [  4]   6.00-7.00   sec  60.3 MBytes   506 Mbits/sec  7717
    [  4]   7.00-8.00   sec  58.1 MBytes   487 Mbits/sec  7436
    [  4]   8.00-9.00   sec  62.9 MBytes   527 Mbits/sec  8048
    [  4]   9.00-10.00  sec  57.0 MBytes   478 Mbits/sec  7291
    - - - - - - - - - - - - - - - - - - - - - - - - -
    [ ID] Interval           Transfer     Bandwidth       Jitter    Lost/Total Datag
    rams
    [  4]   0.00-10.00  sec   592 MBytes   496 Mbits/sec  2.294 ms  75210/75740 (99%)
    [  4] Sent 75740 datagrams

    iperf Done.

#### Customization
I recommend checking out the [iperf documentation](https://iperf.fr/iperf-doc.php) for customization options. Additionally, it shouldn't be too hard to cater this any other Linux distro. Just tweak the package installation and startup config as needed per your distro flavor.

#### Issues?
Check out [open issues](https://github.com/esnet/iperf/issues) on the [Github project](https://github.com/esnet/iperf) or do some google-fu if you run into any problems. Also, make sure your network/security is configured to supported the needed flows.

I ran into one problem with UDP testing that came down to having multiple NICs available on the pi. In my case, it was the eth0 and wlan0 interfaces. I originally had requests coming into the wlan0 interface (whoops) and the outgoing UDP flow would go out of eth0 interface. This seemed to break the test in its own right, but would also have implications on firewall policy and NAT configurations if you're behind a firewall.

I flipped my setup to direct requests to the eth0 IP address, and the issue didn't present itself again. So, there must be some built in NIC preferences that were messing with things. I did [add some info to an existing bug addressing this on the iperf github project](https://github.com/esnet/iperf/issues/637#issuecomment-359676006), so hopefully it will get resolved at something.
