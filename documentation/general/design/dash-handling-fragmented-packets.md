---
title: Handling of Fragments
last update: 02/22/2022
---

# Handling of Fragments

- [Background](#background)
  - [Protocol formats](#protocol-formats)
    - [IP header format](#ip-header-format)
    - [UDP header format](#udp-header-format)
    - [TCP header format](#tcp-header-format)
  - [Fragmentation contextual information](#fragmentation-contextual-information)
- [Issues and Nuances](#issues-and-nuances)
- [PUSH Flag](#push-flag)
- [Do Not Fragment Flag](#do-not-fragment-flag)
- [References](#references)

## Background

### Protocol formats

The information in this section is a quick reminder of the protocol formats.
#### IP header format

The **Internet Protocol** (IP) is designed for use in interconnected systems of
packet-switched computer communication networks. It provides for transmitting
blocks of data called datagrams from sources to destinations, where sources and
destinations are hosts identified by fixed length addresses.  The internet
protocol also provides for **fragmentation** and reassembly of long datagrams,
if necessary, for transmission through *small packet* networks.The figure below
shows the IP header format. For more information, see [RFC
791 - Internet Protocol](https://datatracker.ietf.org/doc/html/rfc791#section-3.1).

<figcaption><b><i>IP header fornat</i></b></figcaption>

![ip-header-format](images/ip-header-format.png)

The options may appear or not in datagrams.  They must be implemented by all IP
modules (host and gateways).  What is optional is their transmission in any
particular datagram, not their implementation. For more information, see [RFC
791 - Internet Protocol](https://datatracker.ietf.org/doc/html/rfc791#section-3.1), page 14.

Notice also when fragmentation occurs, some options are copied, but others remain with the first fragment only. For more information, see [RFC
791 - Internet Protocol](https://datatracker.ietf.org/doc/html/rfc791#section-3.1), page 24.
#### UDP header format

The **User Datagram  Protocol** (UDP) provides a datagram mode in
packet-switched computer communication networks, and in interconnected systems
of such networks. This protocol assumes  that the Internet  Protocol  (IP) is
used as the underlying protocol.

This protocol  provides  a procedure  for application  programs  to send
messages  to other programs  with a minimum  of protocol mechanism.  The
protocol  is transaction oriented, and delivery and duplicate protection are not
guaranteed. The figure below shows the UDP header format. For more information, see [RFC 768 - User Datagram Protocol](https://datatracker.ietf.org/doc/html/rfc768).

<figcaption><b><i>UDP header fornat</i></b></figcaption> 

![udp-header-fromat](images/udp-header-format.png)

#### TCP header format

The **Transmission Control Protocol** (TCP) is intended for use as a highly
reliable host-to-host protocol between hosts in packet-switched computer
communication networks, and in interconnected systems of such networks.

The figure below shows the TCP header format. For more information, see [RFC 793 - Transmission Control Protocol](https://datatracker.ietf.org/doc/html/rfc793#section-3.1).

<figcaption><b><i>TCP header fornat</i></b></figcaption> 

![tcp-header-fromat](images/tcp-header-format.png)

### Fragmentation contextual information

1. After a connection has been identified and processed by normal means it will
result in an **entry** into the **flow table** to **enable fast path
processing** for the duration of the connection.

1. If a subsequent packet arrives that is the start of a fragmented packet, the
**Frag ID** must be used to create a **new temporal flow** that can be
**uniquely identified** by the (**Frag ID**, **DST**, **SRC**) tuple.

1. The temporal flow should be **maintained until the last fragment arrives** or
the flow is **aged**.

1. The aging of a temporal flow **could be different than the connection**
 itself as the packets are close together and generated over a small period of
 time.

1. It is also possible to have **several frames in flight** and hence there may
be **several temporal flows**, however they can be **uniquely identified** by
the (Frag ID, DST, SRC) tuple as each frame will have a different Frag ID.

1. Although this will result in more flow entries **these flows will likely live
for `usecs/msec`** and can be aged much quicker than a connection.

1. As these flows last for such a short period of time, **any implementation may
want to consider using aging timer** that is much shorter than a connection
time-out.

1. If the connection is closed with the arrival of the **FIN packet** then all
**temporal flows must be closed** as well.

## Issues and Nuances

1. A connection that is reset should also result in the related
    connection flow including the temporal flows being closed.

    - 10-15% of all connections are ended with **RST** and **NOT FIN**.
    - Once a RST is set the only response to packets is RST.
    - There could be packets in flight so you may see another RST.
    - This may be the reason why VFP for example does not close connections
    immediately and waits for at least a round trip delay.
    - If we do not wait it is possible to black hole subsequent RSTs which could
    result in suboptimal connection performance at the end points

      <figcaption><b><i>Connection aborting</i></b></figcaption> 

      ![tcp-connection-abort](images/tcp-connection-abort.svg) 
     

1. Also, if the first packet comes in out of order then the fragments belonging
   to the same packet would be dropped as they **do not contain Layer-4
   information** and it **would not have an entry in the temporal flow table**.

    - Fortunately, as everything coming/going between VMs is **encapsulated
      within VXLAN** or methods, **out of order packets are unlikely** as
      switches/routers do not mis-order frames of a single L4 flow.
    - The encapsulated packet will look like a **single L4 flow** to a
      router/switch. 
    - > **Also, to be clear, Azure does not support out-of-order packets. They
        are dropped primarily due to security concerns with Nov 2018
        FragmentSmack attacks and for the stated reasons above which should make
        them very unlikely in the first place as a natural occurrence.**

        <figcaption><b><i>Connection creation and deletion</i></b></figcaption>

        | Connection Creation  | Connection Deletion  |
        |-------------------|----------------------|
        |![tcp-connection-create](images/tcp-connection-create.svg)|![tcp-connection-delete](images/tcp-connection-delete.svg)| 
        

1. Questions may remain on whether to create a flow on the **SYN** regardless of
   the three-way handshake **SYN/SYN-ACK/ACK**.

   - There seems to be no reason to wait for the three way handshake
     SYN/SYN-ACK/ACK as we need to allow for the all subsequent control frames
     to flow through to the end points.

   - If the connection is not formed then a **reset** should be received or
     worst case the connection will be **aged out**.

   - There should also be a **SYN** policer that is imposed on an **ENI basis**
     to prevent denial of service.

1. Questions remain on whether to delete a flow on the **FIN**. 

   - For closing we need to keep the flow alive until we see both directions
     closed.

   - This means that **state must be kept** to transition to **FIN/ACK/FIN/ACK**
     to close the connection in both directions or worst case the connection
     will be aged out.

> [!WARNING] The behavior model needs to be updated to include the above
> behaviors so that standard IP conformance testers can validate the proper
> operation and so that fuzzing can also be applied looking for holes in the
> implementation.

## PUSH Flag

1. Because the appliances are not reassembling frames nor terminating user
   frames there is no need to provide any action whether the **PUSH flag** is
   set or not.

## Do Not Fragment Flag

1. Although it is possible for an appliance to fragment packets we see no useful
    reason to do so. If the end points need to fragment, it is **out of the scope
    of bump-in-wire services** providing policy enforcement and switching.

1. In general **DASH is not covering use cases that require re-assembly of frames**
    as the primary purpose is to be transparent to the end points.

## References 

- [RFC 791 - Internet Protocol](https://datatracker.ietf.org/doc/html/rfc791)
- [RFC 768 - User Datagram Protocol](https://datatracker.ietf.org/doc/html/rfc768)
- [RFC 793 - Transmission Control Protocol](https://datatracker.ietf.org/doc/html/rfc793)
