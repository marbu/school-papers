===========================================
 Configuration and NAT in IPv6 environment
===========================================
:Author: Martin Bukatovic
:Course: Advanced Topics of Linux Administration
:Date: Jul 21 2011

.. contents::

The standardization process of the new IPv6 protocol has produced many different ideas about how the future network should work. 
Some of them were rejected after years of discussion while some others were successfully implemented and accepted among the IPv6 community.
And even now, more than a decade after the beginning of the standardization process,
we are still not sure which way is the right one in many cases.
In this article, I'm going to summarize two essential but still unfinished areas of the new protocol:
network configuration and
possible alternatives to network address translation.


Quick IPv6 overview
===================
Even though the IPv6 shares many common ideas with the IPv4, it is a completely new protocol and can't be considered to be an extension to the older one. Many essential concepts works in a different way and several completelly new features has been introduced.

At first sight the most visible differences compared to the IPv4 are:

* the address length has grown to 16 B
* network interface of particular machine has usually several different addresses
* address types and it's reachability has been redefined

Unicast
-------
A unicast address identifies a single network interface of individual host.
It is the most common type of address and it is used to one to one connections.
The IP address is divided to 2 main 64 bit parts: the *network prefix* and the *interface identifier*. 
Unlike IPv4, this division can't be easily changed because it would break other IPv6 standards such as address autoconfiguration.

Some well known prefixes of unicast addresses are:

* ``fe80::/10`` - link local addresses
* ``fc00::/7`` - unique local addresses
* ``2000::/3`` - global individual addresses

Anycast
-------
An anycast address identifies network interface of closest host from particular group.
Each anycast address has assigned a scope which limits the area of the network where the address is valid.
The scope should be kept small as possible because each router inside it stores individual routing entry for each such address.
As a result there are only few anycast addressees with a global scope such as these dedicated to root DNS servers.

Multicast
---------
A multicast address identifies network interfaces of all hosts from a group.
This type of address is recognized by having prefix ``ff00::/8``.
Moreover unlike IPv4, IPv6 defines 6 different multicast address scopes from link-local to global scope.
For example address for all hosts on particular local network is ``ff02::1`` (note that this is equivalent to IPv4 broadcast).


IPv6 configuration
==================
There are several ways to configure host running the IPv6 protocol.
The first solution is manual configuration, which can be helpful when setting up router or server machine with static address.
For most other uses the new protocol offers several versions of autoconfiguration.

Neighbor discovery
------------------
Before going further, it's worth mentioning the *neighbor discovery protocol* (ND, defined in RFC 4861) which provides:

* translation of link layer addresses (e.g. MAC) to network ones and vice versa
* router discovery (defines how hosts locate routers on the local network)
* autoconfiguration which includes setting up host address and other network parameters
* neighbor unreachability detection
* duplicate address detection
* redirection (defines how router informs a host about better way to reach particular destination)

The protocol defines five ICMPv6 message types which means that unreasonable blocking of ICMPv6 traffic may cause major problems for basic functionality of IPv6 network.

Network to link address
~~~~~~~~~~~~~~~~~~~~~~~
The process of searching the link address is similar to ARP used with IPv4.
Special multicast prefix (``ff02:0:0:0:0:1:ff00::/104``) is dedicated for this purpose. 

Let's see what will happen when host A needs to find out the link address
of host B (supposing host A knows the IP address of host B):

* Host A creates *solicited node address* (multicast) of host B using *the prefix*
  and the last 24 bits of host's B IP address.
* Host A sends *neighbor solicitation* (NS) ICPMv6 message with the full address
  he needs to translate to the *solicited node address*.
* Each host listens on multicast group for each IP address it has assigned.
  In most cases there will just one host in particular multicast
  group, which then checks if the IP address included in the message is
  assigned to him.
  This is true for the host B so he replies with
  *neighbor advertisement* (NA) ICMPv6 message with corresponding link address. 
  If the address changes, host can also send this message without prior NS message.
* Host A receives the NA message with the address and stores it in it's *neighbor cache*.

Link to network address
~~~~~~~~~~~~~~~~~~~~~~~
If a host needs to know an IP address for particular link address, the host will
create NS message with appropriate link address while the IP address will be
specified to ``ff02::1``, which is a multicast group for all hosts on the local
network.


Autoconfiguration and DHCPv6
----------------------------
Automatic configuration basically means that the host is able to generate and assign it's own IP
address based on information received from router on the local network without
any other help. This method uses a fact that local router already
knows all the needed network parameters such as mtu, network address prefix and
so on.

The group of last 64 bits of IP address (*interface identifier*)
which is unique for the host on particular local network can be:

* generated by the host itself from it's link address (IPv6 EUI-64, RFC 4291)
* generated by the host itself as random identifier (privacy extensions, RFC 4941)
* created by cryptographic protocol using secure alternative to neighbor discovery (SEND, RFC 3971)
* assigned from DHCPv6 server

Stateless configuration
~~~~~~~~~~~~~~~~~~~~~~~
This is the simplest autoconfiguration method. In this scenario the host will:

* create link local IP address
* check if the link local address is already in use, in case of conflict the
  autoconfiguration will fail and the network interface will be disabled
* assign itself the link local address
* receive router advertisement ICMPv6 message (by waiting or sending router solicitation message)
  with configuration for this network (network prefixes, default gateway, mtu, ...) 
* assign itself another IP address for each received network prefix with A flag set to 1

As a result if there is no prefix with flag A enabled, the autoconfiguration is disabled
and host ends up with just link local address.
If there is no IPv6 router available, the host will just try to assign itself the link local address.

The *router advertisement* message also contains 2 flags which instruct the host which configuration method to use.
Besides already mentioned *stateless configuration method* there are 2 other possibilities:

* *DHCPv6* (former *statefull configuration*) - host uses DHCPv6 server
* *stateless DHCPv6* - the combination of both previous options

DHCPv6
~~~~~~
The configuration process with DHCPv6 is the same as with *stateless autoconfiguration*
although the last step may be omitted and the DHCPv6 server
is contacted using well known multicast address. 
The protocol also assumes the host has assigned valid link local address because
the communication already takes place over IPv6.
And sice the DHCPv6 server doesn't provide information about the *default gateway*,
this method has to be used in combination with autoconfiguration.


DNS and IPv6
------------
The information about DNS servers can be provided by either
DHCPv6 server or *router advertisement* message during autoconfiguration.
Nevertheless the RFC 6106 which describes the way how to obtain addresses of the DNS servers during
stateless autoconfiguration is quite new with just few experimental implementations.
As a result the DHCPv6 is the only working way to get this information.


Network address translation
===========================
The primary purpose of the NAT mechanism is to reduce usage of network addresses.
Since the sheer number of IPv4 addresses is insufficient considering the current demand, this solution is ubiquitous and unenviable.

NAT in the IPv4 environment
---------------------------
In general, NAT is
a process translating addresses of machines from private subnet to limited set of public
addresses (often just one).
This way the local machines are able to establish
connections directed to the Internet without having it's own global IP addresses.
For the same reason this process makes the connections in the opposite direction impossible with various consequences:

* Good: a little increase in the security for the machines hidden behind NAT.
  Nevertheless NAT was not designed with security in mind.
  Thus without proper additional security measures, it rather gives the users false sense of security.
* Bad: breaks end to end communication protocols. These protocols supposes that hosts are able to connect to each other which is impossible with one of the host behind NAT without global public address.
* Ugly: some protocols (e.g. FTP) sends IP addresses in the data section of the packet.
  Machine providing NAT functionality has to understand the inner structure of packet to be able to translate the address properly. 

Emulation of some NAT features in IPv6
--------------------------------------
What started as a workaround to overcome the size of address space is now shaping our thinking about the way how the network should be run. This leads to an effort to find methods to bring some benefits of NAT to the IPv6 environment without introducing the same mechanism into IPv6 itself.

Changing the ISP
~~~~~~~~~~~~~~~~
The address space of network hidden behind NAT is independent of the service provider's address space 
making a switch to another ISP much easier.
Without renumbering, there is no need to reconfigure any machine with exception of the NAT server.

Obvious IPv6 solution is to use *unique local addresses* and store them in local DNS server.
These addresses are valid only inside our network and will be preserved when the service provider is changed. Thus there is no need for renumbering them.
The problem is that each node needs another global address at the same time.
On the one hand the autoconfiguration combined with DHCPv6 makes renumbering of client hosts seamless
on the other hand reconfiguration of servers dealing with IP addresses such as firewalls, DNS servers or IDS systems still requires significant effort.

Another solution might be the *network prefix translation* (NPTv6, previously called NAT66). Unlike IPv4 NAT, NPTv6 changes just a *network prefix* of an address making local machine accessible from the outside world. However the problem with protocols sending addresses outside the IP header still remains.
Apart from that, the protocol is still not standardized thus there are just few experimental implementations.

Multihoming
~~~~~~~~~~~
The local address space also helps when the network is connected to the Internet via several independent providers at the same time. 
The *provider independent address space* (PI) works with both versions of the internet protocol, but it's targeted only to big organizations. 

More accessible solution might be the shim6 protocol.
The basic idea behind this solution is that the protocol monitors global communication and ISP accessibility. In the case of primary ISP failure the connections are redirected to another ISP using methods borrowed from the mobility support. Unfortunately there are no stable implementations yet.


Privacy and pseudoanonymity
~~~~~~~~~~~~~~~~~~~~~~~~~~~
Node without global IP address can't be easily tracked using just the source
address. One of the ways to make nodes on a local network anonymous while each node having global IPv6 address is to enable "privacy extensions". This approach is described in more detail below.

Privacy extensions
==================
As has been mentioned, when using the autoconfiguration process, the last 64
bits of IPv6 address (the *host identifier*) are always the same.
This raises concern about security and privacy of nodes and it's users.

Although the risk of revealing identity on particular network exists as node
joins with the same IP address every time, the possibility to track node
across different networks is much more dangerous.
Since the IP address itself can't be easily hidden, it's relatively easy to
correlate seemingly unrelated set of addresses.
This can be exploited by web servers to track it's users without need to store
cookies or any other kind of identification on them.
Obviously the same correlation can be done by either any node in the path between
the communicating peers or anyone who has access to the communication logs.
Another way to exploit the fact that the MAC address is incorporated in the IPv6
address is targeted attack (eg. malware exploiting MacOS only).

The answer to this issue are *privacy extensions* for stateless address
autoconfiguration (RFC 4941).
The main idea behind this standard is to generate additional temporary address
with random *host id* part, which is used only for outgoing connections instead
of address generated by autoconfiguration - which is still generated,
but used for incoming
connections only. This way the new standard remains compatible with the
original autoconfiguration process (as defined in RFC 4862).

Description of the standard
---------------------------
The main features of this protocol are as follows:

* additional temporary address based on random *host id* (which replaces identifier based on the MAC address)
  is created for each standard public address created
* new random *host id* is regenerated on a periodic basis
* the sequence of random *host id* is generated in unpredictable way
* the random host id is used for all prefixes of the host reducing the
  number of multicast groups to join (nevertheless there is also a way to
  create different id for each prefix if needed)

How often should be a temporary address regenerated is a question of local
policy and usual value of the lifetime ranges within hours to days.
Moreover temporary address also conforms to *preferred lifetime* for particular
prefix from *router advertisement* message (this means that you can't have temporary
address with lifetime longer than autoconfigurated address has).
Also note when the public address is not created (eg. because the *preferred lifetime*
is set to 0) the temporary address is also not generated.

When temporary address reaches it's *preferred lifetime*, it becomes deprecated and the new one is created based on just computed new random *host id*.
Every new connection uses this new temporary address instead of the old one from now on.
Nevertheless already established connections remains untouched (using the old address) until the deprecated address reaches it's *valid lifetime*, when it becomes invalid and all connections using this address are terminated.

The primary way to enable the extensions is via global setting on each particular host.
Unfortunately there is no way for a local administrator to enforce usage of privacy extensions on the whole network.
However the discussion about this idea takes already place: unfinished draft
(draft-gont-6man-managing-privacy-extensions)
proposes new *router advertisement* flag which would enable or disable privacy extensions
for particular network prefix.

When application runs on a host with enabled privacy extensions, the temporary
address is preferred during the source address selection process unless the
application binds to the particular source address manually.
Since the manual selection is not so practical in IPv6 environment and the
choice of the type of address is good enough in most cases, new addrinfo and
setsockopts flags were defined in RFC 5014.
Using this flags, application can express it's preference of temporary address
without need to care about any details.
This standard is implemented in the Linux kernel since version 2.6.26.

Related problems
----------------
Unfortunately the privacy extensions have several drawbacks:

* Since temporary address is not stored in the DNS, it's not possible
  to do a reverse DNS lookup which can cause problems because some servers
  refuse to grant access to clients without proper DNS name.
  Such application has to select public address instead (using already
  mentioned new addrinfo flag).
  Another solution would be to abandon this DNS check or to create temporary DNS
  entries with random identifiers. But it's hard to guess
  if any of these ways is viable and which solution will be accepted in the future.
* Some applications suppose that all connections are made from the same source address.
* The effort to protect privacy of individual nodes conflicts with the effort to
  maintain and debug the whole network. Thus privacy extensions creates
  additional requirements to network monitoring solutions.

Because of wide range of issues, the latest RFC now recommends to disable the feature by default
and most of the implementations follows this advice.
In spite of this fact, the Windows operating systems are preconfigured
to use privacy extensions by default.

State of the implementations
----------------------------
The privacy extensions are supported by all major operating systems:

* Linux
* BSD family: FreeBSD, NetBSD, OpenBSD (since release 4.8)
* MacOS X
* Windows since Vista (support for version XP has been backported though)

The support among mobile phones and other portable machines which are more and more popular
these days is also noteworthy because such devices naturally travels across many networks and
are often binded to one person, which makes privacy extensions more important.
On the one hand both Android and iPhone iOS systems use kernel with support for privacy extensions,
on the other hand there is no way how to enable it on Android without rooting
the phone and official support for iPhone was introduced in iOS 4.3.

Monitoring
----------
The privacy extensions makes network debugging and monitoring significantly
harder. Because of already mentioned lack of a way to disable it on local network and
the fact that Windows systems has this protocol enabled by default,
every administrator building IPv6 monitoring system should take temporary
address into account.

The obvious thing to do is to create log of usage of IPv6 addresses and corresponding MAC addresses.
This can be done either by monitoring the *neighbor cache* on local routers or by
just analyzing the local IPv6 traffic.
One of monitoring suites using the former approach is MetaNAV, which
periodically collect this information via SNMP.
Such solution is then combined with other monitoring tools used on local
network (such as radius) to create useful data.
Nice description of similar system
which has been deployed in large environment can be found in paper entitled
"Practical IPv6 monitoring on campus".


References
==========

* http://knihy.nic.cz/ipv6
* http://www.lupa.cz/serialy/pohneme-s-ipv6/
* http://en.wikipedia.org/wiki/IPv6_address
* http://en.wikipedia.org/wiki/Network_address_translation
* http://code.google.com/p/android/issues/detail?id=14013
* http://support.apple.com/kb/HT4564
* http://www.terena.org/activities/campus-bp/pdf/gn3-na3-t4-cbpd132.pdf
* http://metanav.uninett.no/

RFC
---
* http://tools.ietf.org/html/rfc4193
* http://tools.ietf.org/html/draft-mrw-nat66-12
* http://tools.ietf.org/html/rfc5902
* http://tools.ietf.org/html/rfc4864
* http://tools.ietf.org/html/draft-ietf-v6ops-ipv6-multihoming-without-ipv6nat-00
* http://tools.ietf.org/html/rfc4861
* http://tools.ietf.org/html/rfc4007
* http://tools.ietf.org/html/rfc4941
* http://tools.ietf.org/html/draft-gont-6man-managing-privacy-extensions-01
* http://tools.ietf.org/html/rfc5014
* http://tools.ietf.org/html/rfc5220
