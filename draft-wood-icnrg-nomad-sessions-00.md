---
title: Nomad Sessions with CCNxKE
abbrev: Nomad Sessions with CCNxKE
docname: draft-wood-icnrg-nomad-sessions-00
category: std

<!-- ipr: pre5378Trust200902 -->
<!-- ipr: None -->
area: General
workgroup: icnrg
keyword: Internet-Draft

stand_alone: yes
pi:
  rfcedstyle: yes
  toc: yes
  tocindent: yes
  sortrefs: yes
  symrefs: yes
  strict: yes
  comments: yes
  inline: yes
  text-list-symbols: -o*+
author:
-
    ins: M. Mosko
    name: M. Mosko
    organization: PARC
    email: marc.mosko@parc.com
-
    ins: C. A. Wood
    name: Christopher A. Wood
    organization: PARC
    email: christopher.wood@parc.com

normative:
  GCM:
        title: "Recommendation for Block Cipher Modes of Operation: Galois/Counter Mode (GCM) and GMAC"
        date: 2007-11
        author:
            ins: M. Dworkin
        seriesinfo:
            NIST: Special Publication 800-38D

--- abstract

Traditional IP-based transport security protocols function between two peers. Sessions
are bound to both peer's addresses. Whenever either end host address changes,
such as when one changes network interfaces or moves between access points,
the session must be re-established to associate it with the new address tuple.
The base CCNxKE protocol allows a producer to migrate a session to trusted
endpoints during the key establishment phase. In this document, we describe
an extension to this protocol that enables this type of producer-side session
migration during normal usage of the session. Furthermore, in instances where
the session is used for bidirectional traffic, we describe an extension that
allows consumers to signal the producer when their routable prefix changes.
Together, this allows for nomad consumer-to-producer (or service) sessions
in CCN, i.e., sessions that are not pinned to prefixes or machines.

--- middle

#  Introduction

CCNxKE is a protocol that enables a consumer and producer to create a session
in a namespace, e.g., /nameA, over which they can communicate securely. An
example of this session is shown below.

~~~
+----------+          /nameA           +----------+
| Consumer +----(encrypted channel)----> Producer |
+----------+                           +----------+
~~~

The standard CCNx protocol allows the producer to migrate this session from
one namespace (prefix) to another. For example, if the producer is mobile and
its prefix changes from /nameA to /nameB, then the producer indicates this
migration through a MoveToken provided to the consumer.

~~~
+----------+          /nameB           +----------+
| Consumer +----(encrypted channel)----> Producer |
+----------+                           +----------+
~~~

In uni-directional sessions where the consumer only sends interests to
the producer, the consumer is free to change prefixes at will. However,
in bi-directional sessions, the consumer must notify the producer of its
prefix change retroactively to migrate the session information as needed.

The remainder of this document describes the signaling protocol used to
perform these prefix migrations and enable nomad sessions.

# Prefix Migration

A CCNxKE peer must only perform prefix migration if it is receiving interests
from another peer in a session. Prefix migration may be proactive or reactive.
In the former case, the migrating party provides a (MoveToken, MovePrefix, MoveTag)
tuple to the peer by sending an interest carrying this information. In the latter
case, this tuple is provided as a response to an interest.
In both cases, the tuple is carried in the CCNxKE signaling fields of the respective
interest or content object.

To generate the (MoveToken, MovePrefix, MoveTag), the migrating peer performs
the following computations.

~~~
   MoveTokenCT, MoveTokenTag =
        AEnc(K, H(H(client_migration_token)) + TS)
   MoveToken = K_id +  MoveTokenCT +  MoveTag
~~~

where K_id is the key identifier for the key K and + is concatenation. Here,
the client_migration_token value is the *source proof* and treated as
the migration token for the migrating peer. (It may therefore be
producer_migration_token if the producer is migrating). Also, AEnc is shorthand
for authenticated encryption that produces a ciphertext and authentication
tag. One such algorithm is AES-GCM {{GCM}}.

Once the peer receives a MoveToken it MUST use the newly provided MovePrefix for
future messages. To complete the migration, the stationary peer must echo the
source proof, e.g., H(client_migration_token), along with the (MoveToken, MoveTag) tuple
to the migrating peer. The migrating peer then confirms correct receipt of the token
by performing the following steps:

1. If K_id is not valid, i.e., the migrating peer has no key with that identifier,
then the session is not updated.
2. Otherwise, the migrating peer computes

~~~
    MoveTokenCT, MoveTokenTag = MoveToken
    token_hash + TS = ADec(K, MoveTokenCT, MoveTokenTag)
~~~

If the decryption fails, i.e., if the encryption is not valid (the ciphertext was
tampered with), then the Interest is dropped.
3. Otherwise, the migrating peer computes H(Proof) and compares it to
H(H(client_migration_token)) in the plaintext. If these values are equal,
then the migrating peer supplies a new Session ID to the stationary
peer and also sends a key-update message to cycle the TS.

## Examples

The following exchange shows a producer retroactively migrating from the prefix
/nameA to /nameB. The producer communicates the migration information in content
object responses sent to the consumer.

~~~
Consumer (stationary)              Producer (migrating)
   |   /nameA, (normal interest)       |
   +---------------------------------->|   (interest)
   |                                   |
   |  (MoveToken,/nameB,MoveTag)       |
   |<----------------------------------+   (content)
   |                                   |
   | /nameB, (MoveToken,MoveTag,Proof) |
   +---------------------------------->|   (interest)
   |                                   |
   |  (SessionID)                      |   
   |<----------------------------------+   (content)
~~~

The following exchange shows a consumer proactively migrating from the prefix
/prefixA to /prefixB. The consumer sends migration information in interests
to the producer.

~~~
Consumer (migrating)                      Producer (stationary)
   |  /prefixA, (normal interest)             |
   |<-----------------------------------------+
   |  (normal data response)                  |
   +----------------------------------------->|
   |                                          |
   |  /nameA, (MoveToken,/prefixB,MoveTag)    |
   +----------------------------------------->|   (interest)
   |                                          |
   |  (ACK data response)                     |
   |<-----------------------------------------+   (content)
   |                                          |
   | /prefixB, (MoveToken,MoveTag,Proof)      |
   <------------------------------------------+   (interest)
   |                                          |
   |  (SessionID)                             |   
   +------------------------------------------>   (content)
~~~

In the case where the producer has a prefix for the consumer (in a bidirectional
session), the producer may send the migration information in an interest according
to the flow above.

# Security Considerations

TODO
