---
title: CoAP operations for YANG push via the Concise Management Interface
docname: draft-birkholz-yang-push-coap-problemstatement-latest
date: 2017-07-19

ipr: trust200902
area: security
wg: SACM Working Group
kw: Internet-Draft
cat: std

coding: us-ascii
pi:
   toc: yes
   sortrefs: yes
   symrefs: yes
   comments: yes

author:
- ins: H. Birkholz
  name: Henk Birkholz
  org: Fraunhofer SIT
  abbrev: Fraunhofer SIT
  email: henk.birkholz@sit.fraunhofer.de
  street: Rheinstrasse 75
  code: '64295'
  city: Darmstadt
  country: Germany

normative:
  RFC7252:
  RFC7641:
  RFC8040:
  RFC7049:
  RFC8132: 
  RFC8071:
  I-D.ietf-core-yang-cbor: yangcbor
  I-D.ietf-core-coap-pubsub: pubsub
  I-D.ietf-core-comi: comi
  I-D.ietf-core-coap-tcp-tls: reliable
  I-D.ietf-netconf-subscribed-notifications: yangnote
  I-D.ietf-netconf-yang-push: yangpush
 
informative:

--- abstract

TEST

--- middle

# Context of the Problem

This document illustrates the problem statement and corresponding gaps of solutions in respect to a
binary transfer protocol to operate on YANG datastores that can be extended to support YANG push
{{-yangpush}} and YANG Subscribed Notifications {{-yangnote}}, respectively.

## Binary YANG transfer protocol

The Concise Management Interface I-D (CoMI {{-comi}}) defines operations for a YANG datastore based
on the Concise Application Protocol (CoAP {{RFC7252}}). CoAP uses a request/response interaction
model that is based on HTTP (similar to RESTCONF {{RFC8040}}) and allows for multiple transports,
including UDP or TCP (see {{-reliable}}. The Concise Binary Object Representation (CBOR {{RFC7049}}
is used for the serialization of data in motion in respect to CoAP operations and the data modeled
with YANG {{-yangcbor}}.

## Subscriptions via CoAP

The CoAP pub/sub I-D defines a CoAP Subscribe operation {{-pubsub}} that is based on observing
resources via the Observe option for the GET operation as defined in {{RFC7641}}. The CoAP pub/sub
draft is intended to provide the capabilities and characteristics of MQTT via a CoAP based protocol.
The only other CoAP operation that supports the Observe option is the FETCH operation defined in
{{RFC8132}}.

The Observe option creates a small corresponding state on the server side that eliminates the need
for continuous polling of a resource via subsequent requests. Instead, subsequent responses including
both the Observe option and using the token of the request that initiated the observation are
returned when the observed resource changes. A subscription (i.e. the observe state retained on the
server) can be discarded by the client by sending a correspond CoAP GET with Observe using an Observe
parameter of 1 or simply by "forget" the observation and return a CoAP Reset after receiving a
notification in the context of the subscription. A subscription can also be discarded by the server
by sending a corresponding response that does not contain an Observe option.

The subscription used in CoAP pub/sub are used to subscribe to a topic provided by a CoAP broker REST API. YANG Push {{-yangpush}} and corresponding YANG Subscribed Notifications are used to subscribe to data node updates provided by a YANG management interface. YANG subscriptions can include a filter expression (either a subtree expression or an XPATH expression). The encoding rules of XPATH expressions in CBOR are covered by {{-yangcbor}}.

## Configured Subscriptions and Call-Home

Configured subscriptions are basically static configuration that creates subscription state on the
YANG data store when it is started and persists between boot-cycles without the need of a client to
create that subscription state. In consequence, a configured subscription can result in unsolicited
pushed notifications in respect to a YANG client.

A popular variant of the configured subscription as defined in {{-yangpush}} is the Call Home
procedure defined in {{RFC8071}}. In this approach, a Transport Layer Application Association is
initiated by the YANG data store with the YANG client. After this "initial phase, in which the YANG
server is as acting like client", the existing Transport Layer connection (or session, in case of,
for example, TLS) is then used to by the YANG client to initiate a subscription (i.e. the YANG
client is initiating a dynamic subscription based on a pre-configured request retained and issued by
the YANG data store).

# Summary of the Problem Statement

Currently, the following gaps are identified:

* no CoAP Subscribe procedure for dynamic YANG subscriptions is standardized that is able to convey
  a filter expression and potentially other metadata required in the context of a YANG Subscribed
  Notifications application association.
* no CoAP call home feature is standardized to support a popular variant of configured YANG
  subscriptions.

In addition to the identified gaps, the semantics of metadata - if there are any - that have to be
conveyed to a YANG data store in order to subscribe to a (filtered) YANG module or data node are not
identified. 

# Potential Approaches and Solutions

There are multiple approaches that could lead to viable solutions that address the identified gaps.
The following sections illustrate the general solution context and some of the most promising
approaches.

## YANG subscription variants

A YANG Push update subscription service both provides support for dynamic subscription (i.e.
subscription state created by a client request, allowing for solicited push notifications in the
context of a up-time cycle of the server) and configured subscription (i.e. subscription
configuration retained on the server, allowing for unsolicited push notifications across up-time
cycles of the server).

## YANG Push via CoAP 

The two CoAP operations that enable a subscription mechanism are GET and FETCH (i.e. by supporting
the Observe option). Both operations are viable candidates for creating a CoAP-based YANG push
mechanism for CoMI.

## Dynamic Subscriptions

Using CoAP, the client issuing the initial subscription request creates the subscription state.
Examples are the GET or FETCH operation including an Observe option using an Observe parameter of 0
(zero).

### YANG Push via GET

This usage scenario requires two consecutive operations. It is not possible to transfer a filter
expression included in a GET operation. In consequence, a POST operation on a collection resource
has to be conducted in order to convey a filter expression to the YANG data store, allowing it to
return an URI that contains the data node information filtered in respect to the posted filter
expression (encoded in CBOR).

This variant allows for multiple clients to observe a specific filtered data node without conducting
a POST operation if the corresponding URI is made known to other clients that did not conduct the
POST operation or, for example, is canonically linked to/derivable from a filter expression.

### YANG Push via FETCH

This usage scenario requires only one operation. A FETCH operation can include a body that is
capable to contain a filter expression and potentially other metadata that might be required to
establish a suitable subscription state on the YANG data store.

It might be possible that this variant could introduce a slight delay in respect to response time if
providing a filtered resource requires a lot of computation time on a constrained device. I.e. the
resource cannot be prepared "beforehand".

## Configured Subscriptions

Using CoAP, the server retains configuration that creates subscription state when the YANG data
store is started. The client has to have or gain knowledge of the CoAP token that are included in
the responses created in the context of the subscription state create from server configuration.

### Retaining the Content of a GET Operation as Configuration

This usage scenario "mimics" the receiving of a subscription request by storing the corresponding
information that are relevant for creating a subscription state as configuration on the YANG data
store. I.e. the configuration would be including the YANG client IP address and the CoAP token to be
used in the responses that convey the subscribed notifications.

This variant requires that the client also knows or gains knowledge of the corresponding CoAP token
in order to not discard the incoming responses.

### Call Home via CoAP

This usage scenario defines the Call Home procedure standardized in {{RFC8071}} as an additional
capability of CoAP. DTLS or TLS state is initiated by the YANG data store and triggers a dynamic
subscription procedure of the YANG client using the session initiated by the YANG data store.

#  IANA considerations

This document includes no requests to IANA, but solutions drafts incubated via this document might.

#  Security Considerations

This document includes no security considerations, but solution drafts incubated via this document
will.

#  Acknowledgements

Carsten Bormann, Klaus Hartke, Eric Voit

#  Change Log

First version -00

--- back
