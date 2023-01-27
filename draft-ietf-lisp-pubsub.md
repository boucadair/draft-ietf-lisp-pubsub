---
title: "Publish/Subscribe Functionality for the Locator/ID Separation Protocol (LISP)"
abbrev: "LISP-PubSub"
category: std

docname: draft-ietf-lisp-pubsub-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Routing"
workgroup: "LISP Working Group"
keyword:
 - Automation
 - Reliable
 - Resiliency

author:
 -
   fullname: Alberto Rodriguez-Natal
   organization: Cisco
   country: Spain
   email: natal@cisco.com

 -
   fullname: Vina Ermagan
   organization: Google
   country: United States of America
   email: ermagan@gmail.com

 -
   fullname: Albert Cabellos
   organization: UPC/BarcelonaTech
   city: Barcelona
   country: Spain
   email: acabello@ac.upc.edu

 -
   fullname: Sharon Barkai
   organization: Nexar
   email: sharon.barkai@getnexar.com

 -
   fullname: Mohamed Boucadair
   organization: Orange
   city: Rennes
   country: France
   email: mohamed.boucadair@orange.com

normative:


informative:
     IANA-LISP:
        title: Locator/ID Separation Protocol (LISP) Parameters
        target: https://www.iana.org/assignments/lisp-parameters/lisp-parameters.xhtml
        date: 2023-01

--- abstract

   This document specifies an extension to the request/reply based
   Locator/ID Separation Protocol (LISP) control plane to enable
   Publish/Subscribe (PubSub) operation.

--- middle

#  Introduction

   The Locator/ID Separation Protocol (LISP) {{!RFC9300}} {{!RFC9301}} splits
   IP addresses in two different namespaces: Endpoint Identifiers (EIDs)
   and Routing Locators (RLOCs).  LISP uses a map-and-encap approach
   that relies on (1) a Mapping System (basically a distributed
   database) that stores and disseminates EID-RLOC mappings and on (2)
   LISP tunnel routers (xTRs) that encapsulate and decapsulate data
   packets based on the content of those mappings.

   Ingress Tunnel Routers (ITRs) / Re-encapsulating Tunnel Routers
   (RTRs) / Proxy Ingress Tunnel Routers (PITRs) pull EID-to-RLOC
   mapping information from the Mapping System by means of an explicit
   request message.  Section 6.1 of {{!RFC9301}} indicates how Egress
   Tunnel Routers (ETRs) can tell ITRs/RTRs/PITRs about mapping changes.
   This document presents a Publish/Subscribe (PubSub) extension in
   which the Mapping System can notify ITRs/RTRs/PITRs about mapping
   changes.  When this mechanism is used, mapping changes can be
   notified faster and can be managed in the Mapping System versus the
   LISP sites.

   In general, when an ITR/RTR/PITR wants to be notified for mapping
   changes for a given EID-Prefix, the following steps occur:

   1. The ITR/RTR/PITR sends a Map-Request for that EID-Prefix.

   1. The ITR/RTR/PITR sets the Notification-Requested bit (N-bit) on
        the Map-Request and includes its xTR-ID and Site-ID.

   1. The Map-Request is forwarded to one of the Map-Servers that the
        EID-Prefix is registered to.

   1. The Map-Server creates subscription state for the ITR/RTR/PITR
        on the EID-Prefix.

   1. The Map-Server sends a Map-Notify to the ITR/RTR/PITR to
        acknowledge the successful subscription.

   1. When there is a change in the mapping of the EID-Prefix, the
        Map-Server sends a Map-Notify message to each ITR/RTR/PITR in
        the subscription list.

   1. Each ITR/RTR/PITR sends a Map-Notify-Ack to acknowledge the
        received Map-Notify.

   This operation is repeated for all EID-Prefixes for which ITRs/RTRs/
   PITRs want to be notified.  An ITR/RTR/PITR can set the N-bit for
   several EID-Prefixes within a single Map-Request.

#  Terminology and Requirements Notation

{::boilerplate bcp14-tagged}

   The document uses the terms defined in Section 3 of {{!RFC9300}}.

#  Deployment Assumptions

   This document makes the following deployment assumptions:

   1. A unique 128-bit xTR-ID (plus a 64-bit Site-ID) identifier is
        assigned to each xTR.

   1. Map-Servers are configured to proxy Map-Replying (i.e., they are
        solicited to generate and send Map-Reply messages) for the
        mappings they are serving.

   If either assumption is not met, a subscription cannot be
   established, and the network will continue operating without this
   enhancement.  The configuration of xTR-IDs (and Site-IDs) are out of
   the scope of this document.

# Map-Request PubSub Additions

   {{Figure-1}} shows the format of the updated Map-Request to support the
   PubSub functionality.  In particular, this document associates a
   meaning with one of the reserved bits (see {{Section-IANA}}).

~~~~
        0                   1                   2                   3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |Type=1 |A|M|P|S|p|s|R|I|  Rsvd   |L|D|   IRC   | Record Count  |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                         Nonce . . .                           |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                         . . . Nonce                           |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |         Source-EID-AFI        |   Source EID Address  ...     |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |         ITR-RLOC-AFI 1        |    ITR-RLOC Address 1  ...    |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                              ...                              |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |         ITR-RLOC-AFI n        |    ITR-RLOC Address n  ...    |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     / |N|   Reserved  | EID mask-len  |        EID-Prefix-AFI         |
   Rec +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     \ |                       EID-Prefix  ...                         |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                   Map-Reply Record  ...                       |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                                                               |
       +                                                               +
       |                                                               |
       +                            xTR-ID                             +
       |                                                               |
       +                                                               +
       |                                                               |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                                                               |
       +                           Site-ID                             +
       |                                                               |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #Figure-1 title='Map-Request with I-bit, N-bit, xTR-ID, and Site-ID' artwork-align="center"}

The following is added to the Map-Request message defined in Section 5.2 of {{!RFC9301}}:

      - xTR-ID bit (I-bit):
      : This bit is set to 1 to indicate that a 128
      bit xTR-ID and a 64-bit Site-ID fields are present at the end of
      the Map-Request message.  For PubSub operation, an xTR MUST be
      configured with an xTR-ID and Site-ID, and it MUST set the I-bit
      to 1 and include its xTR-ID and Site-ID in the Map-Request
      messages it generates.  If the I-bit is set but the Site-ID and/or
      xTR-ID are not included, a receiver can detect the error because
      after processing that last EID-record, there are no bytes left
      from processing the message.  In this case, the receiver will log
      a malformed Map-Request and drop the message.

      - Notification-Requested bit (N-bit):
      : The N-bit of an EID-Record is
      set to 1 to specify that the xTR wants to be notified of updates
      for that mapping record.

      - xTR-ID field:
      : If the I-bit is set, this field if added at the end
      of the Map-Request message, starting after the final Record in the
      message (or the Map-Reply Record, if present).  The xTR-ID is
      specified in Section 5.6 of {{!RFC9301}}.

      - Site-ID field:
      : If the I-bit is set, this field is added at the end
      of the Map-Request message, following the xTR-ID.  The Site-ID is
      defined in Section 5.6 of {{!RFC9301}}.

#  Mapping Request Subscribe Procedures {#Section-5}

   The xTR subscribes for changes for a given EID-Prefix by sending a
   Map-Request to the Mapping System with the N-bit set on the EID-
   Record.  The xTR builds a Map-Request according to Section 5.3 of
   {{!RFC9301}} but also does the following:

   1.  The xTR MUST set the I-bit to 1 and append its xTR-ID and Site-
        ID to the Map-Request.  The xTR-ID uniquely identifies the xTR.

   1.  The xTR MUST set the N-bit to 1 for each EID-Record to which the
        xTR wants to subscribe.


   The Map-Request is forwarded to the appropriate Map-Server through
   the Mapping System.  This document does not assume that a Map-Server
   is pre-assigned to handle the subscription state for a given xTR.
   The Map-Server that receives the Map-Request will be the Map-Server
   responsible to notify that specific xTR about future mapping changes
   for the subscribed mapping records.

   Upon receipt of the Map-Request, the Map-Server processes it as
   described in Section 8.3 of {{!RFC9301}}.  Furthermore, upon processing,
   for each EID-Record that has the N-bit set to 1, the Map-Server
   proceeds to add the xTR-ID contained in the Map-Request to the list
   of xTRs that have requested to be subscribed to that EID-Record.

   If an xTR-ID is successfully added to the list of subscribers for an
   EID-Record, the Map-Server MUST extract the nonce and ITR-RLOCs
   present in the Map-Request, and store the association between the
   EID-Record, xTR-ID, ITR-RLOCs and nonce.  Any already present state
   regarding ITR-RLOCs and/or nonce for the same xTR-ID MUST be
   overwritten.  When the LISP deployment has a single Map-Server, the
   Map-Server can be configured to keep a single nonce per xTR-ID for
   all EID-Records (when used, this option MUST be enabled at the Map-
   Server and all xTRs).

   If the xTR-ID is added to the list, the Map-Server MUST send a Map-
   Notify message back to the xTR to acknowledge the successful
   subscription.  The Map-Server builds the Map-Notify according to
   Sections 5.5 and 5.7 of {{!RFC9301}} with the following considerations:

   1.  The Map-Server MUST use the nonce from the Map-Request as the
        nonce for the Map-Notify.

   1.  The Map-Server MUST use its security association with the xTR
        ({{Section-7.1}}) to sign the authentication data of the Map-Notify.
        The xTR MUST use the security association to verify the received
        authentication data.

   1.  The Map-Server MUST send the Map-Notify to one of the ITR-RLOCs
        received in the Map-Request (which one is implementation
        specific).


   When the xTR receives a Map-Notify with a nonce that matches one in
   the list of outstanding Map-Request messages sent with an N-bit set,
   it knows that the Map-Notify is to acknowledge a successful
   subscription.  The xTR processes this Map-Notify as described in
   Section 5.7 of {{!RFC9301}} with the following considerations.  The xTR
   MUST use the Map-Notify to populate its Map-Cache with the returned
   EID-Prefix and RLOC-set.

   The subscription of an xTR-ID may fail for a number of reasons.  For
   example, it fails because of local configuration policies (such as
   accept and drop lists of subscribers) or because the Map-Server has
   exhausted the resources to dedicate to the subscription of that EID-
   Record (e.g., the number of subscribers excess the capacity of the
   Map-Server).

   If the subscription request fails, the Map-Server MUST send a Map-
   Reply to the originator of the Map-Request, as described in
   Section 8.3 of {{!RFC9301}}.  The xTR processes the Map-Reply as
   specified in Section 8.1 of {{!RFC9301}}.

   The subscription state can also be created explicitly by
   configuration at the Map-Server (possible when a pre-shared security
   association exists, see {{Section-Sec}}).  In this case, the initial nonce
   associated with the xTR-ID (and EID-Record) MUST be randomly
   generated by the Map-Server.

   The following specifies the procedure to remove a subscription.  If
   the Map-Request only has one ITR-RLOC with AFI = 0 (i.e., Unknown
   Address), the Map-Server MUST remove the subscription state for that
   xTR-ID.  In this case, the Map-Server MUST send the Map-Notify to the
   source RLOC of the Map-Request.

   When an EID-Record is removed from the Map-Server (either when
   explicitly withdrawn or when its TTL expires), the Map-Server
   notifies its subscribers (if any) via a Map-Notify with TTL equal 0.

#  Mapping Notification Publish Procedures

   The publish procedure is implemented via Map-Notify messages that the
   Map-Server sends to xTRs.  The xTRs acknowledge the reception of Map-
   Notifies via sending Map-Notify-Ack messages back to the Map-Server.
   The complete mechanism works as follows.

   When a mapping stored in a Map-Server is updated (e.g., via a Map-
   Register from an ETR), the Map-Server MUST notify the subscribers of
   that mapping via sending Map-Notify messages with the most updated
   mapping information.  If subscription state in the Map-Server exists
   for a less-specific EID-Prefix and a more-specific EID-Prefix is
   updated, then the Map-Notify is sent with the more-specific EID-
   Prefix mapping to the subscribers of the less-specific EID-Prefix
   mapping.  The Map-Notify message sent to each of the subscribers as a
   result of an update event follows the encoding and logic defined in
   Section 5.7 of {{!RFC9301}} for Map-Notify, except for the following:

   1.  The Map-Notify MUST be sent to one of the ITR-RLOCs associated
        with the xTR-ID of the subscriber (which one is implementation
        specific).

   1.  The Map-Server increments the nonce by one every time it sends a
        Map-Notify as publication to an xTR-ID for a particular EID-
        Record.

   1.  The Map-Server MUST use its security association with the xTR to
        compute the authentication data of the Map-Notify.


   When the xTR receives a Map-Notify with an EID not local to the xTR,
   the xTR knows that the Map-Notify has been received to update an
   entry on its Map-Cache.  The xTR MUST keep track of the last nonce
   seen in a Map-Notify received as a publication from the Map-Server
   for the EID-Record.  When the LISP deployment has a single Map-
   Server, the xTR can be configured to keep track of a single nonce for
   all EID-Records (when used, this option MUST be enabled at the Map-
   Server and all xTRs).  If a Map-Notify received as a publication has
   a nonce value that is not greater than the saved nonce, the xTR drops
   the Map-Notify message and logs the fact a replay attack could have
   occurred.  The same considerations discussed in Section 5.6 of
   {{!RFC9301}} regarding Map-Register nonces apply here for Map-Notify
   nonces.

   The xTR processes the received Map-Notify as specified in Section 5.7
   of {{!RFC9301}}, with the following considerations.  The xTR MUST use
   its security association with the Map-Server ({{Section-7.1}}) to
   validate the authentication data on the Map-Notify.  The xTR MUST use
   the mapping information carried in the Map-Notify to update its
   internal Map-Cache.  If after a configurable timeout, the Map-Server
   has not received back the Map-Notify-Ack (as per Section 5.7 of
   {{!RFC9301}}), it can try to send the Map-Notify to a different ITR-RLOC
   for that xTR-ID.  If the Map-Server tries all the ITR-RLOCs without
   receiving a response, it may stop trying to send the Map-Notify.

#  Security Considerations {#Section-Sec}

   Generic security considerations related to LISP control messages are
   discussed in Section 9 of {{!RFC9301}}.

   In the particular case of PubSub, cache poisoning via malicious Map-
   Notify messages is avoided by the use of nonce and the security
   association between the ITRs and the Map-Servers.

##  Security Association between ITR and Map-Server {#Section-7.1}

   Since Map-Notifies from the Map-Server to the ITR need to be
   authenticated, there is a need for a soft-state or hard-state
   security association (e.g., a PubSubKey) between the ITRs and the
   Map-Servers.  For some controlled deployments, it might be possible
   to have a shared PubSubKey (or set of keys) between the ITRs and the
   Map-Servers.  However, if pre-shared keys are not used in the
   deployment, LISP-SEC {{!RFC9303}} can be used as follows to create a
   security association between the ITR and the MS.

   First, when the ITR is sending a Map-Request with the N-bit set
   following {{Section-5}}, the ITR also performs the steps described in
   Section 5.4 of {{!RFC9303}}.  The ITR can then generate a PubSubKey by
   deriving a key from the One-Time Key (OTK) as follows: PubSubKey =
   KDF( OTK ), where KDF is the Key Derivation Function indicated by the
   OTK Wrapping ID.  If OTK Wrapping ID equals NULL-KEY-WRAP-128 then
   the PubSubKey is the OTK.  Note that as opposed to the pre-shared
   PubSubKey, this generated PubSubKey is different per EID-Record the
   ITR subscribes to (since the ITR will use a different OTK per Map-
   Request).

   When the Map-Server receives the Map-Request it follows the procedure
   specified in {{Section-5}}.  If the Map-Server has to reply with a Map-
   Reply (e.g., due to PubSub not supported or subscription not
   accepted), then it follows normal LISP-SEC procedure described in
   Section 5.7 of {{!RFC9303}}.  No PubSubKey or security association is
   created in this case.

   Otherwise, if the Map-Server has to reply with a Map-Notify (e.g., due
   to subscription accepted) to a received Map-Request, the following
   extra steps take place:

   *  The Map-Server extracts the OTK and OTK Wrapping ID from the LISP-
      SEC ECM Authentication Data.

   *  The Map-Server generates a PubSubKey by deriving a key from the
      OTK as described before for the ITR.  This is the same PubSubKey
      derived at the ITR which is used to establish a security
      association between the ITR and the Map-Server.

   *  The PubSubKey can now be used to sign and authenticate any Map-
      Notify between the Map-Server and the ITR for the subscribed EID-
      Record.  This includes the Map-Notify sent as a confirmation to
      the subscription.  When the ITR wants to update the security
      association for that Map-Server and EID-Record, it follows again
      the procedure described in this section.

   Note that if the Map-Server replies with a Map-Notify, none of the
   regular LISP-SEC steps regarding Map-Reply described in Section 5.7
   of {{!RFC9303}} takes place).

##  DDoS Attack Mitigation

   Misbehaving nodes may send massive subscription requests which may
   lead to exhaust the resources of a Map-Server.  Furthermore,
   frequently changing the state of a subscription may also be
   considered as an attack vector.  To mitigate such issues, Section 5.3
   of {{!RFC9301}} discusses rate-limiting Map-Requests and Section 5.7 of
   {{!RFC9301}} discusses rate-limiting Map-Notifies.  Note that when the
   Map-Notify rate-limit threshold is met for a particular xTR-ID, the
   Map-Server will discard additional subscription requests from that
   xTR-ID and will fall back to {{!RFC9301}} behavior when receiving a Map-
   Request from that xTR-ID (i.e., the Map-Server will send a Map-
   Reply).

#  Sample PubSub Deployment Experiences

   Early implementations of PubSub have been running in production
   networks for some time.  The following subsections provides an
   inventory of some experience lessons from these deployments.

##  PubSub as a Monitoring Tool

   Some LISP deployments are using PubSub as a way to monitor EID-
   Prefixes (particularly, EID-to-RLOC mappings).  To that aim, some
   LISP implementations have extended the LISP Internet Groper (lig)
   {{?RFC6835}} tool to use PubSub.  Such an extension is meant to support
   an interactive mode with lig, and request subscription for the EID of
   interest.  If there are RLOC changes, the Map-Server sends a
   notification and then the lig client displays that change to the
   user.

##  Mitigating Negative Map-Cache Entries

   Section 8.1 of {{!RFC9301}} suggests two TTL values for Negative Map-
   Replies, either 15-minute (if the EID-Prefix does not exist) or
   1-minute (if the prefix exists but has not been registered).  While
   these values are based on the original operational experience of the
   LISP protocol designers, negative cache entries have two unintended
   effects that were observed in production.

   First, if the xTR keeps receiving traffic for a negative EID
   destination (i.e., an EID-Prefix with no RLOCs associated with it),
   it will try to resolve the destination again once the cached state
   expires, even if the state has not changed in the Map-Server.  It was
   observed in production that this is happening often in networks that
   have a significant amount of traffic addressed for outside of the
   LISP network.  This might result on excessive resolution signaling to
   keep retrieving the same state due to the cache expiring.  PubSub is
   used to relax TTL values and cache negative mapping entries for
   longer periods of time, avoiding unnecessary refreshes of these
   forwarding entries, and drastically reducing signaling in these
   scenarios.  In general, a TTL-based schema is a "polling mechanism"
   that leads to more signaling where PubSub provides an "event
   triggered mechanism" at the cost of state.

   Second, if the state does indeed change in the Map-Server, updates
   based on TTL timeouts might prevent the cached state at the xTR from
   being updated until the TTL expires.  This behavior was observed
   during configuration (or reconfiguration) periods on the network,
   where no-longer-negative EID-Prefixes do not receive the traffic yet
   due to stale Map-Cache entries present in the network.  With the
   activation of PubSub, stale caches can be updated as soon as the
   state changes.

##  Improved Mobility Latency

   An improved convergence time was observed on the presence of mobility
   events on LISP networks running PubSub as compared with running LISP
   {{!RFC9303}}.  As described in Section 4.1.2.1 of
   {{?I-D.ietf-lisp-eid-mobility}}, LISP can rely on data-driven Solicit-
   Map-Requests (SMRs) to ensure eventual network converge.  Generally,
   PubSub offers faster convergence due to (1) no need to wait for a data
   triggered event and (2) less signaling as compared with the SMR-based
   flow.  Note that when a Map-Server running PubSub has to update a
   large number of subscribers at once (i.e., when a popular mapping is
   updated) SMR based convergence may be faster for a small subset of
   the subscribers (those receiving PubSub updates last).  Deployment
   experience reveals that data-driven SMRs and PubSub mechanisms
   complement each other well and combined provide a fast and resilient
   network infrastructure in the presence of mobility events.

   Furthermore, experience showed that not all LISP entities on the
   network need to implement PubSub for the network to get the benefits.
   Concretely, in scenarios with significant traffic coming from outside
   of the LISP network, the experience showed that enabling PubSub in
   the border routers, significantly improves mobility latency overall,
   even if edge xTRs do not implement PubSub and traffic exchanged
   between EID-Prefixes at the edge xTRs still converges based on data-
   driven events and SMR-triggered updates.

##  Enhanced Reachability with Dynamic Redistribution of Prefixes

   There is a need to interconnect LISP networks with other networks
   that might or might not run LISP.  Some of those scenarios are
   similar to the ones described in {{?I-D.haindl-lisp-gb-atn}} and
   {{?I-D.moreno-lisp-uberlay}}.  When connecting LISP to other networks,
   the experience revealed that in many deployments the point of
   interaction with the other domains is not the Mapping System but
   rather the border router of the LISP site.  For those cases the
   border router needs to be aware of the LISP prefixes to redistribute
   them to the other networks.  Over the years different solutions have
   been used.

   First, Map-Servers were collocated with the border routers, but this
   was hard to scale since border routers scale at a different pace than
   Map-Servers.  Second, decoupled Map-Servers and border routers were
   used with static configuration of LISP entries on the border, which
   was problematic when modifications were made.  Third, a routing
   protocol (e.g., BGP) can be used to redistributed LISP prefixes from
   the Map-Servers to a border router, but this comes with some
   implications, particularly the Map-Servers needs to implement an
   additional protocol which consumes resources and needs to be properly
   configured.  Therefore, once PubSub was available, deployments
   started to adapt it to enable border routers to dynamically learn the
   prefixes they need to redistribute without the need of extra
   protocols or extra configuration on the network.

   In other words, PubSub can be used to discover EID-Prefixes so they
   can be imported into other routing domains that do not use LISP.
   Similarly, PubSub can also be used to discover when EID-Prefixes need
   to be withdrawn from other routing domains.  That is, in a typical
   deployment, a border router will withdraw an EID-Prefix it has been
   announcing to external routing domains, if it receives a notification
   that the RLOC-set for that EID-Prefix is now empty.

##  Better Serviceability

   As per Section 6.6.1 of {{?RFC6830}}, the default setting for an EID-to-
   RLOC mapping TTL in the cache is 24 hours.  Upon the expiry of that
   TTL, the xTR checks if these entries are being used and removes any
   entry that is not being used.  The problem with this 24-hour Map-
   Cache TTL is that (in the absence of PubSub) if a mapping changes,
   but it is not being used, the cache remains but it is stale.  This is
   due to no data traffic being sent to the old location to trigger an
   SMR based Map-Cache update as described in Section 4.1.2.1 of
   {{?I-D.ietf-lisp-eid-mobility}}.  If the network operator runs a show
   command on a router to track the state of the Map-Cache, the router
   will display multiple entries waiting to expire but with stale RLOC
   information.  This might be confusing for operators sometimes,
   particularly when they are debugging problems.  Interestingly, with
   PubSub at least the Map-Cache is updated with the correct RLOC
   information, even when it is not being used or waiting to expire,
   which helps debugging.

#  IANA Considerations {#Section-IANA}

   This document requests IANA to assign a new bit from the "LISP
   Control Plane Header Bits: Map-Request" sub-registry under the
   "Locator/ID Separation Protocol (LISP) Parameters" registry available
   at {{IANA-LISP}}.  The suggested position of this bit in the Map-
   Request message can be found in {{Figure-1}}.

~~~~
     +======+===============+==========+=============+===============+
     | Spec | IANA Name     | Bit      | Description | Reference     |
     | Name |               | Position |             |               |
     +======+===============+==========+=============+===============+
     | I    | map-request-I | 11       | xTR-ID Bit  | This-Document |
     +------+---------------+----------+-------------+---------------+

       Table 1: Additions to the Map-Request Header Bits Sub-Registry
~~~~

   This document also requests the creation of a new sub-registry
   entitled "LISP Control Plane Header Bits: Map-Request-Record" under
   the "Locator/ID Separation Protocol (LISP) Parameters" registry
   available at {{IANA-LISP}}.

   The initial content of this sub-registry is shown in Table 2:

~~~~
   +====+=============+========+========================+=============+
   |Spec|IANA Name    |Bit     | Description            |Reference    |
   |Name|             |Position|                        |             |
   +====+=============+========+========================+=============+
   |N   |map-request-N|1       | Notification-Requested |This-Document|
   |    |             |        | Bit                    |             |
   +----+-------------+--------+------------------------+-------------+

     Table 2: Initial Content of LISP Control Plane Header Bits: Map-
                       Request-Record Sub-Registry
~~~~

   The remaining bits are Unassigned.

   The policy for allocating new bits from this sub-registry is
   Specification Required (Section 4.6 of {{!RFC8126}}).

   Review requests are evaluated on the advice of one or more designated experts.
   Criteria that should be applied by the designated experts include
   determining whether the proposed registration duplicates existing
   entries and whether the registration description is sufficiently detailed and fits
   the purpose of this registry.  These criteria are considered in
   addition to those already provided in Section 4.6 of {{!RFC8126}} (e.g.,
   the proposed registration must be documented in a permanent and
   readily available public specification).  The designated
   experts will either approve or deny the registration request,
   communicating this decision to IANA.  Denials should include an
   explanation and, if applicable, suggestions as to how to make the
   request successful.

--- back

# Acknowledgments
{:numbered="false"}

   We would like to thank Marc Portoles, Balaji Venkatachalapathy, and
   Padma Pillay-Esnault for their great suggestions and help regarding
   this document.  This work was partly funded by the ANR LISP-Lab
   project #ANR-13-INFR-009 (https://www.lisp-lab.org).

Many thanls to Alvaro Retano for the careful AD review.

Thanks to Chris M. Lonvick for the security directorate review, Al Morton for the OPS-DIR review, Roni Even for the Gen-ART review, Mike McBride for the rtg-dir review, and  Magnus Westerlund for the tsv directorate review.

# Contributors
{:numbered="false"}

-
fullname: Dino Farinacci
organization: lispers.net
city: San Jose
state: CA
country: USA
email: farinacci@gmail.com

-
fullname: Johnson Leong
email: johnsonleong@gmail.com

-
fullname: Fabio Maino
organization: Cisco
street: 170 Tasman Drive
city: San Jose
state: CA
country: USA
email: fmaino@cisco.com

-
fullname: Christian Jacquenet
organization: Orange
city: Rennes
code: 35000
country: France
email: christian.jacquenet@orange.com

-
fullname: Stefano Secci
organization: Cnam
country: France
email: stefano.secci@cnam.fr