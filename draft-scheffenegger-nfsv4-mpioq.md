---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "NFSv4 advice on multipath IO queueing"
abbrev: "NFS MPIOQ"
category: info

docname: draft-scheffenegger-nfsv4-mpioq-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: TSV
workgroup: NFSv4
keyword:
 - nconnect
 - pnfs
 - multipath
 - leaf-spine
venue:
  group: WG
  type: Working Group
  mail: WG@example.com
  arch: https://example.com/WG
  github: USER/REPO
  latest: https://example.com/LATEST

author:
 -
    ins: R. Scheffenegger
    fullname: Richard Scheffenegger
    organization: NetApp, Inc.
    email: srichard@netapp.com

normative:

informative:


--- abstract

This draft gives guidance on how to optimize the utilization of
modern network topologies, in order to maximize performance
characteristics such as bandwidth, IOPS or per-IO completion time
(latency) when using standard (pNFS) or non-standard (nconnect)
enhancements of the NFS protocol.

Furthermore, an interaction between TCP behavior and clients
issuing NFS IO is described, when IO pauses beyond a session
specific time period (TCP idle), which may lead to unexpectedly
long IO completion times during periods of very low NFS IO load.

--- middle

# Introduction

Modern use of the NFS protocol often includes capabilities to
utilize multiple parallel TCP sessions between the client and
the NFS server. This may be due to built-in protocol features
such as pNFS, or by clients exploiting certain properties of
the NFS protocol to unilaterally deploy similar capabilites,
without a standard describing this action, like the nconnect
mount option frequently found.

While all these enhancements aim to improving performance
between an NFS client and server, by using more than a single
session between the same (or different) IP endpoints, the
implicit underlying assumption especially in the case of
two identical L3 endpoints (e.g. same source and destination
IP address) is that the network properties of this path
will be identical for the set of sessions using the same
addresses.

However, modern network topologies (i.e. leaf-spine) can and
will distribute individual L4 flows along vastly different paths,
breaking the implicit assumption of a shared fate among this group
of sessions.

In order to maximize a performance metric like bandwidth, IOPS
or per-IO completion time, it becomes necessary to monitor
the indivial session performance characteristic, and if needed,
adjust the clients IO scheduling accordingly.

# Round-Robin

Traditionally, implementations of NFS multipath capabilities
such as pNFS or nconnect only implemented the most basic IO
schedling algorithm - round-robin - for an appropriate subset
of NFS operations. This subset typically includes READs, WRITEs,
READDIR and READDIRPLUS.

# Least Queue Depth

When the network topology is forwarding different L4 flows along
different phyiscal paths, such as would be the case when there
are multiple paths of equal cost available to a routing/forwarding
engine, those sessions typically no longer share a common fate.

That is, each flow may expirience different levels of latency,
effective bandwidth, congestion points and cross-traffic.

While the impact may be minimal while the network can supply a high
excess capacity for most of the time, once the utilization factor
of the network increases, those effects become more pronounced.

As the TCP protocol is designed to probe constantly for maximum
network bandwidth, and transmit only at a suitable rate, for traffic
that predominately used the direction from the NFS client towards the
NFS server (e.g. WRITEs), an NFS client implementation can observe
the local Send-Queue utilization, to determine the relative drain
rates between the Queues associated with the set of TCP sessions
used in the communication with the NFS server. Once this infomation
is available, IO can be scheduled accordingly to use the TCP session
where the observed Queue Depth is the lowest.

Probing these parameters on a per-IO basis may introduce an undue
overhead, so for efficiency, the sampling rate may be reduced
significantly, e.g. once every n (10..1000) RPCs, or every n milliseconds,
and in between the NFS client may utilize a modified IO distribution
algorithm with weights associated to each TCP session, where the weight
is updated frequently.

## Challenge of the return channel

Unlike other protocols with built in multipath capabilities, RPC/NFS
requires a response to be returned on the same session.

Therefore, the IO scheduling can not be a unilateral decision by an NFS
server, independently from the client. In fact, the NFS server may not
even be aware, that a subset of TCP sessions belong to the same NFS client
at all in the case of nconnect.

Is it therefore also necessary for the client, to build an understanding
of the servers utilization/queueing levels of the set of TCP sessions used.

Unlike in the above described case of the forward channel from the NFS client
to the NFS server, the NFS client can not have a direct understanding of
Socket Sendbuffer utilization levels. It is therefore necessary for the
client to use some proxy when the dominant data direction is from the NFS
server to the NFS client (e.g. READs).

In order to exclude effects due to IO fetching from a backing stable storage
subsystem, it is undesirable to use the IO completion times for READ requests
directly. Instead, it appears more useful for the NFS client to issue RPC NULL
requests, and use the completion times for those RPC requests while data IOs
are also being processed, as a proxy of both the network bandwidth and
server (instance) responsiveness. Once equipped with this information, the
IO scheduling in the NFS client can be adjusted such to balance the relative
response times - thus maximizing IOPS and bandwidth between NFS client and server

# Interaction between NFS IO and TCP Idle

Another aspect exaberated when an NFS client starts utilizing
multiple TCP sessions to distribute IO load is the interaction
with the TCP idle timer.

When a TCP session does not transmit data for a certain amount
of time some of the dynamic state of the involved TCP session is
reset, and in particular the effective bandwidth (congestion
window) allowed to be used by the session is reset.

After an idle period, the TCP congestion window has to ramp up
from a very small initial value, the Initial Window (between 2
and 10 segment sizes worth of data) - provided that the data
sender provides sufficient data to facilitate this ramp-up.

The length of time, after which this idling happens is effectively
governed by the dynamically calculated retransmission timeout
interval, where most, if not all, TCP implementations also impose
a lower limit in the order of 100 to 400 milliseconds - with
around 200 milliseconds used most commonly at the time of this
writing.

This can have the detrimental effect that when a combination of
the NFS client trying to distribute the IO load across a high
number of sessions, each individual session may only transmit
data with pauses in between exceeding that sessions idle interval,
while transmitting the same (low) volume of IOs across fewer, or
only a single session would maintain the dynamic state of the
involved TCP sessions.

Under low load scenarios, it will therefore be beneficial for
an NFS client to concentrate IOs such that only a smaller subset
of the total number of sessions is kept busy (non-idling), and
distribute IO across more sessions only, when the number of IOs
dispatched is sufficiently high.

Alternatively, a client may also choose to keep the dynamic TCP
state from being invalidated, by providing a baseline load at a
rate slightly higher than when the TCP state would be lost. This
may be achieved by dispatching NULL RPCs with an individual,
per-session timer firing every 200 milliseconds, but reset
whenever other RPCs are issued.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

The selection of a different IO scheduling mechanism has no
security considerations above and beyond the base protocol.


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

The authors want to acknowledge the support and help by Rick Macklem early on while
