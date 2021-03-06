=========================
ECIES-X25519-AEAD-Ratchet
=========================
.. meta::
    :author: zzz
    :created: 2018-11-22
    :thread: http://zzz.i2p/topics/2639
    :lastupdated: 2018-12-06
    :status: Open

.. contents::


Overview
========

This is a proposal for the first new end-to-end encryption type
since the beginning of I2P, to replace ElGamal/AES+SessionTags.

It relies on previous work as follows:

- Common structures spec
- I2NP spec
- ElGamal/AES+Session Tags spec http://i2p-projekt.i2p/en/docs/how/elgamal-aes
- http://zzz.i2p/topics/1768 new asymmetric crypto overview
- Low-level crypto overview https://geti2p.net/spec/cryptography
- ECIES http://zzz.i2p/topics/2418
- 111 NTCP2
- 123 New netDB Entries
- 142 New Crypto Template
- Signal double ratchet algorithm https://signal.org/docs/specifications/doubleratchet/

The goal is to support new encryption for end-to-end,
destination-to-destination commication.


ElGamal Key Locations
---------------------

As a review,
ElGamal 256-byte public keys may be found in the following data structures.
Reference the common structures specification.

- In a Router Identity
  This is the router's encryption key.

- In a Destination
  The public key of the destination was used for the old i2cp-to-i2cp encryption
  which was disabled in version 0.6, it is currently unused except for
  the IV for LeaseSet encryption, which is deprecated.
  The public key in the LeaseSet is used instead.

- In a LeaseSet
  This is the destination's encryption key.

- In a LS2
  This is the destination's encryption key.



EncTypes in Key Certs
---------------------

As a review,
we added support for encryption types when we added support for signature types.
The encryption type field is always zero, both in Destinations and RouterIdentities.
Whether to ever change that is TBD.
Reference the common structures specification.




Asymmetric Crypto Uses
----------------------

As a review, we use ElGamal for:

1) Tunnel Build messages (key is in RouterIdentity)
   Replacement is not covered in this proposal.
   No proposal yet.

2) Router-to-router encryption of netdb and other I2NP msgs (Key is in RouterIdentity)
   Depends on this proposal.
   Requires a proposal for 1) also, or putting the key in the RI options.

3) Client End-to-end ElGamal+AES/SessionTag (key is in LeaseSet, the Destination key is unused)
   Replacement IS covered in this proposal.

4) Ephemeral DH for NTCP1 and SSU
   Replacement is not covered in this proposal.
   See proposal 111 for NTCP2.
   No current proposal for SSU2.


Goals
-----

- Backwards compatible
- Requires and builds on LS2 (proposal 123)
- Leverage new crypto or primitives added for NTCP2 (proposal 111)
- No new crypto or primitives required for support
- Maintain decoupling of crypto and signing; support all current and future versions
- Enable new crypto for destinations
- Enable new crypto for routers, but only for garlic messages - tunnel building would
  be a separate proposal
- Don't break anything that relies on 32-byte binary destination hashes, e.g. bittorrent
- Maintain 0-RTT message delivery using ephemeral-static DH
- Upgrade to ephemeral-ephemeral DH after 1 RTT
- Maintain handling of out-of-order messages
- Maintain 256-bit security
- Add forward secrecy
- Add authentication (AEAD)
- Much more CPU-efficient than ElGamal
- Don't rely on Java jbigi to make DH efficient
- Minimize DH operations
- Much more bandwidth-efficient than ElGamal (514 byte ElGamal block)
- Eliminate several problems with session tags, including:
  - Inability to use AES until the first reply
  - Unreliability and stalls if tag delivery assumed
  - Bandwidth inefficient, especially on first delivery
  - Huge space inefficiency to store tags
  - Huge bandwidth overhead to deliver tags
  - Highly complex, difficult to implement
  - Difficult to tune for various use cases
  (streaming vs. datagrams, server vs. client, high vs. low bandwidth)
  - Memory exhaustion vulnerabilities due to tag delivery
- Support new and old crypto on same tunnel if desired
- Recipient is able to efficiently distinguish new from old crypto coming down
  same tunnel
- Others cannot distinguish new from old crypto
- Eliminate new vs. existing session length classification (support padding)
- No new I2NP messages required
- Replace SHA-256 checksum in AES payload with AEAD
- (Optimistic) Add extensions or hooks to support multicast
- Support binding of transmit and receive sessions so that
  acknowledgements may happen within the protocol, rather than solely out-of-band.
  This will also allow replies to have forward secrecy immediately.


Non-Goals / Out-of-scope
------------------------

- LS2 format (see proposal 123)
- New DHT rotation algorithm or shared random generation
- New encryption for tunnel building.
  That would be in a separate proposal.
- Methods of encryption, transmission, and reception of I2NP DLM / DSM / DSRM messages.
  Not changing.
- No LS1-to-LS2 or ElGamal/AES-to-this-proposal communication is supported.
  This proposal is a bidirectional protocol.
  Destinations may handle backward compatibility by publishing two leasesets
  using the same tunnels, or put both encryption types in the LS2.
- Threat model changes
- Implementation details are not discussed here and are left to each project.



Justification
-------------

ElGamal/AES+SessionTag has been our sole end-to-end protocol for around for about 15 years,
essentially without modifications to the protocol.
There are now cryptographic primitives that are faster.
We need to enhance the security of the protocol.
We have also developed heuristic strategies and workarounds to minimize the
memory and bandwidth overhead of the protocol, but those strategies
are fragile, difficult to tune, and render the protocol even more prone
to break, causing the session to drop.

For about the same time period, the ElGamal/AES+SessionTag specification and related
documentation have described how bandwidth-expensive it is to deliver session tags,
and have proposed replacing session tag delivery with a "synchronized PRNG".
A synchronized PRNG deterministically generates the same tags at both ends,
derived from a common seed.
A synchronized PRNG can also be termed a "ratchet".
This proposal (finally) specifies that ratchet mechanism, and eliminates tag delivery.



New Cryptographic Primitives for I2P
====================================

Existing I2P router implementations will require implementations for
the following standard cryptographic primitives,
which are not required for current I2P protocols:

1) ECIES (but this is essentially X25519)

Existing I2P router implementations that have not yet implemented NTCP2 (Proposal 111)
will also require implementations for:

1) X25519 key generation and DH

2) AEAD_ChaCha20_Poly1305 (abbreviated as ChaChaPoly below)

3) HKDF


Detailed Proposal
=================

This proposal defines a new end-to-end protocol to replace ElGamal/AES+SessionTags.

There are five portions of the protocol to be redesigned:


- The new and existing session container format
  is replaced with a new format.
- ElGamal (256 byte public keys, 128 byte private keys) is be replaced
  with ECIES-X25519 (32 byte public and private keys)
- AES is be replaced with
  AEAD_ChaCha20_Poly1305 (abbreviated as ChaChaPoly below)
- SessionTags will be replaced with ratchets,
  which is essentially a cryptographic, synchronized PRNG.
- The AES payload, as defined in the ElGamal/AES+SessionTags specification,
  is replaced with a block format similar to that in NTCP2.


Crypto Type
-----------

The crypto type (used in the LS2) is 4.
This indicates a 32-byte X25519 public key,
and the end-to-end protocol specified here.

Crypto type 0 is ElGamal.
Crypto types 1-3 are reserved for an upcoming ECIES-ECDH-AES-SessionTag proposal.



Sessions
--------

The current ElGamal/AES+SessionTag protocol is unidirectional.
At this layer, the receiver doesn't know where a message is from.
Outbound and inbound sessions are not associated.
Acknowledgements are out-of-band using a DeliveryStatusMessage
(wrapped in a GarlicMessage) in the clove.

There is substantial inefficiency in a unidirectional protocol.
Any reply must also use an expensive 'new session' message.
This causes higher bandwidth, CPU, and memory usage.

There are also security weaknesses in a unidirectional protocol.
All sessions are based on ephemeral-static DH.
Without a return path, there is no way for Bob to "ratchet" his static key
to an ephemeral key.
Without knowing where a message is from, there's no way to use
the received ephemeral key for outbound messages,
so the initial reply also uses ephemeral-static DH.

For this proposal, we define two mechanisms to create a bidirectional protocol -
"pairing" and "binding".
These mechanisms provide increased efficiency and security.


Session Context
```````````````

As with ElGamal/AES+SessionTags, all inbound and outbound sessions
must be in a given context, either the router's context or
the context for a particular local destination.
In Java I2P, this context is called the Session Key Manager.

Sessions must not be shared among contexts, as that would
allow correlation among the various local destinations,
or between a local destination and a router.

When a given destination supports both ElGamal/AES+SessionTags
and this proposal, both types of sessions may share a context.
See section 1c) below.



Pairing Inbound and Outbound Sessions
`````````````````````````````````````

When an outbound session is created at the originator (Alice),
a new inbound session is created and paired with the outbound session,
unless no reply is expected (e.g. raw datagrams).

A new inbound session is always paired with a new outbound session,
unless no reply is requested (e.g. raw datagrams).

If a reply is requested and bound to a far-end destination or router,
that new outbound session is bound to that destination or router,
and replaces any previous outbound session to that destination or router.

Pairing inbound and outbound sessions provides a bidirectional protocol
with the capability of ratcheting the DH keys.



Binding Sessions and Destinations
`````````````````````````````````

There is only one outbound session to a given destination or router.
There may be several current inbound sessions from a given destination or router.
Generally, when a new inbound session is created, and traffic is received
on that session (which serves as an ACK), any others will be marked
to expire relatively quickly, within a minute or so.
The previous messages sent (PN) value is checked, and if there are no
unreceived messages (within the window size) in the previous inbound session,
the previous session may be deleted immediately.


When an outbound session is created at the originator (Alice),
it is bound to the far-end Destination (Bob),
and any paired inbound session will also be bound to the far-end Destination.
As the sessions ratchet, they continue to be bound to the far-end Destination.

When an inbound session is created at the receiver (Bob),
it may be bound to the far-end Destination (Alice), at Alice's option.
If Alice includes binding information (her Destination hash and signature) in the new session message,
the session will be bound to that destination,
and a outbound session will be created and bound to same Destination.
As the sessions ratchet, they continue to be bound to the far-end Destination.


Benefits of Binding and Pairing
```````````````````````````````

For the common, streaming case, we expect Alice and Bob to use the protocol as follows:

- Alice pairs her new outbound session to a new inbound session, both bound to the far-end destination (Bob).
- Alice includes the binding information and signature, and a reply request, in the
  new session message sent to Bob.
- Bob pairs his new inbound session to a new outbound session, both bound to the far-end destination (Alice).
- Bob sends a reply (ack) to Alice in the paired session, with a ratchet to a new DH key.
- Alice ratchets to a new outbound session with Bob's new key, paired to the existing inbound session.

By binding an inbound session to a far-end Destination, and pairing the inbound session
to an outbound session bound to the same Destination, we achieve two major benefits:

1) The initial reply from Bob to Alice uses ephemeral-ephemeral DH

2) After Alice receives Bob's reply and ratchets, all subsequent messages from Alice to Bob
use ephemeral-ephemeral DH.


Message ACKs
````````````

In ElGamal/AES+SessionTags, when a LeaseSet is bundled as a garlic clove,
or tags are delivered, the sending router requests an ACK.
This is a separate garlic clove containing a DeliveryStatus Message.
For additional security, the DeliveryStatus Message is wrapped in a Garlic Message.
This mechanism is out-of-band from the perspective of the protocol.

In the new protocol, since the inbound and outbound sessions are paired,
we can have ACKs in-band. No separate clove is required.

An explicit ACK is simply an existing session message with no I2NP block.
However, in most cases, an explict ACK can be avoided, as there is reverse
traffic. Implementations should set a short timer (a few hundred ms)
before sending an explicit ACK.

Implementations will also need to defer any ACK sending until after the
I2NP block is processed, as the Garlic Message may contain a Database Store Message
with a lease set. A recent lease set will be necessary to route the ACK,
and the far-end destination (contained in the lease set) will be necessary to
verify the binding signature.


Session Timeouts
````````````````

Outbound sessions should always expire before inbound sessions.
One an outbound session expires, and a new one is created, a new paired inbound
session will be created as well. If there was an old inbound session,
it will be allowed to expire.


Bandwidth overhead estimate
----------------------------

Message overhead for the first two messages in each direction are as follows.
This assumes only one message in each direction before the ACK,
or that any additional messages are sent speculatively as existing session messages.
If there is no speculative acks of delivered session tags, the
overhead or the old protocol is much higher.

No padding assumed for the new protocol.


For ElGamal/AES+SessionTags
```````````````````````````

New session message, Same each direction.

ElGamal block:
514 bytes

AES block:
- 2 byte tag count
- 1024 bytes of tags (32 typical)
- 4 byte payload size
- 32 byte hash of payload
- 1 byte flags
- 8 byte (average) padding to 16 bytes
1071 total

Total:
1585 bytes

Existing session messages, same each direction

AES block:
- 32 byte session tag
- 2 byte tag count
- 4 byte payload size
- 32 byte hash of payload
- 1 byte flags
- 8 byte (average) padding to 16 bytes
79 total

Four message total (two each direction)
3328 bytes


For ECIES-X25519-AEAD-Ratchet
`````````````````````````````

TODO update this section after proposal is stable.


Alice-Bob new session message:
- 32 byte public key
- 8 byte nonce
- 6 byte message ID block
- 7 byte options block
- 37 byte next key ratchet block
- 103 byte ack request block
- 3 byte I2NP block overhead ?
- 16 byte Poly1305 tag

Total:
212 bytes

Bob-Alice existing session message:
- 8 byte session tag
- 6 byte message ID block
- 7 byte options block
- 37 byte next key ratchet block
- 4 byte ack request block
- 3 byte I2NP block overhead ?
- 16 byte Poly1305 tag

Total:
81 bytes

Existing session messages, same each direction
- 8 byte session tag
- 6 byte message ID block
- 3 byte I2NP block overhead ?
- 16 byte Poly1305 tag

Total:
33 bytes

Four message total (two each direction)
359 bytes
89% (approx. 10x) reduction compared to ElGamal/AEs+SessionTags


Processing overhead estimate
----------------------------

TODO update this section after proposal is stable.

The following cryptographic operations are required by each party to initiate
a new session and do the first ratchet:

- HMAC-SHA256: 3 per HKDF, total TBD
- ChaChaPoly: 2 each
- X25519 key generation: 2 Alice, 1 Bob
- X25519 DH: 3 each
- Signature verification: 1 (Bob)


The following cryptographic operations are required by each party for each data phase message:

- ChaChaPoly: 1



Session Tag Length Analysis
---------------------------

Current session tag length is 32 bytes.
We have not yet found any justification for that length, but we are continuing to research the archives.

The session tag ratchet is assumed to generate random, uniformly distributed tags.
There is no cryptographic reason for a particular session tag length.
The session tag ratchet is synchronized to, but generates an independent output from,
the symmetric key ratchet. The outputs of the two ratchets may be different lengths.

Therefore, the only concern is session tag collision.
It is assumed that implementations will not attempt to handle collisions
by trying to decrypt with both sessions;
implementations will simply associate the tag with either the previous or new
session, and any message received with that tag on the other session
will be dropped after the decryption fails.

The goal is to select a session tag length that is large enough
to minimize the risk of collisions, while small enough
to minimize memory usage.

This assumes that implementations limit session tag storage to
prevent memory exhaustion attacks. This also will greatly reduce the chances that an attacker
can create collisions. See the Implementation Considerations section below.

For a worst case, assume a busy server with 64 new inbound sessions per second.
Assume 15 minute inbound session tag lifetime (same as now, probably should be reduced).
Assume inbound session tag window of 32.
64 * 15 * 60 * 32 =  1,843,200 tags
Current Java I2P max inbound tags is 750,000 and has never been hit as far as we know.

A target of 1 in a million (1e-6) session tag collisions is probably sufficient.
The probability of dropping a message along the way due to congestion is far higher than that.

Ref: https://en.wikipedia.org/wiki/Birthday_paradox
Probability table section.

With 32 byte session tags (256 bits) the session tag space is 1.2e77.
The probability of a collision with probability 1e-18 requires 4.8e29 entries.
The probability of a collision with probability 1e-6 requires 4.8e35 entries.
1.8 million tags of 32 bytes each is about 59 MB total.

With 16 byte session tags (128 bits) the session tag space is 3.4e38.
The probability of a collision with probability 1e-18 requires 2.6e10 entries.
The probability of a collision with probability 1e-6 requires 2.6e16 entries.
1.8 million tags of 16 bytes each is about 30 MB total.

With 8 byte session tags (64 bits) the session tag space is 1.8e19.
The probability of a collision with probability 1e-18 requires 6.1 entries.
The probability of a collision with probability 1e-6 requires 6.1e6 (6,100,000) entries.
1.8 million tags of 8 bytes each is about 15 MB total.

6.1 million active tags is over 3x more than our worst-case estimate of 1.8 million tags.
So the probability of collision would be less than one in a million.
We therefore conclude that 8 byte session tags are sufficient.
This results in a 4x reduction of storage space,
in addition to the 2x reduction because transmit tags are not stored.
So we will have a 8x reduction in session tag memory usage compared to ElGamal/AES+SessionTags.

To maintain flexibility should these assumptions be wrong,
we will include a session tag length field in the options,
so that the default length may be overridden on a per-session basis.

Implementations should, at a minimum, recognize session tag collisions,
handle them gracefully, and log or count the number of collisions.
While still extremely unlikely, they will be much more likely than
they were for ElGamal/AES+SessionTags, and could actually happen.


Multicast
---------

TBD



1) Message format
-----------------

Review of ElGamal/AES+SessionTags Message Format
````````````````````````````````````````````````

In ElGamal/AES+SessionTags, there are two message formats:

1) New session:
- 514 byte ElGamal block
- AES block (128 bytes minimum, multiple of 16)

2) Existing session:
- 32 byte Session Tag
- AES block (128 bytes minimum, multiple of 16)

The minimum padding to 128 is as implemented in Java I2P but is not enforced on reception.

These messages are encapsulated in a I2NP garlic message, which contains
a length field, so the length is known.

Note that there is no padding defined to a non-mod-16 length,
so the new session is always (mod 16 == 2),
and an existing session is always (mod 16 == 0).
We need to fix this.

The receiver first attempts to look up the first 32 bytes as a Session Tag.
If found, he decrypts the AES block.
If not found, and the data is at least (514+16) long, he attempts to decrypt the ElGamal block,
and if successful, decrypts the AES block.


New Session Tags and comparison to Signal
`````````````````````````````````````````

In Signal Double Ratchet, the header contains:

- DH: Current ratchet public key
- PN: Previous chain message length
- N: Message Number

By using a session tag, we can eliminate most of that.

In new session, we put only the public key in the unencrytped header.

In existing session, we use a session tag for the header.
The session tag is associated with the current ratchet public key,
and the message number.

In both new and existing session, PN and N are in the encrypted body.

In Signal, things are constantly ratcheting. A new DH public key requires the
receiver to ratchet and send a new public key back, which also serves
as the ack for the received public key.
This would be far too many DH operations for us.
So we separate the ack of the received key and the transmission of a new public key.
Any message using a session tag generated from the new DH public key constitutes an ACK.
We only transmit a new public key when we wish to rekey.

The maximum number of messages before the DH must ratchet is 65535.

When delivering a session key, we derive the "Tag Set" from it,
rather than having to deliver session tags as well.
A Tag Set can be up to 65536 tags.
However, receivers should implement a "look-ahead" strategy, rather
than generating all possible tags at once.
Only generate at most N tags past the last good tag received.
N might be at most 128, but 32 or even less may be a better choice.


Typical Usage Patterns
----------------------


HTTP GET
````````

Alice sends a small request with a single new Session message, bundling a reply leaseset.
Alice includes immediate ratchet to new key.
Includes sig to bind to destination. No ack requested.

Bob ratchets immediately.

Alice ratchets immediately.

Continues on with those sessions.


HTTP POST
`````````

Alice has three options:

1) Send the first message only (window size = 1), as in HTTP GET.

2) Send up to streaming window, but using same cleartext public key.
   All messages contain same next public key (ratchet).
   This will be visible to OBGW/IBEP because they all start with the same cleartext.
   Things proceed as in 1).

3) Send up to streaming window, but using a different cleartext public key (session) for each.
   All messages contain same next public key (ratchet).
   This will not be visible to OBGW/IBEP because they all start with different cleartext.
   Bob must recognize that they all contain the same next public key,
   and respond to all with the same ratchet.
   Alice uses that next public key and continues.


Repliable Datagram
``````````````````
As in HTTP GET, but with smaller options for session tag window size and lifetime.
Contains signature to bind the session to the destination.
Maybe don't request a ratchet.


Raw Datagram
````````````
New session message is sent.
No reply LS is bundled. No signature included. No reply or ratchet is requested.
No ratchet is sent.
Options set session tags window to zero.


Long-Lived Sessions
```````````````````
Long-lived sessions may ratchet, or request a ratchet, at any time,
to maintain forward secrecy from that point in time.
Sessions must ratchet as they approach the limit of sent messages per-session (65535).

Implementation Considerations
`````````````````````````````

As with the existing ElGamal/AES+SessionTag protocol, implementations must
limit session tag storage and protect against memory exhaustion attacks.

Some recommended strategies include:

- Hard limit on number of session tags stored
- Aggressive expiration of idle inbound sessions when under memory pressure
- Limit on number of inbound sessions bound to a single far-end destination
- Adaptive reduction of session tag window and deletion of old unused tags
  when under memory pressure
- Refusal to ratchet when requested, if under memory pressure





1a) New session format
----------------------

Public key (32 bytes)
Nonce (8 bytes)
Encrypted data and MAC (see section 3 below)


Format
``````
Encrypted:

.. raw:: html

  {% highlight lang='dataspec' %}
+----+----+----+----+----+----+----+----+
  |                                       |
  +                                       +
  |       New Session Public Key          |
  +                                       +
  |                                       |
  +                                       +
  |                                       |
  +----+----+----+----+----+----+----+----+
  |                                       |
  +                                       +
  |       ChaCha20 encrypted data         |
  +                                       +
  |                                       |
  +                                       +
  |                                       |
  +                                       +
  |                                       |
  +----+----+----+----+----+----+----+----+
  |  Poly1305 Message Authentication Code |
  +              (MAC)                    +
  |             16 bytes                  |
  +----+----+----+----+----+----+----+----+
  |                                       |
  +                                       +
  |       ChaCha20 encrypted data         |
  ~                                       ~
  |                                       |
  +                                       +
  |                                       |
  +----+----+----+----+----+----+----+----+
  |  Poly1305 Message Authentication Code |
  +              (MAC)                    +
  |             16 bytes                  |
  +----+----+----+----+----+----+----+----+

  Public Key :: 32 bytes, little endian, cleartext

  encrypted data 1 :: 40 bytes

  MAC 1 :: Poly1305 message authentication code, 16 bytes

  encrypted data 2 :: Same size as plaintext data, Size varies

  MAC 2 :: Poly1305 message authentication code, 16 bytes

{% endhighlight %}


Decrypted data 1:

.. raw:: html

  {% highlight lang='dataspec' %}
+----+----+----+----+----+----+----+----+
  |                                       |
  +                                       +
  |       DH Ratchet Public Key           |
  +                                       +
  |                                       |
  +                                       +
  |                                       |
  +----+----+----+----+----+----+----+----+
  |   Flags, IV, sequence number, TODO    |
  +----+----+----+----+----+----+----+----+

  Public Key :: 32 bytes, little endian

  TODO :: 8 bytes

{% endhighlight %}




Decrypted data 2:

  See AEAD section below.





KDF
```


.. raw:: html

  {% highlight lang='text' %}

See message key ratchet below.

Key: KDF TBD
IV: As published in a LS2 property?
Nonce: From header

{% endhighlight %}


Justification
`````````````

By using a ratchet (a synchronized PRNG) to generate the
session tags, we eliminate the overhead of sending session tags
in the new session message and subsequent messages when needed.
For a typical tag set of 32 tags, this is 1KB.
This also eliminates the storage of session tags on the sending side,
thus cutting the storage requirements in half.


Notes
`````

This allows sending multiple new session messages with the same initial ratchet key,
which is more efficient, e.g. for a POST.
These messages will have a different cleartext (new session) key but contain
the same ratchet key inside the first AEAD block.
New session keys are never reused.
This prevents external observers from identifying a POST sequence through
seeing duplicate cleartext keys. However, these messages may still be
identified as containing keys, unless we use Elligator2.
The first AEAD block will contain a sequence number and/or IV so the second block may
be decrypted correctly.


Issues
``````

- Obfuscation of cleartext key? We could do Elligator 2 but that's expensive.

- Do we need a nonce? Does it need to be 8 bytes? 4?

- IV is in the LS2 property? Alternative: Send a 16 byte IV instead of 8 byte nonce,
  and use a nonce of 0.



1b) Existing session format
---------------------------

Session tag (8 bytes)
Encrypted data and MAC (see section 3 below)


Format
``````
Encrypted:

.. raw:: html

  {% highlight lang='dataspec' %}
+----+----+----+----+----+----+----+----+
  |       Session Tag                     |
  +----+----+----+----+----+----+----+----+
  |                                       |
  +                                       +
  |       ChaCha20 encrypted data         |
  ~                                       ~
  |                                       |
  +                                       +
  |                                       |
  +----+----+----+----+----+----+----+----+
  |  Poly1305 Message Authentication Code |
  +              (MAC)                    +
  |             16 bytes                  |
  +----+----+----+----+----+----+----+----+

  Session Tag :: 8 bytes, cleartext

  encrypted data :: Same size as plaintext data, size varies

  MAC :: Poly1305 message authentication code, 16 bytes

{% endhighlight %}


Decrypted:
  See AEAD section below.


KDF
```

.. raw:: html

  {% highlight lang='text' %}

See message key ratchet below.

Key: KDF TBD
IV: KDF TBD
Nonce: The message number N in the current chain, as retrieved from the associated Session Tag.

{% endhighlight %}


Justification
`````````````

Notes
`````


Issues
``````



1c) Identification at Receiver
------------------------------

Following are recommendations for classifying incoming messages.


X25519 Only
```````````

On a tunnel that is solely used with this protocol, do identification
as is done currently with ElGamal/AES+SessionTags:

First, treat the initial data as a session tag, and look up the session tag.
If found, decrypt using the stored data associated with that session tag.

If not found, treat the initial data as a DH public key and nonce.
Perform a DH operation and the specified KDF, and attempt to decrypt the remaining data.


X25519 Shared with ElGamal/AES+SessionTags
``````````````````````````````````````````

On a tunnel that supports both this protocol and
ElGamal/AES+SessionTags, classify incoming messages as follows:

Due to a flaw in the ElGamal/AES+SessionTags specification,
the AES block is not padded to a random non-mod-16 length.
Therefore, the length of existing session messages mod 16 is always 0,
and the length of new session messages mod 16 is always 2 (since the
ElGamal block is 514 bytes long).

If the length mod 16 is not 0 or 2,
treat the initial data as a session tag, and look up the session tag.
If found, decrypt using the stored data associated with that session tag.

If not found, and the length mod 16 is not 0 or 2,
treat the initial data as a DH public key and nonce.
Perform a DH operation and the specified KDF, and attempt to decrypt the remaining data.
(based on the relative traffic mix, and the relative costs of X25519 and ElGamal DH operations,
ths step may be done last instead)

Otherwise, if the length mod 16 is 0,
treat the initial data as a ElGamal/AES session tag, and look up the session tag.
If found, decrypt using the stored data associated with that session tag.

If not found, and the data is at least 642 (514 + 128) bytes long,
and the length mod 16 is 2,
treat the initial data as a ElGamal block.
Attempt to decrypt the remaining data.

Note that if the ElGamal/AES+SessionTag spec is updated to allow
non-mod-16 padding, things will need to be done differently.



2) ECIES-X25519
---------------


Format
``````

32-byte public and private keys.


Justification
`````````````

Used in NTCP2.



Notes
`````


Issues
``````











3) AEAD (ChaChaPoly)
--------------------

AEAD using ChaCha20 and Poly1305, same as in NTCP2.


Format
``````

Inputs to the encryption/decryption functions:

.. raw:: html

  {% highlight lang='dataspec' %}
k :: 32 byte cipher key, as generated from KDF

  nonce :: Counter-based nonce, 12 bytes.
           Starts at 0 and incremented for each message.
           First four bytes are always zero.
           In new session message:
           Last eight bytes are the nonce from the message header.
           In existing session message:
           Last eight bytes are the message number (N), little-endian encoded.
           Maximum value is 2**64 - 2.
           Session must be ratcheted before N reaches that value.
           The value 2**64 - 1 must never be used.

  ad :: In new session message:
        Associated data, 32 bytes.
        The SHA256 hash of the preceding data (public key and nonce)
        In existing session message:
        Zero bytes

  data :: Plaintext data, 0 or more bytes

{% endhighlight %}


Output of the encryption function, input to the decryption function:

.. raw:: html

  {% highlight lang='dataspec' %}
+----+----+----+----+----+----+----+----+
  |                                       |
  +                                       +
  |       ChaCha20 encrypted data         |
  ~               .   .   .               ~
  |                                       |
  +----+----+----+----+----+----+----+----+
  |  Poly1305 Message Authentication Code |
  +              (MAC)                    +
  |             16 bytes                  |
  +----+----+----+----+----+----+----+----+

  Obfs Len :: Length of (encrypted data + MAC) to follow, 16 - 65535
              Obfuscation using SipHash (see below)
              Not used in message 1 or 2, or message 3 part 1, where the length is fixed
              Not used in message 3 part 1, as the length is specified in message 1

  encrypted data :: Same size as plaintext data, 0 - 65519 bytes

  MAC :: Poly1305 message authentication code, 16 bytes

{% endhighlight %}

For ChaCha20, what is described here corresponds to [RFC-7539]_, which is also
used similarly in TLS [RFC-7905]_.

Notes
`````
- Since ChaCha20 is a stream cipher, plaintexts need not be padded.
  Additional keystream bytes are discarded.

- The key for the cipher (256 bits) is agreed upon by means of the SHA256 KDF.
  The details of the KDF for each message are in separate sections below.

- ChaChaPoly frames are of known size as they are encapsulated in the I2NP data message.

- For all messages,
  padding is inside the authenticated
  data frame.


AEAD Error Handling
```````````````````

TBD


Justification
`````````````

Used in NTCP2.


Notes
`````


Issues
``````

We may need a different AEAD with a larger nonce that's resistant to nonce reuse,
so we can use random nonces. (SIV?)





4) Ratchets
-----------

We still use session tags, as before, but we use ratchets to generate them.
Session tags also had a rekey option that we never implemented.
So it's like a double ratchet but we never did the second one.

Here we define something like Signal Double Ratchet.
The session tags are generated deterministically and identically on
the receiver and sender sides.

By using a symmetric key/tag ratchet, we eliminate memory usage to store session tags on the sender side.
We also eliminate the bandwidth consumption of sending tag sets.
Receiver side usage is still significant, but we can reduce it further
as we will shrink the session tag from 32 bytes to 8 bytes.

We do not use header encryption as is specified (and optional) in Signal,
we use session tags instead.

By using a DH ratchet, we acheive forward secrecy, which was never implemented
in ElGamal/AES+SessionTags.

Note: The new session public key is not part of the ratchet, its sole function
is to encrypt Alice's initial DH ratchet key.


Message Numbers
```````````````

The Double Ratchet handles lost or out-of-order messages by including in each message header
the message's number in the sending chain (N=0,1,2,...)
and the length (number of message keys) in the previous sending chain (PN).
This enables the recipient to advance to the relevant message key while storing skipped message keys
in case the skipped messages arrive later.

On receiving a message, if a DH ratchet step is triggered then the received PN
minus the length of the current receiving chain is the number of skipped messages in that receiving chain.
The received N is the number of skipped messages in the new receiving chain (i.e. the chain after the DH ratchet).

If a DH ratchet step isn't triggered, then the received N minus the length of the receiving chain
is the number of skipped messages in that chain.


Session Tag Ratchet
```````````````````

Ratchets for every message, as in Signal.
The session tag ratchet is synchronized with the symmetric key ratchet,
but the receiver key ratchet may "lag behind" to save memory.

Transmitter ratchets once for each message transmitted.
No additional tags must be stored.
The transmitter must also keep a counter for 'N', the message number
of the message in the current chain. The 'N' value is included
in the sent message.
See the Message Number block definition.

Receiver must ratchet ahead by the max window size and store the tags in a "tag set",
which is associated with the session.
Once received, the stored tag may be discarded, and if there are no previous
unreceived tags, the window may be advanced.
The receiver should keep the 'N' value associated with each session tag,
and check that the number in the sent message matches this value.
See the Message Number block definition.


KDF:

.. raw:: html

  {% highlight lang='text' %}

  Inputs:
  1) Chain key
     First time: output from DH ratchet
     Subsequent times: output from previous session tag ratchet
  2) input_key_material = constant (from where? SHA-256(some constant)?)
     Must be unique for this chain (generated from chain key),
     so that the sequence isn't predictable

  TBD


  Outputs:
  1) N (the current session tag number)
  2) the session tag (and symmetric key, probably)
  3) the next Chain Key (KDF input for the next session tag ratchet)


{% endhighlight %}


Symmetric Key Ratchet
`````````````````````

Ratchets for every message, as in Signal.
Each symmetric key has an associated message number and session tag.
The session key ratchet is synchronized with the symmetric tag ratchet,
but the receiver key ratchet may "lag behind" to save memory.

Transmitter ratchets once for each message transmitted.
No additional keys must be stored.

When receiver gets a session tag, if it has not already ratcheted the
symmetric key ratchet ahead to the associated key, it must "catch up" to the associated key.
The receiver will probably cache the keys for any previous tags
that have not yet been received.
Once received, the stored key may be discarded, and if there are no previous
unreceived tags, the window may be advanced.

For efficiency, the session tag and symmetric key ratchets should be separate so
the session tag ratchet can run ahead of the symmetric key ratchet.
This also provides some additional security, since the session tags go out on the wire.


KDF:

.. raw:: html

  {% highlight lang='text' %}

  Probably output from the session tag ratchet, not run separately?


{% endhighlight %}


DH Ratchet
``````````

Ratchets but not nearly as fast as Signal does.
We separate the ack of the received key from generating the new key.
In typical usage, Alice and Bob will each ratchet (twice) immediately in a new session,
but will not ratchet again.

Note that a ratchet is for a single direction, and generates a new session tag / message key ratchet chain for that direction.
To generate keys for both directions, you have to ratchet twice.

You ratchet every time you generate and send a new key.
You ratchet every time you receive a new key.

Alice ratchets once when she initiates a new outbound session and creates the corresponding inbound session.
Bob ratchets twice when he receives the inbound session and creates the corresponding outbound session,
once for the new key received, and once for the new key generated.
Alice ratchets once when she receives the new key on the inbound session and replaces the corresponding outbound session.
So each side ratchets twice total, in the typical case.

The frequency of ratchets after the initial handshake is implementation-dependent.
While the protocol places a limit of 65535 messages before a ratchet is required,
more frequent ratcheting (based on message count, elapsed time, or both)
may provide additional security.


KDF:

.. raw:: html

  {% highlight lang='text' %}

  TODO there's two AEAD blocks in the new session message, explain KDF for both

  Inputs:
  1) Root key (first time from where? see Signal section 3.3)
  2) input_key_material

  First time:
  Alice generates her ephemeral DH key pair e.

  // DH(e, rs) == DH(s, re)
  Define input_key_material = 32 byte DH result of Alice's ephemeral key and Bob's static key
  Set input_key_material = X25519 DH result

  After that:
  Alice or Bob generates her or his ephemeral DH key pair e.

  // DH(e, re) == DH(e, re)
  Define input_key_material = 32 byte DH result of Alice's ephemeral key and Bob's ephemeral key
  Set input_key_material = X25519 DH result


  // MixKey(DH())

  TBD


  Outputs:
  1) the chain key to initialize the session tag and symmetric key ratchets
  2) the next Root Key (KDF input for the next ratchet)


{% endhighlight %}




5) Payload
----------

This replaces the AES section format defined in the ElGamal/AES+SessionTags specification.

This uses the same block format as defined in the NTCP2 specification.
Where appropriate, the same block numbers are used.
We do not reuse NTCP2 block numbers for different uses, so
that implementations may share code with NTCP2.

TODO there are concerns that encouraging implementers to share code
may lead to parsing issues. Implementers should carefully consider
the benefits and risks of sharing code, and ensure that the
ordering and valid block rules are different for the two contexts.



Unencrypted data
````````````````
There are zero or more blocks in the encrypted frame.
Each block contains a one-byte identifier, a two-byte length,
and zero or more bytes of data.

For extensibility, receivers must ignore blocks with unknown identifiers,
and treat them as padding.

Encrypted data is 65535 bytes max, including a 16-byte authentication header,
so the max unencrypted data is 65519 bytes.

(Poly1305 auth tag not shown):

.. raw:: html

  {% highlight lang='dataspec' %}
+----+----+----+----+----+----+----+----+
  |blk |  size   |       data             |
  +----+----+----+                        +
  |                                       |
  ~               .   .   .               ~
  |                                       |
  +----+----+----+----+----+----+----+----+
  |blk |  size   |       data             |
  +----+----+----+                        +
  |                                       |
  ~               .   .   .               ~
  |                                       |
  +----+----+----+----+----+----+----+----+
  ~               .   .   .               ~

  blk :: 1 byte
         0-2 reserved (used in NTCP2)
         3 for I2NP message (Garlic Message only)
         4 termination
         5 options
         6 message number and previous message number (ratchet)
         7 next session key
         8 ack of reverse session key
         9 reply delivery instructions
         224-253 reserved for experimental features
         254 for padding
         255 reserved for future extension
  size :: 2 bytes, big endian, size of data to follow, 0 - 65516
  data :: the data

  Maximum ChaChaPoly frame is 65535 bytes.
  Poly1305 tag is 16 bytes
  Maximum total block size is 65519 bytes
  Maximum single block size is 65519 bytes
  Block type is 1 byte
  Block length is 2 bytes
  Maximum single block data size is 65516 bytes.

{% endhighlight %}


Block Ordering Rules
````````````````````
In the new session message, order must be:
TBD
xx, followed by Options if present, followed by Padding if present.
No other blocks are allowed.

In the existing session message, order is unspecified, except for the
following requirements:
TBD
Padding, if present, must be the last block.
Termination, if present, must be the last block except for Padding.

There may be multiple I2NP blocks in a single frame.
Multiple Padding blocks are not allowed in a single frame.
Other block types probably won't have multiple blocks in
a single frame, but it is not prohibited.



I2NP Message
````````````

An single I2NP message with a modified header.
I2NP messages may not be fragmented across blocks or
across ChaChaPoly frames.

This uses the first 9 bytes from the standard NTCP I2NP header,
and removes the last 7 bytes of the header, as follows:
truncate the expiration from 8 to 4 bytes,
remove the 2 byte length (use the block size - 9),
and remove the one-byte SHA256 checksum.


.. raw:: html

  {% highlight lang='dataspec' %}
+----+----+----+----+----+----+----+----+
  | 3  |  size   |type|    msg id         |
  +----+----+----+----+----+----+----+----+
  |   short exp       |     message       |
  +----+----+----+----+                   +
  |                                       |
  ~               .   .   .               ~
  |                                       |
  +----+----+----+----+----+----+----+----+

  blk :: 3
  size :: 2 bytes, big endian, size of type + msg id + exp + message to follow
          I2NP message body size is (size - 9).
  type :: 1 byte, I2NP msg type, see I2NP spec
  msg id :: 4 bytes, big endian, I2NP message ID
  short exp :: 4 bytes, big endian, I2NP message expiration, Unix timestamp, unsigned seconds.
               Wraps around in 2106
  message :: I2NP message body

{% endhighlight %}

Notes
`````
- Implementers must ensure that when reading a block,
  malformed or malicious data will not cause reads to
  overrun into the next block.



Termination
```````````
Drop the session.
This must be the last non-padding block in the frame.


.. raw:: html

  {% highlight lang='dataspec' %}
+----+----+----+----+----+----+----+----+
  | 4  |  size   |    valid data frames
  +----+----+----+----+----+----+----+----+
      received   | rsn|     addl data     |
  +----+----+----+----+                   +
  ~               .   .   .               ~
  +----+----+----+----+----+----+----+----+

  blk :: 4
  size :: 2 bytes, big endian, value = 9 or more
  valid data frames received :: The number of valid AEAD data phase frames received
                                (current receive nonce value)
                                0 if error occurs in handshake phase
                                8 bytes, big endian
  rsn :: reason, 1 byte:
         0: normal close or unspecified
         1: termination received
  addl data :: optional, 0 or more bytes, for future expansion, debugging,
               or reason text.
               Format unspecified and may vary based on reason code.

{% endhighlight %}

Notes
`````

Not all reasons may actually be used, implementation dependent.
Additional reasons listed are for consistency, logging, debugging, or if policy changes.




Options
```````
Pass updated options.
Options include: TBD.

Options block will be variable length.


.. raw:: html

  {% highlight lang='dataspec' %}
+----+----+----+----+----+----+----+----+
  | 5  |  size   |STL |OTW |STimeout |MITW|
  +----+----+----+----+----+----+----+----+
  |flg |         more_options             |
  +----+                                  +
  |                                       |
  ~               .   .   .               ~
  |                                       |
  +----+----+----+----+----+----+----+----+

  blk :: 5
  size :: 2 bytes, big endian, size of options to follow, TBD bytes minimum
  STL :: Session tag length (default 8), min and max TBD
  OTW :: Outbound Session tag window (max lookahead)
  STimeout :: Session idle timeout
  MITW :: Max Inbound Session tag window (max lookahead)
  flg :: 1 byte flags
         bit order: 76543210
         bit 0: 1 to request a ratchet (new key), 0 if not
         bits 7-1: Unused, set to 0 for future compatibility

  more_options :: Format TBD

{% endhighlight %}


Options Notes
`````````````
- Support for non-default session tag length is optional,
  probably not necessary

- The tag window is MAX_SKIP in the Signal documentation.



Options Issues
``````````````
- Options format is TBD.
- Options negotiation is TBD.
- Padding parameters also?


Message Numbers
```````````````

The message's number in the sending chain (N=0,1,2,...)
and the length (number of message keys) in the previous sending chain (PN).
Also contains the public key id, used for acks.


.. raw:: html

  {% highlight lang='dataspec' %}
+----+----+----+----+----+----+----+----+
  | 6  |  size   | key ID |   PN    |  N
 +----+----+----+----+----+----+----+----+
      |
 +----+

  blk :: 6
  size :: 6
  Key ID :: 2 bytes big endian
  PN :: 2 bytes big endian. The number of keys in the previous sending chain.
        i.e. one more than the last 'N' sent in the previous chain.
        Use 0 if there was no previous sending chain.
  N :: 2 bytes big endian. Starts with 0.

{% endhighlight %}


Notes
``````
- Maximum PN and N is 65535. Do not allow to roll over. Sender must ratchet the DH key, send it,
  and receive an ack, before the sending chain reaches 65535.

- N is not strictly needed in an existing session message, as it's associated with the Session Tag

- This is similar to what Signal does, but in Signal, PN and N are in the header.
  Here, they're in the encrypted message body.

- Key ID can be just an incrementing counter.
  It may not be strictly necessary, but it's useful for debugging.
  Also, we use it for explicit ACKs.
  Signal does not use a key ID.




Next DH Ratchet Public Key
``````````````````````````
This is in the header in Signal, we put it in the payload,
and it is optional. We don't ratchet every time.
For typical usage patterns, Alice and Bob each ratchet a single time
at the beginning.


.. raw:: html

  {% highlight lang='dataspec' %}
+----+----+----+----+----+----+----+----+
  | 7  |  size   |  key ID |              |
  +----+----+----+----+----+              +
  |                                       |
  +                                       +
  |     Next DH Ratchet Public Key        |
  +                                       +
  |                                       |
  +                        +----+----+----+
  |                        |
  +----+----+----+----+----+

  blk :: 7
  size :: 34
  key ID :: 2 bytes, big endian, used for ack
  Public Key :: The next public key, 32 bytes, little endian

  TBD :: Format TBD

{% endhighlight %}



Notes
``````

- Key ID can be just an incrementing counter.
  It may not be strictly necessary, but it's useful for debugging.
  Also, we use it for explicit ACKs.
  Signal does not use a key ID.


Issues
``````



Ack
```
This is only if an explicit ack was requested by the far end.
Multiple acks may be present to ack multiple messages.



.. raw:: html

  {% highlight lang='dataspec' %}
+----+----+----+----+----+----+----+----+
  | 8  |  size   |  key id |   N     |    |
  +----+----+----+----+----+----+----+    +
  |             more acks                 |
  ~               .   .   .               ~
  |                                       |
  +----+----+----+----+----+----+----+----+

  blk :: 8
  size :: 4 * number of acks to follow, minimum 1 ack
  for each ack:
  key ID :: 2 bytes, big endian, from the message being acked
  N :: 2 bytes, big endian, from the message being acked


{% endhighlight %}


Notes
``````
See ACK section above for more information.


Issues
``````



Ack Request
```````````
To replace the out-of-band DeliveryStatus Message in the Garlic Clove.
Also (optionally) binds the outbound session to the far-end Destination or Router.

If an explicit ack is requested, the current key ID and message number (N)
are returned in an ack block. When a next public key is included,
any message sent to that key constitutes an ack, no explicit ack is required.



.. raw:: html

  {% highlight lang='dataspec' %}
+----+----+----+----+----+----+----+----+
  |  9 |  size   |flg |                   |
  +----+----+----+----+                   +
  |    Garlic Clove Delivery Instructions |
  ~               .   .   .               ~
  |                                       |
  +----+----+----+----+----+----+----+----+
  | SigType |                             |
  +----+----+                             +
  |  (opt) Signature of next Public Key   |
  ~               .   .   .               ~
  |                                       |
  +----+----+----+----+----+----+----+----+

  blk :: 9
  size :: varies, typically 100
  session ID :: reverse session ID, length TBD
  flg :: 1 byte flags
         bit order: 76543210
         bit 0: 1 if explicit ack is requested, 0 if not
         bit 1: 1 if delivery instructions included, 0 if not
         bit 2: 1 if signature is present, 0 if not
         bits 7-3: Unused, set to 0 for future compatibility
  flag :: 1 byte
  Delivery Instructions :: as defined in I2NP spec, 33 bytes for DESTINATION type
  Sig Type :: Type of signature to follow
              Only present if flag is set and delivery instruction type is DESTINATION or ROUTER
              Typically 7 (Ed25519)
  Signature :: Signature of the the next DH ratchet public key,
               by the Destination or RouterIdentity's signing private key
               (online or offline)
               Can only be verified if receiver has the RI or LS.
               Only present if flag is set and delivery instruction type is DESTINATION or ROUTER
               Typically 64 bytes (Ed25519)


{% endhighlight %}


Notes
``````

- When the delivery instructions contains the hash of the destination,
  and the session is not previously bound, this binds the session to the destination.

- After a session is bound, any subsequent destination delivery instructions must contain
  the same hash as previously, or this is an error.

- If the receiving router does not have a current lease set,
  verification of the signature must be deferred until after processing the
  I2NP block, which will hopefully contain a clove with the lease set.

- See ACK section above for more information.


Issues
``````

- Java router must have the actual signing private key, not a dummy,
  see new I2CP Create LeaseSet2 Message in proposal 123.

- For easier processing, LS clove should precede Garlic clove in the message.

- Is the next public key the right thing to sign?



Padding
```````
This is for padding inside AEAD frames.
Padding for messages 1 and 2 are outside AEAD frames.
All padding for message 3 and the data phase are inside AEAD frames.

Padding inside AEAD should roughly adhere to the negotiated parameters.
Bob sent his requested tx/rx min/max parameters in message 2.
Alice sent her requested tx/rx min/max parameters in message 3.
Updated options may be sent during the data phase.
See options block information above.

If present, this must be the last block in the frame.



.. raw:: html

  {% highlight lang='dataspec' %}
+----+----+----+----+----+----+----+----+
  |254 |  size   |      padding           |
  +----+----+----+                        +
  |                                       |
  ~               .   .   .               ~
  |                                       |
  +----+----+----+----+----+----+----+----+

  blk :: 254
  size :: 2 bytes, big endian, size of padding to follow
  padding :: random data

{% endhighlight %}

Notes
`````
- Padding strategies TBD.
- Minimum padding TBD.
- Padding-only frames are allowed.
- Padding defaults TBD.
- See options block for padding parameter negotiation
- See options block for min/max padding parameters
- Noise limits messages to 64KB. If more padding is necessary, send multiple frames.
- Router response on violation of negotiated padding is implementation-dependent.


Other block types
`````````````````
Implementations should ignore unknown block types for
forward compatibility, except in message 3 part 2, where
unknown blocks are not allowed.


Future work
```````````
- The padding length is either to be decided on a per-message basis and
  estimates of the length distribution, or random delays should be added.
  These countermeasures are to be included to resist DPI, as message sizes
  would otherwise reveal that I2P traffic is being carried by the transport
  protocol. The exact padding scheme is an area of future work, Appendix A
  provides more information on the topic.










Common Structures Spec Changes Required
=======================================

TODO


Key Certificates
----------------



Encryption Spec Changes Required
================================

TODO



I2NP Changes Required
=====================

TODO



I2CP Changes Required
=====================

I2CP Options
------------

This section is copied from proposal 123.

New options in SessionConfig Mapping:

::

  i2cp.leaseSetEncType=nnn  The encryption type to be used.
                            0: ElGamal
                            4: This proposal.
                            Other values to be defined in future proposals.


Create Leaseset2 Message
------------------------

See proposal 123 for specification.


SAM Changes Required
====================

TODO



BOB Changes Required
====================

TODO




Publishing, Migration, Compatibility
====================================

TODO



References
==========

.. [Prop111]
    {{ proposal_url('111') }}

.. [Prop123]
    {{ proposal_url('123') }}

.. [Prop142]
    {{ proposal_url('142') }}

.. [RFC-2104]
    https://tools.ietf.org/html/rfc2104

.. [RFC-7539]
    https://tools.ietf.org/html/rfc7539

.. [RFC-7748]
    https://tools.ietf.org/html/rfc7748

.. [RFC-7905]
    https://tools.ietf.org/html/rfc7905

.. [RFC-4880-S5.1]
    https://tools.ietf.org/html/rfc4880#section-5.1

.. [Signal]
    https://signal.org/docs/specifications/doubleratchet/
