---
author: rn.wolfe
date: 2016-09-27 00:54:09+00:00
draft: false
title: PAN Virtual Wire with Cisco HA Pair
type: post
url: /2016/09/27/pan-virtual-wire-with-cisco-ha-pair/
categories:
- Tech
- Troubleshooting
tags:
- asa
- cisco
- configure
- ha
- palo alto networks
- panfw
- tcp
- vwire
---

Recently, I was setting up a Palo Alto Networks Next Generation firewall to perform URL Filtering and Threat Prevention on internet-bound traffic for a client. The firewall was deployed into the environment using a network tap to get traffic through the device. The tap sat between a pair of ASAs and some backend IPS appliances. This may seem a bit redundant, but the primary purpose of the PAN was to perform URL Filtering, and the threat prevention as simply a nice addition. Additionally, this is a fairly common first step in the deployment of a PAN. It works well being deployed as a platform in a certain capacity and then turning on additional features down the line once the administrator(s) begin to become familiar with the interface.

To get started, I had two virtual wires configured. One vwire for the internet link going to the primary ASA and another for the internet link going to the secondary ASA.

As those who are familiar with the ASA may know, in an active/standby HA pair, the standby firewall is not sending any traffic despite the interfaces actually being active. They use standby IP addressing, while the active IP is kept on the firewall that is.. well, active. All routing peers and traffic headed to the actual firewall use the active IPs. When the firewall fails over, the IP addressing is swapped and the secondary firewall takes over forwarding data plane traffic. In addition, the standby interfaces and active interfaces exchange heartbeats to ensure that the respective interface on the opposing firewall is still up and running. In the event that the interface or path is down for some reason, it will trigger a failover assuming your fail thresholds are met.

The challenge I ran into was that the heartbeats didn't seem to be getting through successfully after the PAN was put inline behind the inside interface of each firewall. Now, at this point, the security policies were simply permitting all traffic and no drops were seen in the traffic logs of the PAN. Having seen this before, I took to the packet capture functionality. Traffic that is dropped due to reasons other than the security policy of security profiles will not show up in the traffic logs. I set up the packet captures to capture all traffic on the vwire, and set the drop stage to save to a file. When I viewed the capture, I only saw what seemed to be one type of traffic being dropped: IP Protocol 105, SCPS. It turns out that this is the IP protocol that [Cisco ASAs use for heartbeat exchange](https://supportforums.cisco.com/discussion/10934686/asa5520-keepalive-ip-protocol-105-scsp).

![screen-shot-2016-09-26-at-8-11-01-pm](https://www.somewolfe.com/wp-content/uploads/2016/09/Screen-Shot-2016-09-26-at-8.11.01-PM-1-1024x280.png)
Well, that makes a lot of sense given I don't see the heartbeats getting where there need to go.. but why?

As I look at the PAN CLI using the "show counter global filter severity drop" command, I also noticed drops due to invalid interface.

    > show counter global filter severity drop
    Global counters:
    Elapsed time since last sampling: 34.999 seconds
    name                                   value     rate severity  category  aspect    description
    -------------------------------------------------------------------------------
    flow_no_interface                        263        0 drop      flow      parse     Packets dropped: invalid interface
    --truncated--

This indicates that the firewall doesn't know where to send the traffic, which is a bit bizarre to me given it's deployed in virtual wire mode.

I suspected it may have been because the PAN doesn't like IP Protocol 105; however, I sure hoped that wasn't the case as it surely would've presented itself as a big issue to PANW by now. Looking further into the frame, I noticed that the frames were actually coming through as 802.1Q encapsulated traffic with a VLAN tag specified.

![Screen Shot 2016-09-26 at 8.15.11 PM.png](https://www.somewolfe.com/wp-content/uploads/2016/09/Screen-Shot-2016-09-26-at-8.15.11-PM.png)


I went ahead and added the VLAN ID as a Wireshark header, and as you continue to look at the packets, you can see an incremental increase in the VLAN ID. It appears that the ASA starts with VLAN ID 3001 and adds 100 for each interface it is monitoring:

![Screen Shot 2016-09-26 at 8.20.59 PM.png](https://www.somewolfe.com/wp-content/uploads/2016/09/Screen-Shot-2016-09-26-at-8.20.59-PM.png)


Then, it dawns on me, that with a virtual wire you have to specify the VLAN ID that the virtual wire interface will be passing. In this case, it was a layer 3 link, so I had specified "0" for untagged traffic. I changed it to permit 3001, 3101, and 3201 and bingo. Everything was working, failover was back as it should be.

Unfortunately, not everything was going perfectly quite yet. There seemed to be some asymmetric traffic flows which the firewall was not liking. I believe this was partially a result of the failed failover since the interfaces weren’t monitoring correctly on the ASA at first. Additionally, when a failover occurs, since state is maintained for most traffic on the ASA, flows will initially _seem_ like non-established sessions attempting to come through a different interface of the PAN. This results in them being labeled as “non_syn_tcp” meaning the traffic is sending traffic as if the TCP handshake was completed, but it never was (rather, a TCP SYN was never seen by the firewall). Enabling the flow of "non_syn_tcp" flows fixed this issue. Additionally, with this design, I suspect enabling this globally, or per-zone using a Zone Protection Profile, will prevent traffic interruption during a failover event due to the stateful nature of the ASA HA Pair. This may not be an issue with non-stateful HA pairs.

Additionally, I was seeing issues with loading HTTP web pages in particular, while HTTPS remained working. The highest increasing counter at this time was "tcp_drop_out_of_wnd." Admittedly, for this one, I can't really concretely determine why this was causing issues with HTTP traffic (open to feedback on this); however, enabling asymmetric-path bypass did the trick and everything started working.

From the CLI, these can be configured globally using:

    set deviceconfig setting session tcp-reject-non-syn no
    set deviceconfig setting tcp asymmetric-path bypass

To do this with a zone protection profile use the following settings and apply it to the respective zone:

![zpp.png](https://www.somewolfe.com/wp-content/uploads/2016/09/zpp.png)

Note: I do not recommend setting these globally due to the inherent security that is disabled in the stateful engines of the firewall. In this scenario there were actually upstream _and _downstream devices sandwiching the firewall that were performing the same checks (ASA and IPS) that allowed us to modify these setting safely; however, I still used Zone Protection Profiles to be as specific in application as possible just in case the firewall provides dual purpose in the future where those check should be enabled.

Some good reference links:

* [https://live.paloaltonetworks.com/t5/Learning-Articles/Palo-Alto-Networks-TCP-Settings-and-Counters/ta-p/5404](https://live.paloaltonetworks.com/t5/Learning-Articles/Palo-Alto-Networks-TCP-Settings-and-Counters/ta-p/5404)
* [https://live.paloaltonetworks.com/t5/Configuration-Articles/SYN-ACK-Issues-with-Asymmetric-Routing/ta-p/54090](https://live.paloaltonetworks.com/t5/Configuration-Articles/SYN-ACK-Issues-with-Asymmetric-Routing/ta-p/54090)
* [https://live.paloaltonetworks.com/t5/Management-Articles/Packets-are-Dropped-Due-to-TCP-Reassembly/ta-p/57139](https://live.paloaltonetworks.com/t5/Management-Articles/Packets-are-Dropped-Due-to-TCP-Reassembly/ta-p/57139)
* [https://live.paloaltonetworks.com/t5/Management-Articles/Packet-Capture-Debug-Flow-basic-and-Counter-Commands/ta-p/66224](https://live.paloaltonetworks.com/t5/Management-Articles/Packet-Capture-Debug-Flow-basic-and-Counter-Commands/ta-p/66224)

