---
layout: post
title: "evpn-route-types-1-4"
date: 2026-03-24
---

# EVPN route types 1 and 4

When you start with EVPN routes everything is pretty straightforward. There is a need (in my case at least) to remember to distinguish between data (usually VXLAN) and control plain and remind yourself, that EVPN is just a BGP, but routes in essence are not that hard:

Route type 2 = "hey, these endpoints are connected to me"  
Route type 3 = "hey, I want BUM traffic for these L2VNIs"  
Route type 5 = "hey, these subnets can be routed via me out of fabric"  

Where is L3vni? There is no reason to advertise it discretely as routes are not filtered before they reach the Leaves anyway. Also they are in route type 5 MPLS field and in route type 2 as additional route info. 

As briefly discussed in a previous post, there is usually a request for a redundant connection to fabric, and it means that we want to protect ourselves from one leaf going down. Software update requires switch reload so connection to only one leaf means guaranteed downtime. MLAG is good but has some caveats. And what if we want some of the schemas such as connection to more than 2 switches or connecting servers to switches 1 and 2, 2 and 3, 1 and 3 at the same time (not that it sounds like a good design)? MLAG supports only 2 fixed switches. Good design accommodates for that, but the possibility of eliminating peer-links and increasing flexibility is not bad. Also it helps with some extreme cases of connecting to two geographically remote Leaves.

## EVPN-MH

That's where you could dive into the world of EVPN-MH. And where route types 1 and 4 make an appearance.  Which are not widely used and many explanations I’ve seen are directly based on RFC 7432 (MPLS-based rfc). You should (and probably already did) read it anyway at least for the packet formats.  

Terminology:  
- ES - Ethernet segment. Pretty much L2 domain, connected to the fabric  
- ESI - Ethernet segment identifier. 10-byte identifier  

Before going into routes it has to be said that in case of all-active multihoming LAG is required per RFC. All said below is applicable for active-active (all-active) topology as active-passive is not really giving us any benefits except redundancy and increased throughput is pretty nice to have.

## Route type 4 - BUM traffic

As for any loop-happy design, the main problem is BUM traffic. For now lets skip the idea that we need to advertise host routes as being behind many switches at the same time and think about making our scheme loop-free.  

The idea is pretty simple - only one Leaf, connected to one L2 domain, should forward BUM traffic into it. More than that, it would be much more efficient if they could use the same approach as PVST - one per VLAN. So all Leaves should select one between themselves. This Leaf is named “designated forwarder” - DF - and does exactly that.

Route type 4 is named Ethernet Segment Route and has the following format

```
              +---------------------------------------+
               |  RD (8 octets)                        |
               +---------------------------------------+
               |Ethernet Segment Identifier (10 octets)|
               +---------------------------------------+
               |  IP Address Length (1 octet)          |
               +---------------------------------------+
               |  Originating Router's IP Address      |
               |          (4 or 16 octets)             |
               +---------------------------------------+
```

Leaf generates routes as soon as it sees ESI not equal to all 0 in its config (and interface is up). Any receiving switch installs the route if it has the same ESI configured (and interface is up). 

### Now for the DF selection part. 

Selection is little different for “VLAN-aware” and “VLAN-based” so lets get how and then why:

RFC-compliant switches do not elect DF, they use formula:

for VLAN-based
DF = V mod N = i, 
V - VLAN ID, 
N - switches with this ESI in segment (in ordered list of the IP addresses)

VLAN-aware bundle service - the numerically lowest VLAN value in that bundle on that ES is used in the modulo function. So only one DF for ES.

There is a delay before computing, but it works like this, so you should make sure that VLAN IDs are the same on ESI-configured trunks.

Cumulus straight up uses DF preference. The switch with the highest DF preference setting becomes the DF. Read docs every time you want to implement something!

Now for the “VLAN-aware” and “VLAN-based”. Terminology is sometimes inconsistent between vendors so we use RFC one for this.

#### VLAN-Based
In a VLAN-based service interface, each VLAN is associated with one bridge domain and one EVI. This one is the simplest one and used extensively by vendors who do not want to dive into VLAN normalization and mapping.

#### VLAN-aware

VLANs have the same EVI but separate bridge tables.

as EVPN is a control plane it does not really care what encapsulation is. RFC uses MPLS and has the name “BGP MPLS-Based Ethernet VPN”. So terminology may be wary, as you’ll see many PE/CE related stuff in there.

Back to VLAN-aware. It is a middle ground between VLAN-Based and VLAN Bundle.
In case of no VLAN translation VLAN tag is preserved. It is more fun in cases where the same VLAN ID is used by different clients (and thus is present in different VNI). To make them unique, translation has to be performed. This translation of VIDs into unique VIDs (either single or double) is referred to as "VID normalization". Please read RFC 7209 and 8214. 

Additional read: [ipspace](https://blog.ipspace.net/2022/10/evpn-vlan-aware-bundle-service)

#### VLAN Bundle
In the VLAN Bundle Service Interface, multiple VLANs share the same bridge table. And MAC addresses have to be unique in one such table so no trunks allowed.
Frames MUST remain tagged with the originating VID. Tag translation is NOT permitted. Not really useful IMO.

So what was that about “Terminology is  sometimes inconsistent ”. Well there is Cumulus:
Cumulus uses a mode named “VLAN-aware” as the only mode that works with EVPN-MH. The second one is “Traditional Bridge” mode which is a Linux subinterface one. But for the purpose of EVPN? “In Cumulus Linux, MAC VRFs use a VLAN-Based Service Interface (RFC 7432). Therefore, MAC VRFs are VLANs and there is a direct one-to-one mapping between layer 2 VNIs and VLANs (VLAN to VNI mapping), which you must specify”. Read docs every time you want to implement something!

Well we have our BUM traffic sorted out. Now it is time to fix host routes, as endpoints are behind not one switch, but a couple, at the same time.

## Route type 1 - aliasing and mass withdrawal and split-horizon and some more

First of all, route type 1 can be 
- per ESI (all link affected)
- per EVI (in usual case of VLAN-Based - only one VLAN is affected)  

Technically you can say that route type 4 is per ESI route.

Route type 1 is named Ethernet Auto-discovery Route and has the following format

```
               +---------------------------------------+
                |  Route Distinguisher (RD) (8 octets)  |
                +---------------------------------------+
                |Ethernet Segment Identifier (10 octets)|
                +---------------------------------------+
                |  Ethernet Tag ID (4 octets)           |
                +---------------------------------------+
                |  MPLS Label (3 octets)                |
                +---------------------------------------+
```

which looks familiar to route type 4 except for the last field. And also “MPLS label” (which is VNI in our usual VXLAN/EVPN network) is a route attribute and not even part of the NLRI prefix.

So basically route type 4 could be used, but route type 1 is smaller (no need for originating IP field) and in case of many-many-many routes it kinda helps.

So now why are there 2 types of route type 1?

### Per ESI  <ESI>

The first reason to have the Per ESI route is the “Split-Horizon label”. This label is used when non-DF switch forwards BUM traffic to DF switch. Pretty much “do not forward this back to ES” label.

There (as usual) isolation between layers and planes stops working. There is no label in VXLAN, which means no split-horizon label. Which should not mean there is a loop possible. 

VXLAN uses a "local bias" mechanism for that which is described in RFC 8365. It is source IP filtering - if a BUM packet is received from a VTEP which has the same ESI - packet is dropped. Also per the same RFC "it is required that the ingress NVE perform replication locally to all directly attached Ethernet segments (regardless of the DF election state)" which breaks previous DF approach. 

There is also new RFC 9746 which discusses MPLS and VXLAN mechanisms for split-horizon. You should read it.

Per ESI route type 1 next purpose is basically mass withdrawal in case of losing connectivity to ES. Link goes down and there is no need to withdraw every route type 2 - every host route - 1 route is enough.

This route also has an “all-active” or “single-active” flag set so the remote switch understands if it should balance traffic between ES-connected switches or not.

This route has its Ethernet Tag ID to max (all F, in the console you’ll see route [1]:[4294967295]:[ESI]).

Pretty useful route all things considered. Could be used for DF elections too.

RD is usually set to RouterID:0 (technically RouterID:0,1,2,3 etc and may vary with different vendor implementations), MPLS label is set to 0.  
You’ll see route type 1 per ESI and route type 4 in “sh bgp route l2vpn evpn“ via the same RD.

### Per EVI <ESI, Ethernet Tag ID>

Ethernet Tag ID: this field identifies a sub-broadcast domain in an ES. If this field is set to all 0s, the EVI contains only one broadcast domain.

You’ll usually see route [1]:[0]:[ESI]  
This route has the same RD as route types 2 and 3 (L2VNI/mac-vrf).

Per EVI route type 1 purpose is Aliasing. So as mentioned above (multiple times) now some hosts are behind multiple Leaves/PEs/switches. What happens if traffic balances from some host to one switch only? Other ones connected to ES should know that it is connected to them too, and remote ones should know that they can forward traffic to the host via any switch connected to that ES.

What should be said 2 pages above is the reminder that route type 2 has an ESI field that is set to all 0s usually. In case of a host found behind ES, the ESI field is set to real ESI (not all 0s) so a remote switch can identify that this advertised host is located behind ESI.

Aliasing label is local to every switch and in examples you’ll see something like 

ESI:  [ESI]
 - PE 1 - aliasing label 1
 - PE 2 - aliasing label 2  

It is basically used by remote PEs to balance traffic to PEs. It has nothing to do with VXLAN so if you want you should read “14.1.2.  All-Active Redundancy Mode” section of the RFC 7432.

That's cool and all but that's MPLS. What about VXLAN? We have VNIs, no labels. Well, the MPLS field is 3octets which is exactly what is needed to include 24bits VNI. Nothing new, same as all the other types.

There is no need for MPLS labels as VXLAN is MAC-in-UDP. Why do we even need this route? Does the switch need an EVI in addition to ESI if we know where ESI is (route type 1 field) and that host is behind determined ESI (route type 2 field)? Route type 2 specifies EVI (Ethernet Tag ID field) so there is no need to relearn it again. 

IMO not really (in case of no misconfigurations) but it reassures that ES-connected interface is configured correctly and remote Leaf actually has VNI connected via said ES. So we stick with an additional safety route. Per RFC “Support of this route is OPTIONAL.” More than that, it is directly said that “The Ethernet A-D per EVI route MUST NOT be used for traffic forwarding by a remote PE until it also receives the associated set of Ethernet A-D per ES routes”.

In the case of cumulus: ESI is associated with NHG (Nexthop group ID) which has a list of RemoteVTEPs. That's Linux specifics and hopefully I’ll dig into that in PBR posts.

## Conclusion

I hoped for this post to be short and to the point, but unfortunately it went into some details that I had to reintroduce myself to. Now it is kinda long and probably riddled with mistakes.

As always I’m happy to see your letters so ping me via email you can find at the about page.
