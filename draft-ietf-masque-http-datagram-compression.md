---
title: "Extensions to Compress and Derive Fields in HTTP Datagrams"
abbrev: "HTTP Datagram Compression"
category: std

docname: draft-ietf-masque-http-datagram-compression-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Multiplexed Application Substrate over QUIC Encryption"
keyword:
 - template
 - checksum
 - connect-ip
venue:
  group: "Multiplexed Application Substrate over QUIC Encryption"
  type: "Working Group"
  mail: "masque@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/masque/"
  github: "yaroslavros/connect-ip-optimizations"
  latest: "https://yaroslavros.github.io/connect-ip-optimizations/draft-ietf-masque-http-datagram-compression.html"

author:
 -
    fullname: Yaroslav Rosomakho
    organization: Zscaler
    email: yrosomakho@zscaler.com
 -
    fullname: Tommy Pauly
    organization: Apple
    email: tpauly@apple.com

normative:

informative:
  IEEE802.3: DOI.10.1109/IEEESTD.2022.9844436
  IEEE802.1Q: DOI.10.1109/IEEESTD.2022.10004498
  LINUX-CHECKSUM-OFFLOAD:
    title: "Checksum Offloads"
    author:
      - org: "The Linux Kernel Development Community"
    target: https://docs.kernel.org/networking/checksum-offloads.html
  LINUX-TUN-TAP:
    title: "Linux v7.2 TUN/TAP userspace API definitions"
    author:
      - org: "The Linux Kernel Development Community"
    target: https://raw.githubusercontent.com/torvalds/linux/v7.2/include/uapi/linux/if_tun.h

--- abstract

This document defines extensions for HTTP Datagram-based protocols that improve transmission efficiency by introducing templates for compressing or deriving datagram fields.

These templates allow endpoints to define parts of datagrams that are static and can be removed, and other parts that can be derived (such as packet lengths and checksum values).

Additionally, this document defines a checksum offload procedure enabling receivers to complete Internet checksums using sender-provided partial values.

These optimisations reduce per-packet overhead and processing cost. Compression can allow larger original packets to fit in QUIC DATAGRAM frames, depending on the applicable processing context and path maximum transmission unit (MTU).

--- middle

# Introduction

The CONNECT-IP method {{!CONNECT-IP=RFC9484}} allows an HTTP client to establish an IP tunnel through an HTTP proxy and exchange IP packets using either HTTP/3 Datagrams ({{Section 2.1 of !HTTP-DATAGRAMS=RFC9297}}) or DATAGRAM capsules ({{Section 3.5 of HTTP-DATAGRAMS}}). Similarly, CONNECT-ETHERNET {{!CONNECT-ETHERNET=I-D.ietf-masque-connect-ethernet}} allows sending Ethernet frames over HTTP Datagrams. These protocols send complete packets or frames by default, including all transport and network headers. This is a simple approach, but incurs per-packet overhead due to the repeated transmission of largely invariant header fields.

Other HTTP Datagram-based protocols share similar properties: datagrams often contain structured packets where many header fields remain constant across a flow while only a subset of bytes change between packets. Transmitting complete packets therefore wastes bandwidth and processing resources.

This document introduces a set of optional extensions that define Processing Contexts for HTTP Datagram payloads. A Processing Context describes transformations applied to a received datagram payload prior to delivery to the target protocol and may reference a parent context, forming a processing chain.

The processing context framework is extensible and intended for use with HTTP Datagram-based protocols in general. Reusable templates operate on byte regions without requiring the receiver to identify the protocol carried in the payload. For example, templates can compress application protocol fields that remain constant across UDP payloads carried by CONNECT-UDP {{CONNECT-UDP}}.

Reusable templates allow endpoints to associate a Context Identifier with a reusable packet layout consisting of static and variable byte regions. Once a template has been installed using reliable Capsules, datagrams referencing the same Context Identifier carry only the variable portions of the packet. This reduces the size of transmitted datagrams and processing overhead, while remaining compatibile with intermediaries that are unaware of these optimisations.

Derived field processing allows the receiver to reconstruct certain header fields (for example packet length fields and complete checksums) based on the size and structure of the reconstructed packet. This eliminates the need for the sender to transmit such fields for every packet.

Unlike reusable templates, Derived Field Types depend on the payload format. The initial types defined in {{derived-field-definitions}} apply to CONNECT-IP and CONNECT-ETHERNET. Additional types can be registered in {{iana-derived-fields}} to support other payload formats and HTTP Datagram-based protocols, with specifications defining how the relevant fields are located and reconstructed.

In addition, this document defines a checksum offload procedure enabling endpoints to cooperatively compute Internet checksums, where the sender provides a partial checksum and the receiver completes the computation after reconstruction. This mirrors hardware checksum-offload behavior used on network interfaces and tunnel devices, reducing per-packet CPU cost for encapsulating or decapsulating CONNECT-IP and CONNECT-ETHERNET traffic.

When HTTP Datagrams are encapsulated in QUIC DATAGRAM frames, compression can allow larger original packets to fit within the same path maximum transmission unit (MTU). The benefit depends on the processing context applicable to each packet and might not provide a uniform increase in tunnel MTU; see {{mtu-considerations}}.

All extensions are negotiated during the HTTP request/response handshake and signalled using Capsules on the reliable control stream. Endpoints can fall back to transmitting datagrams without these optimisations using the Context Identifier appropriate to the underlying HTTP Datagram protocol or extension. This is Context Identifier 0 for protocols that use it to carry unoptimised datagrams, but can also be a Context Identifier defined by another mechanism.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Where this document defines capsule formats, it uses the notation from {{Section 1.3 of !QUIC=RFC9000}}. Where fields within capsules are integers, they are encoded using the variable-length integer encoding from {{Section 16 of QUIC}}. Integer values do not need to be encoded on the minimum number of bytes necessary.

The following terms are used in this document:

Context Identifier (Context ID):

: A numeric identifier that determines how an HTTP Datagram payload is interpreted. Context ID encoding and allocation follow the rules defined in {{Section 4 of !CONNECT-UDP=RFC9298}}. A Context ID can identify a Processing Context or a context defined by the underlying HTTP Datagram protocol or another extension.

Processing Context:

: A set of rules describing how an HTTP Datagram payload is transformed before delivery to the target protocol. A Processing Context may reference a parent context, forming a processing chain. Processing Contexts are immutable once created.

Terminal Context:

: The context at which a processing chain ends. It is either Context ID 0 or a context defined by another mechanism, rather than by an *_ASSIGN capsule defined in this specification. The reconstructed payload is interpreted according to the semantics of this context.

Template:

: A reusable packet layout consisting of a sequence of static and variable segments. Static segments contain bytes removed from optimized datagrams, while variable segments correspond to bytes still carried in the datagram payload.

Derived Field:

: A header field whose value is generated by the receiver during reconstruction and written into the reconstructed packet rather than being transmitted in the datagram payload. Derived fields include length fields and complete checksums.

Checksum Offload:

: A capability allowing the receiver to complete the Internet checksum according to {{!INTERNET-CHECKSUM=RFC1071}} using a sender-provided partial checksum after reconstruction of the packet.

Capsule:

: A reliable control-stream message, as defined in {{Section 3 of HTTP-DATAGRAMS}}, used in this specification to signal creation, acknowledgement, or deletion of Processing Contexts.

# Negotiation of Capabilities {#negotiation}

Endpoints negotiate support for HTTP Datagram processing contexts during the HTTP request/response handshake by using the `http-datagram-contexts` HTTP header field, whose value is a Structured Field Dictionary as defined in {{Section 3.2 of !STRUCTURED-HTTP=RFC9651}}.

## Header Definition

~~~
http-datagram-contexts = sf-dictionary
~~~
{: #fig-http-datagram-contexts-header title="http-datagram-contexts header field"}

This document defines the following optional dictionary keys:

`max-templates` (Integer):

: Maximum number of concurrently active template contexts the sender is willing to maintain for templates created by the peer. Absence of this key or value of 0 indicates that the sender does not support reusable templates.

`max-templates-segments` (Integer):

: Maximum number of static segments accepted within a single template. Absence of this key or value of 0 indicates that the sender does not impose a limit on number of static segments in a single template.

`derived` (Inner List):

: A list of supported Derived Field Types as defined in {{iana-derived-fields}}. Initial field definitions and reconstruction procedures are specified in {{derived-field-definitions}}.

`checksum` (Boolean):

: Indicates support for the checksum offload procedure defined in this document. A value of ?1 means the endpoint is willing to complete checksums using sender-provided partial values. If omitted or set to ?0, checksum offload is not supported.

`mtu` (Integer):

: Upper limit on reconstructed packet size the receiver is willing to accept, in bytes. This is a receiver limit, not a measurement of the path MTU or a guarantee that a packet of this size can be carried in a QUIC DATAGRAM frame.

Endpoints MUST ignore unknown dictionary members. The absence of a member implies that the corresponding capability is not supported for contexts created by the peer.

## Negotiation Behavior

Capabilities are directional. Each endpoint advertises the processing contexts it is willing to receive and maintain for datagrams sent by the peer. An endpoint MAY create a context only if the peer advertised support for the corresponding capability.

### Templates

If the peer advertises the `max-templates`  value greater than 0, the endpoint MAY create template contexts up to that limit using capsules defined in {{capsules}}.

An endpoint MUST NOT create templates exceeding the peer's advertised `max-template-segments` limit when that parameter is present.

If the peer advertises an `mtu` limit, the sender MUST NOT transmit a datagram that would reconstruct into a packet larger than the advertised limit after all processing contexts have been applied.

This limit applies whether the datagram is carried in a QUIC DATAGRAM frame or a DATAGRAM Capsule, including when no processing context is used.

### Derived Fields

An endpoint MAY create a derived context only if every operation in the capsule appears in the peer's `derived` list.

### Checksum Offload

An endpoint MAY create a checksum offload context only if the peer advertised `checksum=?1`.

## Example

HTTP/3 sample request (client to proxy):

~~~
:method = CONNECT
:protocol = connect-ip
:scheme = https
:path = /.well-known/masque/ip/*/*/
:authority = proxy.example.net
capsule-protocol = ?1
http-datagram-contexts = max-templates=20000, max-templates-segments=32, derived=(0 2 4), checksum=?1, mtu=1500
~~~
{: #fig-connect-ip-http-datagram-contexts-request-example title="CONNECT-IP with http-datagram-contexts request example"}

HTTP/3 sample response (proxy to client):

~~~
:status = 200
capsule-protocol = ?1
http-datagram-contexts = max-templates=65535, derived=(0 1), checksum=?0, mtu=1500
~~~
{: #fig-connect-ip-http-datagram-contexts-response-example title="CONNECT-IP with http-datagram-contexts response example"}

In this example, both peers support reusable templates. The proxy supports a subset of derived fields (ipv4-total-length, ipv4-udp-length and ipv4-header-checksum) and the checksum offload. The client supports a different subset of derived fields (ipv4-total-length and ipv6-payload-length) without the checksum offload. Both endpoints indicate that the maximum packet size after reconstruction must not exceed 1500 bytes.

# Processing Context Capsules {#capsules}

This specification defines multiple capsule types to construct, acknowledge, and delete processing contexts.

## Processing Context Overview

### Processing Context Construction {#assign}

Processing contexts are created using capsules that define a new unique non-zero `Context ID` encoded as a variable-length integer. A Context ID MUST NOT be reused, including one allocated by another mechanism on the same HTTP request. As specified in {{Section 4 of CONNECT-UDP}}, even-numbered Context IDs are allocated by the client and odd-numbered by the proxy.

Each processing context references a next context using `Next Context ID` encoded as a variable-length integer. This MUST identify Context ID 0, a previously defined Processing Context, or a context already established by another mechanism. The referenced context MUST be available for use in the same sending direction. Forward references are not permitted. If `Next Context ID` identifies a Processing Context, the chain continues through that context. Otherwise, it identifies the Terminal Context and ends the processing chain.

A processing chain MUST contain at most one Template context (created by TEMPLATE_ASSIGN), at most one Derived Field context (created by DERIVED_ASSIGN), and at most one Checksum Offload context (created by CHECKSUM_ASSIGN). Other processing context chains can also have invalid combinations, if they include contradictory or mutually exclusive steps. Any capsule definition used for these contexts needs to define such cases if they are possible (such as for checksum offload contexts, {{checksum}}). A receiver that detects such an invalid processing chain condition MUST treat the context as malformed and follow the error-handling procedure defined in {{Section 3.3 of HTTP-DATAGRAMS}}.

A receiver of an *_ASSIGN capsule with an invalid `Context ID` or unknown `Next Context ID` MUST treat it as malformed and follow the error-handling procedure defined in {{Section 3.3 of HTTP-DATAGRAMS}}.

### Processing Context Acknowledgement {#ack}

For each *_ASSIGN capsule received, the receiver MUST transmit the corresponding *_ACK capsule after successfully installing the context.

Endpoints MAY transmit datagrams referencing contexts prior to receiving the *_ACK. A receiver MAY buffer datagrams referencing unknown Context IDs but MUST bound buffering by time and memory.

A receiver of an *_ACK capsule with an unknown Context ID, a Context ID not assigned by the corresponding *_ASSIGN capsule, or any data after Context ID MUST treat it as malformed and follow the error-handling procedure defined in {{Section 3.3 of HTTP-DATAGRAMS}}.

### Processing Context Closure {#close}

Processing Contexts are retired by sending corresponding *_CLOSE capsule. Closing a context implicitly closes all contexts that reference it directly or transitively.

Closing a Terminal Context defined by another mechanism also closes all Processing Contexts that reference it directly or transitively. The closure and any retention of the Terminal Context are governed by the mechanism that defined it. Closing a Processing Context does not close its Terminal Context.

A receiver of a *_CLOSE capsule SHOULD retain the closed context and its descendants for a short period to allow in-flight datagrams to arrive, but MUST bound the retention time and memory usage.

*_CLOSE capsules with an unknown Context ID, a Context ID not assigned by the corresponding *_ASSIGN capsule, or any data after Context ID MUST be treated as malformed. Receiver of such capsules MUST follow the error-handling procedure defined in {{Section 3.3 of HTTP-DATAGRAMS}}.

## Template Capsules

### TEMPLATE_ASSIGN Capsule {#template-assign}

~~~
TEMPLATE_ASSIGN Capsule {
  Type (i) = 0x3ee3143f,
  Length (i),
  Context ID (i),
  Next Context ID (i),
  Static Segment (..) ...
}
~~~
{: #fig-template-assign-capsule title="TEMPLATE_ASSIGN Capsule Format"}

The TEMPLATE_ASSIGN capsule contains a sequence of one or more Static Segments.

~~~
Static Segment {
  Segment Offset (i),
  Segment Length (i),
  Segment Payload (..),
}
~~~
{: #fig-static-segment title="Static Segment Format"}

Each Static Segment contains following fields:

Segment Offset:

: Byte offset from the start of the reconstructed packet, encoded as a variable-length integer

Segment Length:

: Length of the Segement Payload field, encoded as a variable-length integer

Segment Payload:

: Static bytes to insert at the Segment Offset

#### Parsing and validation

The receiver parses a TEMPLATE_ASSIGN capsule by reading, in order: the Context ID, the Next Context ID, and one or more static segments whose encodings consume exactly the remaining length of the capsule. Context ID and Next Context ID processing is described in {{assign}}.

Each Static Segment consists of a Segment Offset, a Segment Length, and exactly Segment Length octets of Segment Payload. Static segments MUST appear in strictly increasing Segment Offset order and MUST NOT overlap. There MUST be at least 1 byte between consecutive segments.

A receiver that advertised a `max-templates-segments` limit MUST ensure that the template does not contain more static segments. A receiver that advertised a `mtu` limit in `http-datagram-contexts` MUST ensure that the sum of Segment Offset and Segment Length of the final segment does not exceed the MTU limit. Final reconstructed packet size validation is performed during packet reconstruction ({{reconstruction}}). The capsule MUST end immediately after the last static segment.

If any of the capsule fields are malformed upon reception, the receiver of the capsule MUST follow the error-handling procedure defined in {{Section 3.3 of HTTP-DATAGRAMS}}.

A receiver that has already accepted the maximum number of templates it advertised via the `max-templates` member in `http-datagram-contexts` MUST treat any additional TEMPLATE_ASSIGN capsule an error and MUST follow the same error-handling procedure.

Per-packet validation uses the reconstruction procedure described in {{reconstruction}}.

### TEMPLATE_ACK Capsule

~~~
TEMPLATE_ACK Capsule {
  Type (i) = 0x3ee31440,
  Length (i),
  Context ID (i),
}
~~~
{: #fig-template-ack-capsule title="TEMPLATE_ACK Capsule Format"}

Processing of the TEMPLATE_ACK capsule is described in {{ack}}

### TEMPLATE_CLOSE Capsule

~~~
TEMPLATE_CLOSE Capsule {
  Type (i) = 0x3ee31441,
  Length (i),
  Context ID (i),
}
~~~
{: #fig-template-close-capsule title="TEMPLATE_CLOSE Capsule Format"}

Processing of the TEMPLATE_CLOSE capsule is described in {{close}}

## Derived Field Capsules

### DERIVED_ASSIGN Capsule

~~~
DERIVED_ASSIGN Capsule {
  Type (i) = 0x3ee31442,
  Length (i),
  Context ID (i),
  Next Context ID (i),
  Derived Field Type (i) ...
}
~~~
{: #fig-derived-assign-capsule title="DERIVED_ASSIGN Capsule Format"}

The DERIVED_ASSIGN capsule defines a processing context that generates and inserts one or more derived fields into the reconstructed packet. The sender does not transmit these fields in the datagram payload.

The Derived Field Types defined by this document and their reconstruction procedures are specified in {{derived-field-definitions}}. The numeric code points for these field types are registered in {{iana-derived-fields}}.

#### Parsing and validation

The receiver parses a DERIVED_ASSIGN capsule by reading, in order: the Context ID, the Next Context ID, and one or more Derived Field Type values encoded as variable-length integers. Context ID and Next Context ID processing is described in {{assign}}.

If a Derived Field Type is not present in the receiver's advertised `derived` capability list in `http-datagram-contexts` or if any Derived Field Type appears more than once in the capsule, the receiver MUST treat the capsule as malformed and follow the error-handling procedure defined in {{Section 3.3 of HTTP-DATAGRAMS}}.

Per-packet validation uses the reconstruction procedure described in {{reconstruction}}.

### DERIVED_ACK Capsule

~~~
DERIVED_ACK Capsule {
  Type (i) = 0x3ee31443,
  Length (i),
  Context ID (i),
}
~~~
{: #fig-derived-ack-capsule title="DERIVED_ACK Capsule Format"}

Processing of the DERIVED_ACK capsule is described in {{ack}}

### DERIVED_CLOSE Capsule

~~~
DERIVED_CLOSE Capsule {
  Type (i) = 0x3ee31444,
  Length (i),
  Context ID (i),
}
~~~
{: #fig-derived-close-capsule title="DERIVED_CLOSE Capsule Format"}

Processing of the DERIVED_CLOSE capsule is described in {{close}}

## Checksum Offload Capsules {#checksum}

### CHECKSUM_ASSIGN Capsule

~~~
CHECKSUM_ASSIGN Capsule {
  Type (i) = 0x3ee31445,
  Length (i),
  Context ID (i),
  Next Context ID (i),
  Checksum Field Offset (i),
  Checksum Start Offset (i),
}
~~~
{: #fig-checksum-assign-capsule title="CHECKSUM_ASSIGN Capsule Format"}

The CHECKSUM_ASSIGN capsule defines a processing context that completes an Internet checksum for the reconstructed packet using a sender-provided partial checksum.

In addition to `Context ID` and `Next Context ID` CHECKSUM_ASSIGN capsule contains following fields encoded as variable-length integers:

`Checksum Field Offset`:

: Byte offset of the 16-bit Internet checksum field within the reconstructed packet

`Checksum Start Offset`:

: Byte offset where checksum coverage begins.  Coverage runs from this offset to the end of the reconstructed packet.

#### Parsing and validation

The receiver parses a CHECKSUM_ASSIGN capsule by reading, in order: the Context ID, the Next Context ID, Checksum Field Offset and Checksum Start Offset. Context ID and Next Context ID processing is described in {{assign}}.

If the peer did not advertise `checksum=?1` in `http-datagram-contexts`, the receiver MUST treat the capsule as malformed and follow the error-handling procedure defined in {{Section 3.3 of HTTP-DATAGRAMS}}.

If `Checksum Start Offset` is 0, the receiver MUST treat the capsule as malformed and follow the same error-handling procedure.

Per-packet validation uses the reconstruction procedure described in {{reconstruction}}.

Chaining a CHECKSUM_ASSIGN capsule with a DERIVED_ASSIGN capsule that defines derivation for a UDP or TCP checksum ({{iana-derived-fields}}) would create an invalid processing chain, as described in {{assign}}. Such chains MUST NOT be defined by senders, and MUST be treated as an error by receivers.

### CHECKSUM_ACK Capsule

~~~
CHECKSUM_ACK Capsule {
  Type (i) = 0x3ee31446,
  Length (i),
  Context ID (i),
}
~~~
{: #fig-checksum-ack-capsule title="CHECKSUM_ACK Capsule Format"}

Processing of the CHECKSUM_ACK capsule is described in {{ack}}

### CHECKSUM_CLOSE Capsule

~~~
CHECKSUM_CLOSE Capsule {
  Type (i) = 0x3ee31447,
  Length (i),
  Context ID (i),
}
~~~
{: #fig-checksum-close-capsule title="CHECKSUM_CLOSE Capsule Format"}

Processing of the CHECKSUM_CLOSE capsule is described in {{close}}

# Processing Context Operation

This section defines how endpoints construct and consume HTTP Datagram payloads using Processing Contexts.

A datagram carries a Context Identifier that selects either a Processing Context or a Terminal Context. A processing chain is resolved by following Next Context ID through contexts defined by the *_ASSIGN capsules in this specification until a Terminal Context is reached.

After applying the Processing Contexts, the reconstructed payload is delivered using the Terminal Context ID and its associated semantics. A datagram that directly selects a Terminal Context is handled according to those semantics without applying the transformations defined in this specification.

For example, if another extension has established Context ID 2, a TEMPLATE_ASSIGN capsule can define Context ID 4 with Next Context ID 2. A datagram received with Context ID 4 is reconstructed using that template and then handled as a datagram with Context ID 2.

## Sender behavior

When sending a datagram using a Processing Context, the sender constructs the payload so that the receiver can reconstruct the payload expected by the Terminal Context after applying the processing chain. The Terminal Context ID is conveyed through the chain's final Next Context ID and is not included in the payload being transformed.

The sender MUST use a Processing Context ID only after the corresponding *_ASSIGN capsule has been transmitted.

### Template Contexts

If the selected context chain contains a Template context, the sender constructs the datagram payload as the concatenation of all variable byte regions not covered by static segments.

Variable regions are emitted in strictly increasing offset order starting at offset 0.

If the context chain contains no Template context, the payload MUST be the complete packet.

### Derived Field Contexts

Derived fields are not transmitted by the sender. When a derived context is in use, the sender MUST remove the octets corresponding to derived fields from the datagram payload. These octets are supplied by the receiver during reconstruction.

The sender MUST construct the payload as if the derived field octets were not part of the variable regions. That is, the payload MUST contain only the remaining variable octets in strictly increasing offset order.

### Checksum Offload Contexts

If the context chain contains a checksum offload context, the sender MUST place a precomputed partial Internet checksum value into the checksum field at Checksum Field Offset in the reconstructed packet image prior to transmission. This value is combined with the receiver computation as described in {{reconstruction}}.

The partial value MUST be the 16-bit one's-complement sum of the checksum contributions not included in the receiver's checksum coverage, such as the pseudo-header. The sender MUST fold any carries into the low 16 bits without complementing the sum and store the value in network byte order.

Checksum offload reduces per-packet CPU cost by allowing the sender to preserve a packet's partial checksum instead of completing the checksum before encapsulation. The partial value includes the pseudo-header contribution and is carried in the variable portion of the datagram, with Checksum Start Offset pointing to the transport header. After any template reconstruction and derived field processing, the receiver can delegate checksum completion to a capable NIC {{LINUX-CHECKSUM-OFFLOAD}} or pass the reconstructed packet with checksum-offload metadata through an operating-system interface, such as Linux TUN/TAP {{LINUX-TUN-TAP}}.

### Context Selection

If a packet does not match any available Processing Context, the sender MUST use the appropriate Terminal Context ID and transmit the complete payload required by that context, subject to the size limits and oversized-packet handling described in {{mtu-considerations}}.

To reduce variation in compression across packets, senders SHOULD define broad contexts that will be able to handle most or all expected traffic.

### MTU Considerations {#mtu-considerations}

The advertised `mtu` bounds the reconstructed packet size. A packet that would not fit in a QUIC DATAGRAM frame without compression can still be sent if compression makes it fit. The transport size checks apply after compression: the resulting QUIC DATAGRAM frame MUST respect the peer's frame size limit, and the outer IP packet, including HTTP/3, QUIC, UDP, and IP overhead, MUST fit within the current path MTU. Even packets below the advertised `mtu` can exceed these transport limits when compression is insufficient or no matching context is available.

Compression can differ between packets of the same size, including between PMTU probes and other traffic. A successful probe therefore does not establish that all packets of that original size will fit. PMTU information can also be shared across flows that use different contexts; see the discussion of shared PLPMTU state in item 9 of {{Section 3 of ?DPLPMTUD=RFC8899}}. When choosing a tunnel MTU or reporting a PMTU to carried traffic, endpoints SHOULD account for the least compression available to the traffic sharing that MTU, including packets sent without compression, rather than assuming the savings of a particular context.

Oversized packets are handled according to the underlying HTTP Datagram protocol's MTU and error-signalling rules. In particular, {{Section 10.1 of CONNECT-IP}} and {{Section 6.1 of CONNECT-UDP}} recommend dropping oversized packets and sending ICMP Packet Too Big feedback instead of switching to DATAGRAM Capsules. Where permitted by the underlying protocol, DATAGRAM Capsules can carry larger datagrams across multiple transport packets, but introduce reliable delivery, head-of-line blocking, additional buffering and retransmission costs, and possible reordering relative to QUIC DATAGRAM frames. Such delivery can hide size limitations from PMTU discovery for the carried traffic; it does not increase the path MTU or relax the advertised `mtu` limit.

### Conservative Template Composition {#template-composition}

Reusable templates are most effective when they remain valid for many datagrams. This guidance applies to the packets using a particular template. A template can be used for a subset of the HTTP Datagrams associated with an HTTP request. Senders SHOULD construct templates conservatively by including only fields that are expected to remain stable across applicable packets for the lifetime of the Processing Context. Fields whose values can change across packets SHOULD remain in the variable portion of the datagram unless the sender has specific knowledge that those fields will remain constant or be particularly frequent for some period of time.

Fields that are commonly suitable for static segments include:

* Ethernet source and destination addresses, when stable for the frame sequence
* EtherType
* 802.1Q tag
* IP version
* Source and destination IP addresses
* IPv4 TTL / IPv6 Hop Limit
* IPv4 Protocol / IPv6 Next Header
* Source and destination transport ports

Fields that commonly vary between packets, or that can be modified by endpoints or network devices, are generally poor candidates for static segments. Examples include:

* IPv4 DSCP and ECN bits, and IPv6 Traffic Class
* IPv4 Identification
* IPv4 Flags and Fragment Offset, unless packets using the template are known not to be fragmented
* IPv6 Flow Label, unless the sender knows it has the same value in all packets using the template
* TCP Sequence Number
* TCP Acknowledgement Number
* TCP Flags
* TCP Window
* TCP Timestamp values
* Packet length fields
* Checksum fields

When a field can be represented as a Derived Field defined by this specification, implementations SHOULD prefer using the corresponding Derived Field instead of including that field in a static segment or transmitting it as part of the variable payload. Packet length fields and complete checksums are examples of fields that are generally better represented using Derived Fields when supported by the peer.

The suitability of a field for inclusion in a static segment depends on the deployment and the traffic being carried. For example, some deployments might preserve DSCP values while others rewrite them, and some applications might intentionally vary the IPv6 Flow Label. Implementations are encouraged to prefer templates that remain valid across many datagrams, even if those templates omit fewer bytes than a more aggressive template.

## Receiver behavior {#reconstruction}

Upon receiving an HTTP Datagram whose Context ID identifies a Processing Context, the receiver retrieves that context and follows Next Context ID through any further Processing Contexts. The receiver MUST stop resolving the processing chain when it reaches Context ID 0 or a context known to have been defined by another mechanism. The receiver MUST retain this Terminal Context ID for delivery of the reconstructed payload.

If any referenced context is unknown, the receiver MAY buffer the datagram as described in {{ack}} or drop it. An unknown Context ID MUST NOT be treated as a valid Terminal Context merely because it does not identify an installed Processing Context.

If multiple processing contexts are present in a chain, the receiver MUST apply them in the following order:

1. Template reconstruction (if present)
1. Derived field processing (if present)
1. Checksum offload processing (if present)

After reconstruction, the receiver MUST handle the resulting payload according to the Terminal Context's semantics, including any validation required by the underlying protocol or extension.

### Template Reconstruction

If a Template context is present, the receiver reconstructs the packet as follows:

1. Allocate a buffer large enough to contain the reconstructed packet.
1. Insert static segment bytes at their specified offsets.
1. Fill all remaining gaps using bytes from the datagram payload in strictly increasing offset order.

If payload bytes are exhausted before all gaps have been filled the datagram MUST be dropped.

Packets larger than the advertised `mtu` in `http-datagram-contexts` MUST be dropped.

### Derived Field Processing

For each derived field present in the context chain, the receiver computes the field value and inserts it into the reconstructed packet at the location defined by the derived field type.

Derived fields are inserted into the packet image and therefore increase the reconstructed packet size. The receiver MUST compute derived field values based on the final reconstructed packet size and structure.

Initial field definitions and reconstruction procedures are specified in {{derived-field-definitions}}.

If the required header cannot be located, the packet MUST be dropped.

### Checksum Offload Processing

If a checksum offload context is present, the receiver completes the Internet checksum after all derived fields have been inserted.

The receiver MUST complete the checksum as follows, using one's-complement arithmetic as described in {{INTERNET-CHECKSUM}}:

1. Read and retain the 16-bit partial sum from the checksum field in network byte order.
1. Compute the one's-complement sum of the octets from Checksum Start Offset to the end of the reconstructed packet, treating the checksum field as zero for this computation. Pair octets into 16-bit words in network byte order; if the number of octets is odd, pad the final octet with a zero octet for the computation only.
1. Add the retained partial sum to this sum using end-around carry: repeatedly add any bits above the low 16 bits back into the low 16 bits until no carry remains.
1. Take the bitwise one's complement of the resulting 16-bit sum and write it to the checksum field in network byte order. For a UDP checksum, encode a zero result as 0xffff.

If any offset exceeds the reconstructed packet length, the packet MUST be dropped.

# Derived Field Definitions {#derived-field-definitions}

This section defines the initial Derived Field Types registered in {{iana-derived-fields}}. The common processing rules and header checks below apply together with each field's definition. The IPv4 and IPv6 variants of UDP Length, TCP Checksum, and UDP Checksum share their respective field locations and computation procedures.

All offsets and lengths in this section are expressed in octets. Field tables specify offsets relative to the indicated header or frame in the reconstructed packet. All multi-octet integer fields are encoded in network byte order.

## Common Processing Rules {#derived-common-processing}

### Sender Behavior

Before using a Derived Field Type, the sender MUST perform the applicable header checks in {{derived-header-validation}} and any additional checks in the field's definition. If a required header or field cannot be located, or an applicability check fails, the sender MUST NOT use that Derived Field Type for the packet.

The sender MUST verify that the original field value equals the value that the receiver would generate using the computation in the field's definition for the reconstructed packet. This comparison includes any specified encoding of special values, such as encoding a zero UDP checksum as 0xffff. If the values differ, the sender MUST NOT use that Derived Field Type for the packet.

When using a Derived Field Type, the sender removes the corresponding field octets from the datagram payload.

### Receiver Behavior

The receiver MUST perform the applicable header checks in {{derived-header-validation}} and the field-specific checks below. If the packet is too short to contain a required header or field, a required header cannot be located, a validation check fails, or the computed value is outside the range specified for the field, the receiver MUST drop the packet.

The receiver computes and inserts length fields before checksum fields, using the final reconstructed packet size and structure as described in {{reconstruction}}. In particular, IPv4 Total Length, IPv6 Payload Length, and UDP Length are computed before IPv4 header, TCP, or UDP checksums. The receiver writes each computed value at the location specified in the field's definition.

## Header Location and Validation {#derived-header-validation}

Unless a check explicitly applies only to the sender or receiver, the checks in this section apply to both endpoints. The sender applies them to the original packet and the receiver to the reconstructed packet, with failure handled as specified in {{derived-common-processing}}.

### IP Headers {#derived-ip-headers}

For CONNECT-IP, the IP header starts at offset 0. For CONNECT-ETHERNET, the IP- and transport-layer Derived Field Types apply to Ethernet II frames carrying IPv4 or IPv6, including frames with a single IEEE 802.1Q VLAN tag:

Encapsulation | EtherType offset | IP header offset
--- | --- | ---
Ethernet II, untagged {{IEEE802.3}} | 12 | 14
Ethernet II, single IEEE 802.1Q VLAN tag {{IEEE802.1Q}} | 16 | 18

These offsets are measured from the beginning of the Ethernet frame. For the tagged form, the Tag Protocol Identifier (TPID) at offset 12 has value 0x8100. Frames with multiple VLAN tags or other link-layer encapsulations are not processed by these Derived Field Types unless a future specification defines how the relevant network-layer header is located.

The sender and receiver MUST locate the IP header and verify that its first four bits contain the version required by the Derived Field Type: 4 for IPv4 {{?IPv4=RFC791}} or 6 for IPv6 {{?IPv6=RFC8200}}. For CONNECT-ETHERNET, the EtherType MUST also identify that version: 0x0800 for IPv4 or 0x86dd for IPv6.

For all IPv4 Derived Field Types, the receiver MUST verify that the Internet Header Length (IHL) field indicates a header length of at least 20 octets and no larger than the reconstructed IPv4 packet. The sender's IHL check for IPv4 Header Checksum is specified in {{derived-ipv4-header-checksum}}.

### Transport Headers {#derived-transport-headers}

The following checks apply to UDP Length, TCP Checksum, and UDP Checksum. The required transport protocol and minimum header size are:

Derived field | Transport protocol number | Minimum header size
--- | --- | ---
UDP Length or UDP Checksum | 17 (UDP) | 8
TCP Checksum | 6 (TCP) | 20

For an IPv4 variant, the IPv4 Protocol field MUST identify the required transport protocol. The packet MUST NOT be fragmented: the More Fragments flag and Fragment Offset field MUST both be zero. The transport header starts immediately after the IPv4 header, whose length is determined by IHL.

For an IPv6 variant, the sender and receiver MUST locate the required transport header by traversing the IPv6 Next Header chain from the fixed 40-octet IPv6 header. The traversal follows extension headers that identify the following header using a Next Header field and whose length can be determined from the packet contents. If traversal encounters an IPv6 Fragment header before the required transport header, or a header whose following header cannot be determined, the required header is considered not located.

The IP packet MUST contain at least the minimum header size at the resulting transport header offset. For TCP, the Data Offset field MUST indicate a header length of at least 20 octets and no larger than the TCP segment contained in the IP packet. The TCP header length is the Data Offset value multiplied by 4.

## IPv4 Total Length {#derived-ipv4-total-length}

The `ipv4-total-length` field corresponds to the Total Length field of the IPv4 header defined in {{Section 3.1 of IPv4}}. It uses the IPv4 header checks in {{derived-ip-headers}}.

Field | Offset from start of IPv4 header | Length
--- | --- | ---
Total Length | 2 | 2

The value is the number of octets from the start of the IPv4 header to the end of the reconstructed IPv4 packet. It cannot exceed 65535.

## IPv6 Payload Length {#derived-ipv6-payload-length}

The `ipv6-payload-length` field corresponds to the Payload Length field of the IPv6 header defined in {{Section 3 of IPv6}}. It uses the IPv6 header checks in {{derived-ip-headers}}.

Field | Offset from start of IPv6 header | Length
--- | --- | ---
Payload Length | 4 | 2

The value is the number of octets following the fixed 40-octet IPv6 header in the reconstructed IPv6 packet. Its permitted range is 1 to 65535.

This Derived Field Type does not support IPv6 jumbograms ({{?IPv6-JUMBO=RFC2675}}). If the IPv6 Payload Length field in the original packet is zero, the sender MUST NOT use this Derived Field Type.

## UDP Length {#derived-udp-length}

The two UDP Length types derive the Length field of the UDP header defined in {{?UDP=RFC768}}. Both use the IP header checks in {{derived-ip-headers}} and the UDP header checks in {{derived-transport-headers}} for their respective IP version.

Field | Offset from start of UDP header | Length
--- | --- | ---
Length | 4 | 2

The value is the number of octets from the start of the UDP header to the end of the reconstructed IP packet. Its permitted range is 8 to 65535. The sender's comparison in {{derived-common-processing}} therefore requires the original UDP Length to cover all of these octets; otherwise, this derivation cannot be used.

UDP Options ({{Section 7 of ?UDP-OPTIONS=RFC9868}}) occupy a surplus area after the UDP payload but within the IP packet. This check prevents length derivation from incorrectly treating that area as part of the UDP payload.

### UDP Length over IPv4 {#derived-ipv4-udp-length}

The `ipv4-udp-length` type applies this procedure to UDP carried over IPv4.

### UDP Length over IPv6 {#derived-ipv6-udp-length}

The `ipv6-udp-length` type applies this procedure to UDP carried over IPv6. Octets preceding the UDP header, including IPv6 extension headers, are not included in the derived length.

## IPv4 Header Checksum {#derived-ipv4-header-checksum}

The `ipv4-header-checksum` field corresponds to the Header Checksum field of the IPv4 header defined in {{Section 3.1 of IPv4}}. It uses the IPv4 header checks in {{derived-ip-headers}}.

Field | Offset from start of IPv4 header | Length
--- | --- | ---
Header Checksum | 10 | 2

The sender MUST verify that IHL indicates an IPv4 header length of at least 20 octets and no larger than the original IPv4 packet.

The value is the Internet checksum computed over the IPv4 header, whose length is determined by IHL, with the Header Checksum field treated as zero for the computation.

## TCP Checksum {#derived-tcp-checksum}

The two TCP Checksum types derive the Checksum field of the TCP header defined in {{Section 3.1 of ?TCP=RFC9293}}. Both use the IP header checks in {{derived-ip-headers}} and the TCP header checks in {{derived-transport-headers}} for their respective IP version.

Field | Offset from start of TCP header | Length
--- | --- | ---
Checksum | 16 | 2

The value is the Internet checksum computed over the appropriate IP pseudo-header, the TCP header, and the TCP payload, with the TCP Checksum field treated as zero for the computation.

### TCP Checksum over IPv4 {#derived-ipv4-tcp-checksum}

The `ipv4-tcp-checksum` type applies this procedure to TCP carried over IPv4, using the IPv4 pseudo-header.

### TCP Checksum over IPv6 {#derived-ipv6-tcp-checksum}

The `ipv6-tcp-checksum` type applies this procedure to TCP carried over IPv6, using the IPv6 pseudo-header.

## UDP Checksum {#derived-udp-checksum}

The two UDP Checksum types derive the Checksum field of the UDP header defined in {{UDP}}. Both use the IP header checks in {{derived-ip-headers}} and the UDP header checks in {{derived-transport-headers}} for their respective IP version.

Field | Offset from start of UDP header | Length
--- | --- | ---
Checksum | 6 | 2

The sender and receiver MUST verify that UDP Length is at least 8 and no larger than the number of octets from the start of the UDP header to the end of the IP packet. If UDP Length is also derived, the receiver uses its reconstructed value.

The value is the Internet checksum computed over the appropriate IP pseudo-header, the UDP header, and the UDP payload, with the UDP Checksum field treated as zero for the computation. UDP Length determines the extent of the UDP header and payload covered by this computation. If the computed checksum is zero, the field value is 0xffff; this encoding applies both to the sender's comparison and to the value written by the receiver.

### UDP Checksum over IPv4 {#derived-ipv4-udp-checksum}

The `ipv4-udp-checksum` type applies this procedure to UDP carried over IPv4, using the IPv4 pseudo-header. It always generates a UDP checksum. The sender MUST NOT use this Derived Field Type for IPv4 UDP packets that use a UDP Checksum field value of zero to indicate that no checksum is present.

### UDP Checksum over IPv6 {#derived-ipv6-udp-checksum}

The `ipv6-udp-checksum` type applies this procedure to UDP carried over IPv6, using the IPv6 pseudo-header.

## Ethernet Frame Check Sequence {#derived-ethernet-fcs}

The `ethernet-fcs` field corresponds to the Frame Check Sequence (FCS) field of the Ethernet frame defined in {{IEEE802.3}}. This Derived Field Type applies only to CONNECT-ETHERNET and uses the common processing rules in {{derived-common-processing}}; the IP and transport header checks do not apply.

Field | Offset from start of Ethernet frame | Length
--- | --- | ---
Frame Check Sequence | Final 4 octets of frame | 4

The sender MUST only use this Derived Field Type for Ethernet frames that contain an FCS field. The sender and receiver MUST verify that the frame is long enough to contain the Destination MAC Address, Source MAC Address, EtherType or length field, and FCS field.

The value is the Ethernet Frame Check Sequence computed over the frame starting at the Destination MAC Address field and ending immediately before the FCS field.

# Examples

This section illustrates how contexts are created and how senders form compact payloads. All offsets and lengths are in bits in the packet diagrams and field tables. All offsets and lengths are in bytes in segment tables and sample capsules.

The examples below are illustrative. Implementations are expected to follow {{template-composition}} and select static and variable fields based on the traffic being carried and the stability of those fields in the relevant deployment.

## CONNECT-IP: TCP over IPv6 with template, derived fields and checksum offload

Original sample {{TCP}} over {{IPv6}} packet layout is illustrated below. In addition to basic IPv6 and TCP headers it contains Timestamp option as defined in {{Section 3 of ?TCP-PERF=RFC7323}}.

This packet is to be transmitted from the client to the proxy over CONNECT-IP.

~~~ aasvg
|0              7|8             15|16            23|16            31|
+--------+-------+-------+--------+----------------+----------------+
|0 1 1 0 |     0x00      |                 0x4bcde                  | ^
|Version | Traffic Class |               Flow Label                 | |
+--------+---------------+--------+----------------+----------------+ |
|             0x0020              |      0x06      |      0x79      | I
|         Payload length          |  Next header   |   Hop limit    | P
+---------------------------------+----------------+----------------+
|                                                                   | H
|              2001:0db8:85a3:0000:0000:8a2e:0370:7334              | E
|                          Source Address                           | A
|                                                                   | D
+-------------------------------------------------------------------+ E
|                                                                   | R
|              2001:0db8:a42b:0000:0000:7c3a:143a:1529              | |
|                       Destination Address                         | |
|                                                                   | v
+---------------------------------+---------------------------------+
|             0x0050              |             0xd475              | ^
|           Source port           |        Destination port         | |
+---------------------------------+---------------------------------+ |
|                            0x6caa4bd7                             | |
|                          Sequence number                          | |
+-------------------------------------------------------------------+ |
|                            0x9b16794e                             | T
|                       Acknowledgment number                       | C
+--------+------------------------+---------------------------------+ P
|1 0 0 0 |0 0 0 0 0 0 0 1 0 0 0 0 |             0x041e              |
|Hdr Len |       TCP Flags        |             Window              | H
+--------+------------------------+---------------------------------+ E
|             0x8f6b              |             0x0000              | A
|            Checksum             |         Urgent Pointer          | D
+----------------+----------------+----------------+----------------+ E
|      0x01      |      0x01      |      0x08      |      0x0a      | R
|  No-Op Option  |  No-Op Option  |TimeStamp Option|     Length     | |
+----------------+----------------+----------------+----------------+ |
|                            0x119a5db3                             | |
|                          Timestamp value                          | |
+-------------------------------------------------------------------+ |
|                            0xd9b4d48d                             | |
|                       Timestamp echo reply                        | v
+-------------------------------------------------------------------+
~~~
{: #original-tcp-ipv6 title="Example TCP over IPv6 packet before optimization"}

This example assumes that the peer supports templates with at least two segments per template, IPv6 payload length derived field and checksum offloading. These capabilities were communicated using the following `http-datagram-contexts` HTTP field in proxy response confirming CONNECT-IP extended CONNECT.

~~~
http-datagram-contexts = max-templates=1, max-templates-segments=2, derived=(1), checksum=?1, mtu=1500
~~~
{: #fig-http-datagram-contexts-example-1 title="http-datagram-contexts response example"}

Since the proxy does not support TCP checksum derivation, but it supports checksum offloading, the client calculates the folded, uncomplemented one's-complement sum of the IPv6 pseudo-header and places it in the TCP checksum field. Context for the offloaded checksum is defined using the CHECKSUM_ASSIGN capsule:

~~~
CHECKSUM_ASSIGN Capsule {
  Type (i) = 0x3ee31445,
  Length (i) = 4,
  Context ID (i) = 2,
  Next Context ID (i) = 0,
  Checksum Field Offset (i) = 56,
  Checksum Start Offset (i) = 40,
}
~~~
{: #ipv6-tcp-checksum-assign title="CHECKSUM_ASSIGN Capsule for example IPv6/TCP packet"}

Payload length field in IPv6 header can derived by the peer, so it is removed before calculating static segments. Resulting context for the derived field is defined using DERIVED_ASSIGN capsule:

~~~
DERIVED_ASSIGN Capsule {
  Type (i) = 0x3ee31442,
  Length (i) = 3,
  Context ID (i) = 4,
  Next Context ID (i) = 2,
  Derived Field Type (i) = 1
}
~~~
{: #ipv6-tcp-derived-assign title="DERIVED_ASSIGN Capsule for example IPv6/TCP packet"}

The table below illustrates fields present in IPv6 and TCP headers after derived field was removed, their offsets in bits from the beginning of the packet and whether they are likely to be static for most packets of a given traffic flow

| Offset | Field name | Length | Value | Static |
| --- | --- | --- | --- | --- |
| 0 | Version | 4 | 0110b | Yes |
| 4 | Traffic Class | 8 | 0x00 | Yes |
| 12 | Flow label | 20 | 0x4bcde | Yes |
| 32 | Next header | 8 | 0x06 | Yes |
| 40 | Hop limit | 8 | 0x79 | Yes |
| 48 | Source address | 128 | 2001:0db8:85a3::8a2e:0370:7334 | Yes |
| 176 | Destination address | 128 | 2001:0db8:a42b::7c3a:143a:1529 | Yes |
| 304 | Source port | 16 | 0x0050 | Yes |
| 320 | Destination port | 16 | 0xd475 | Yes |
| 336 | Sequence number | 32 | 0x6caa4bd7 | No |
| 368 | Acknowledgement number | 32 | 0x9b16794e | No |
| 400 | TCP header length | 4 | 1000b | Yes |
| 404 | TCP Flags | 12 | 000000010000b | No |
| 416 | Window | 16 | 0x041e | No |
| 432 | Checksum | 16 | 0x8f6b | No |
| 448 | Urgent pointer | 16 | 0x0000 | Yes |
| 464 | No-Op option | 8 | 0x01 | Yes |
| 472 | No-Op option | 8 | 0x01 | Yes |
| 480 | Timestamp option | 8 | 0x08 | Yes |
| 488 | Timestamp option length | 8 | 0x0a | Yes |
| 496 | Timestamp value | 32 | 0x119a5db3 | No |
| 528 | Timestamp echo reply | 32 | 0xd9b4d48d | No |
{: #ipv6-tcp-fields title="IPv6 and TCP header fields in example packet"}

Static segments model the invariant parts except for the isolated 4-bit TCP header length.

Resulting static segments:

| Segment Offset | Segment Length | Segment Contents | Segment Payload |
| --- | --- | --- | --- |
| 0 | 42 | Version, Traffic Class, Flow Label, Next header, Hop limit, Source address, Destination address, Source port, Destination port | 0x6004bcde067920010db885a3... |
| 56 | 6 | Urgent pointer, 2 No-Op TCP options, Timestamp option code and length | 0x00000101080a |
{: #ipv6-tcp-segments title="Static segments for example IPv6/TCP packet"}

Resulting TEMPLATE_ASSIGN capsule with client-allocated even context id is illustrated below:

~~~
TEMPLATE_ASSIGN Capsule {
  Type (i) = 0x3ee3143f,
  Length (i) = 54,
  Context ID (i) = 6,
  Next Context ID (i) = 4,
  Static Segment {
    Segment Offset (i) = 0,
    Segment Length (i) = 42,
    Segment Payload = 0x6004bcde067920010db885a3000000008a2e0370733420010db8a42b000000007c3a143a15290050d475,
  },
  Static Segment {
    Segment Offset (i) = 56,
    Segment Length (i) = 6,
    Segment Payload = 0x00000101080a,
  }
}
~~~
{: #ipv6-tcp-template-assign title="TEMPLATE_ASSIGN Capsule for example IPv6/TCP packet"}

The resulting processing context chain removes 50 bytes of repeated header material from packets that match the chain, reducing the space they require in QUIC DATAGRAM frames.

The sender concatenates all variable regions in increasing offset order. Packets that do not match this template (for example packets with IPv6 extension headers or without TCP options) are sent using Context ID 0 or associated with a new context.

Upon receiving the datagram with Context ID 6, proxy re-assembles the datagram by concatenating static and variable segments according to the offsets, re-calculates Payload Length and inserts it into IPv6 header and completes the TCP checksum using the sender-provided pseudo-header partial checksum.

## CONNECT-ETHERNET: UDP over IPv4 with template and derived fields

Original sample {{UDP}} over {{IPv4}} Ethernet frame layout is illustrated below.

This frame is to be transmitted from the proxy to the client over CONNECT-ETHERNET.

~~~ aasvg
|0      7|8     15|16    23|24    31|32    39|40    47|
+--------+--------+--------+--------+--------+--------+
|               Destination MAC address               |
|                  00:00:5E:00:53:01                  |
+--------+--------+--------+--------+--------+--------+
|                 Source MAC address                  |
|                  00:00:5E:00:53:02                  |
+--------+--------+--------+--------+--------+--------+
|     0x0800      |
|    EtherType    |            ETHERNET HEADER
+-----------------+

|0              7|8             15|16            23|16            31|
+--------+-------+----------------+----------------+----------------+
|0 1 0 0 |0 1 0 1|      0x02      |             0x04cc              | ^
|Version |Hdr Len| Traffic Class  |          Total length           | |
+--------+-------+----------------+------+--------------------------+ I
|             0x0000              |0 1 0 |0 0 0 0 0 0 0 0 0 0 0 0 0 | P
|         Identification          |Flags |     Fragment offset      |
+----------------+----------------+------+--------------------------+ H
|      0x40      |      0x11      |             0xb21b              | E
|      TTL       |    Protocol    |         Header checksum         | A
+----------------+----------------+---------------------------------+ D
|                             192.0.2.1                             | E
|                          Source Address                           | R
+-------------------------------------------------------------------+ |
|                             192.0.2.2                             | |
|                       Destination Address                         | v
+---------------------------------+---------------------------------+
|             0xc199              |             0x1151              | ^
|           Source port           |        Destination port         | |
+---------------------------------+---------------------------------+ |
|             0x04b8              |             0x72de              | U
|             Length              |            Checksum             | D
+---------------------------------+---------------------------------+ P
|                                                                   | |
|                      UDP payload (1200 bytes)                     | |
|                                ...                                | v
~~~
{: #original-udp-ipv4-ethernet title="Example UDP over IPv4 Ethernet frame before optimization"}

This example assumes that the peer supports templates, IPv4 total length, IPv4 header checksum, UDP length in IPv4 packet and UDP checksum in IPv4 packet derived field. These capabilities were communicated using the following `http-datagram-contexts` HTTP field in client requesting CONNECT-ETHERNET extended CONNECT.

~~~
http-datagram-contexts = max-templates=1, max-templates-segments=1, derived=(0 2 4 7), mtu=1500
~~~
{: #fig-http-datagram-contexts-example-2 title="http-datagram-contexts request example"}

Total length and header checksum in IPv4 header as well as length and checksum in UDP header can be derived by the peer, so these fields are removed before calculating static segments. Resulting context for the derived field is defined using DERIVED_ASSIGN capsule:

~~~
DERIVED_ASSIGN Capsule {
  Type (i) = 0x3ee31442,
  Length (i) = 6,
  Context ID (i) = 1,
  Next Context ID (i) = 0,
  Derived Field Type (i) = 0
  Derived Field Type (i) = 2
  Derived Field Type (i) = 4
  Derived Field Type (i) = 7
}
~~~
{: #ipv4-udp-ethernet-derived-assign title="DERIVED_ASSIGN Capsule for example IPv4/UDP ethernet frame"}

Table below illustrates fields present in Ethernet, IPv4 and UDP headers after derived fields were removed, their offsets in bits from the beginning of the frame and if they are likely to be static for most packets of a given traffic flow

| Offset | Field name | Length | Value | Static |
| --- | --- | --- | --- | --- |
| 0 | Destination MAC address | 48 | 00:00:5E:00:53:01 | Yes |
| 48 | Source MAC address | 48 | 00:00:5E:00:53:02 | Yes |
| 96 | EtherType | 16 | 0x0800 | Yes |
| 112 | Version | 4 | 0100b | Yes |
| 116 | Header length | 4 | 0101b | Yes |
| 120 | Traffic Class | 8 | 0x02 | Yes |
| 128 | Identification | 16 | 0x0000 | Yes |
| 144 | Flags | 3 | 010b | Yes |
| 147 | Fragment offset | 13 | 0000000000000b | Yes |
| 160 | TTL | 8 | 0x40 | Yes |
| 168 | Protocol | 8 | 0x11 | Yes |
| 176 | Source address | 32 | 192.0.2.1 | Yes |
| 208 | Destination address | 32 | 192.0.2.2 | Yes |
| 240 | Source port | 16 | 0xc199 | Yes |
| 256 | Destination port | 16 | 0x1151 | Yes |
| 272 | UDP payload | 9600 | ... | No |
{: #ipv4-udp-ethernet-fields title="Ethernet, IPv4 and UDP header fields in example frame"}

A single static segment model can be used for the initial part of the HTTP datagram after derived fields were removed:

| Segment Offset | Segment Length | Segment Contents | Segment Payload |
| --- | --- | --- | --- |
| 0 | 34 | Source MAC address, Destination MAC address, EtherType, Version, Header length and Traffic Class, Identification, Flags, Fragment offset, TTL, Protocol, Source address, Destination address, Source port and Destination port | 0x00005E00530100005E00530208004502000040004011c0000201c0000202c1991151 |
{: #ipv4-udp-segment title="Static segment for example Ethernet/IPv4/UDP frame"}

Resulting TEMPLATE_ASSIGN capsule with proxy-allocated odd Context ID is illustrated below:

~~~
TEMPLATE_ASSIGN Capsule {
  Type (i) = 0x3ee3143f,
  Length (i) = 38,
  Context ID (i) = 3,
  Next Context ID (i) = 1,
  Static Segment {
    Segment Offset (i) = 0,
    Segment Length (i) = 34,
    Segment Payload = 0x00005E00530100005E00530208004502000040004011c0000201c0000202c1991151,
  }
}
~~~
{: #ipv4-udp-template-assign title="TEMPLATE_ASSIGN Capsule for example IPv4/UDP ethernet frame"}

The resulting processing context chain removes 34 bytes of repeated header material from frames that match the chain, reducing the space they require in QUIC DATAGRAM frames.

The sender concatenates all variable regions in increasing offset order.

Upon receiving the datagram with Context ID 3, client re-assembles the datagram by appending variable segments to the static, re-calculates derived fields and inserts them at appropriate locations in the datagram.

# Security Considerations

This specification changes how HTTP Datagrams are reconstructed but does not weaken transport-layer integrity or confidentiality protections provided by the underlying HTTP mapping. All Capsules travel on the reliable control stream and inherit those protections.

## Resource Exhaustion

Processing contexts introduce receiver state and reconstruction work. An attacker could attempt to exhaust memory or CPU by creating excessive numbers of templates and static segments, purposely sending datagrams referencing not-yet-installed contexts and causing excessive buffering of unknown Context IDs.

Implementations MUST enforce limits on number of active templates and static segments and restrict memory used for buffering datagrams with unknown contexts.

## Amplification

Derived fields and template reconstruction increase the size of the reconstructed packet relative to the received datagram payload. An attacker could exploit this to amplify processing cost and perform a denial-of-service attack.

Endpoints MUST ensure that reconstructed packet size does not exceed the negotiated MTU and SHOULD apply rate limiting when expansion ratios are abnormally high.

# IANA Considerations

## HTTP Capsule Types Registration

This specification registers the following values in the "HTTP Capsule Types" registry:

Value | Capsule Type
--- | ---
0x3ee3143f | TEMPLATE_ASSIGN
0x3ee31440 | TEMPLATE_ACK
0x3ee31441 | TEMPLATE_CLOSE
0x3ee31442 | DERIVED_ASSIGN
0x3ee31443 | DERIVED_ACK
0x3ee31444 | DERIVED_CLOSE
0x3ee31445 | CHECKSUM_ASSIGN
0x3ee31446 | CHECKSUM_ACK
0x3ee31447 | CHECKSUM_CLOSE

All of these new entries use the following values for these fields:

Status:

: provisional (permanent if this document is approved)

Reference:

: This document

Change Controller:

: IETF

Contact:

: MASQUE Working Group masque@ietf.org

Notes:
: None

## HTTP Field Name Registration

This specification registers the following value in the "HTTP Field Name" registry:

- Field Name: http-datagram-contexts
- Status: provisional (permanent if approved)
- Structured Type: Dictionary
- Reference: This document
- Comments: None

## HTTP Datagram Derived Field Types Registry {#iana-derived-fields}

IANA is requested to create a new registry titled "HTTP Datagram Derived Field Types". The registration policy is expert review as specified in {{Section 4.5 of ?IANA-POLICY=RFC8126}}. This new registry governs the Derived Field types that appear in DERIVED_ASSIGN capsule and `derived` list of `http-datagram-contexts` dictionary.

Registrations are expected to reference a specification that defines how the field is derived. Such specifications are expected to describe the sender behavior and receiver behavior for the Derived Field Type, including how the relevant packet or frame header is located, the offset and length of the field, how the field value is computed, and any conditions under which the Derived Field Type is not applicable.

This new registry contains five columns:

Type:

: A positive integer identifying the field type

Name:

: A short name of the field

Description:

: A description of the field

Protocols:

: A list of HTTP Upgrade Tokens that the derived field type can apply

Reference:

: An optional reference defining the use of the entry.

The registry's initial entries are as follows:

Type | Name | Description | Protocols | Reference
--- | --- | --- | --- | ---
0 | ipv4-total-length | IPv4 Total Length field derived from reconstructed packet size | connect-ip, connect-ethernet | {{derived-ipv4-total-length}}
1 | ipv6-payload-length | IPv6 Payload Length field derived from reconstructed packet size | connect-ip, connect-ethernet | {{derived-ipv6-payload-length}}
2 | ipv4-udp-length | UDP Length derived from UDP header to end of IPv4 packet | connect-ip, connect-ethernet | {{derived-ipv4-udp-length}}
3 | ipv6-udp-length | UDP Length derived from UDP header to end of IPv6 packet | connect-ip, connect-ethernet | {{derived-ipv6-udp-length}}
4 | ipv4-header-checksum | IPv4 header checksum computed over IPv4 header | connect-ip, connect-ethernet | {{derived-ipv4-header-checksum}}
5 | ipv4-tcp-checksum | TCP checksum computed over IPv4 pseudo-header and segment | connect-ip, connect-ethernet | {{derived-ipv4-tcp-checksum}}
6 | ipv6-tcp-checksum | TCP checksum computed over IPv6 pseudo-header and segment | connect-ip, connect-ethernet | {{derived-ipv6-tcp-checksum}}
7 | ipv4-udp-checksum | UDP checksum computed over IPv4 pseudo-header and segment | connect-ip, connect-ethernet | {{derived-ipv4-udp-checksum}}
8 | ipv6-udp-checksum | UDP checksum computed over IPv6 pseudo-header and segment | connect-ip, connect-ethernet | {{derived-ipv6-udp-checksum}}
9 | ethernet-fcs | Ethernet Frame Check Sequence computed over the reconstructed Ethernet frame | connect-ethernet | {{derived-ethernet-fcs}}

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
