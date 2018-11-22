=========================
ECIES-X25519-AEAD-Ratchet
=========================
.. meta::
    :author: zzz
    :created: 2018-11-22
    :thread: http://zzz.i2p/topics/2639
    :lastupdated: 2018-11-22
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
=====================

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
=====================

As a review,
we added support for encryption types when we added support for signature types.
The encryption type field is always zero, both in Destinations and RouterIdentities.
Whether to ever change that is TBD.
Reference the common structures specification.




Asymmetric Crypto Uses
======================

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
=====

- Backwards compatible
- Requires and builds on LS2 (proposal 123)
- Leverage new crypto or primitives added for NTCP2 (proposal 111)
- No new crypto or primitives required for support
- Maintain decoupling of crypto and signing; support all current and future versions
- Enable new crypto for destinations
- Don't break anything that relies on 32-byte binary destination hashes, e.g. bittorrent
- Add forward secrecy
- Add authentication (AEAD)
- Much more CPU-efficient than ElGamal
- Minimize DH operations
- Much more bandwidth-efficient than ElGamal (514 byte ElGamal block)
- Eliminate several problems with session tags, including:
  - Inability to use AES until the first reply
  - Unreliability and stalls if tag delivery assumed
  - Bandwidth inefficient, especially on first delivery
  - Huge space inefficiency to store tags
  - Huge bandwidth overhead for datagrams
  - Highly complex, difficult to implement
  - Difficult to tune for various use cases
  (streaming vs. datagrams, server vs. client, high vs. low bandwidth)
- Support new and old crypto on same tunnel
- Recipient is able to efficiently distinguish new from old crypto coming down
  same tunnel
- Others cannot distinguish new from old crypto
- Eliminate new vs. existing session length classification (support padding)
- No new I2NP messages required
- Replace SHA-256 checksum in AES payload with AEAD
- (Optimistic) Add extensions or hooks to support multicast
- Support binding of transmit and receive sessions so that
  acknowledgements may happen within the protocol, rather than solely out-of-band.


Non-Goals / Out-of-scope
========================

- LS2 format (see proposal 123)
- New DHT rotation algorithm or shared random generation
- The specific new encryption type and end-to-end encryption scheme
  to use that new type would be in a separate proposal.
  No new crypto is specified or discussed here.
- New encryption for tunnel building.
  That would be in a separate proposal.
- Methods of encryption, transmission, and reception of I2NP DLM / DSM / DSRM messages.
  Not changing.
- Threat model changes
- Offline storage format, or methods to store/retrieve/share the data.
- Implementation details are not discussed here and are left to each project.



Justification
-------------


Proposal
========

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


Sessions
--------

The current ElGamal/AES+SessionTag protocol is unidirectional.
At this layer, the receiver doesn't know where a message is from.
Outbound and inbound sessions are not associated.
Acknowledgements are out-of-band using a DeliveryStatusMessage
(wrapped in a GarlicMessage) in the clove.

There is substantial inefficiency in a unidirectional protocol.
Any reply must also use an expensive 'new session' message.

For the new protocol, a new outbound session is always paired with a new inbound session,
unless no reply is expected (e.g. raw datagrams).
A new inbound session is always paired with a new outbound session,
unless no reply is requested (e.g. raw datagrams).

If a reply is requested and bound to a far-end destination or router,
that new outbound session is bound to that destination or router,
and replaces any previous outbound session to that destination or router.

There is only one outbound session to a given destination or router.
There may be several current inbound sessions from a given destination or router.
Generally, when a new inbound session is created, and traffic is received
on that session (which serves as an ACK), any others will be marked
to expire relatively quickly, within a minute or so.




1) Container format
-------------------

In ElGamal/AES+SessionTags, there are two formats:

New session:
- 514 byte ElGamal block
- AES block (128 bytes minimum, multiple of 16)

Existing session:
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
If not found, and the data is at least (514+16) long, he attempts to decrypt the ElGamal block.



Session Tags and comparison to Signal
`````````````````````````````````````

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



Format
``````
Encrypted:

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
  |                Nonce                  |
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

  Public Key :: 32 bytes, cleartext

  Nonce :: 8 bytes, cleartext

  encrypted data :: Same size as plaintext data, 40 bytes

  MAC :: Poly1305 message authentication code, 16 bytes

{% endhighlight %}


Decrypted:

  See AEAD section below.





KDF
```

Key: KDF TBD
IV: As published in a LS2 property?
Nonce: From header



Justification
`````````````

Notes
`````

This allows sending multiple new session messages with the same cleartext key,
which is less privacy but much more efficient, e.g. for a POST.
Send different nonces with each message.
Nonce should be generated randomly.


Issues
``````


2a) Existing session format
---------------------------

Encrypted header (56 bytes)
Encrypted body (see section 3 below)


Format
``````
Encrypted:

.. raw:: html

  {% highlight lang='dataspec' %}
+----+----+----+----+----+----+----+----+
  |                                       |
  +                                       +
  |       Session Tag                     |
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

  Session Tag :: 32 bytes, cleartext

  encrypted data :: Same size as plaintext data, 40 bytes

  MAC :: Poly1305 message authentication code, 16 bytes

{% endhighlight %}


Decrypted:
  See AEAD section below.


KDF
```

Key: KDF TBD
IV: KDF TBD
Nonce: The message number N in the current chain, as retrieved from the associated Session Tag.



Justification
`````````````

Notes
`````


Issues
``````

- Can we reduce session tag to 16 bytes? 8 bytes?

- Obfuscation of key?



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
           Last eight bytes are the counter, little-endian encoded.
           Maximum value is 2**64 - 2.
           Connection must be dropped and restarted after
           it reaches that value.
           The value 2**64 - 1 must never be sent.

  ad :: In new session message:
        Associated data, 32 bytes.
        The SHA256 hash of all preceding data.
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

Ratchets replace session tags.
Session tags also have a rekey option that we never implemented.
So it's like a double ratchet but we never did the second one.

Here we define something like Signal Double Ratchet with Header Encryption.

By using ratchets, we eliminate memory usage on the sender side.
Receiver side usage is still significant.

We do not use header encryption as is specified (and optional) in Signal,
we use session tags instead.


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
``````````````````

Ratchets for every message, as in Signal.
The session tag ratchet is synchronized with the symmetric key ratchet,
but the receiver key ratchet may "lag behind" to save memory.

Transmitter ratchets once for each message transmitted.
No additional tags must be stored.

Receiver must ratchet ahead by the max window size and store the tags in a "tag set",
which is associated with the session.
Once received, the stored tag may be discarded, and if there are no previous
unreceived tags, the window may be advanced.


Symmetric Key Ratchet
`````````````````````

Ratchets for every message, as in Signal.
Each symmetric key has an associated message number and session tag.
The session key ratchet is synchronized with the symmetric tag ratchet,
but the receiver key ratchet may "lag behind" to save memory.

Transmitter ratchets once for each message transmitted.
No additional keys must be stored.

When receiver gets a session tag, if it has not already ratcheted the
symmetric key ratchet ahead to the associated key, it "catch up" to the associated key.
The receiver will probably cache the keys for any previous tags
that have not yet been received.
Once received, the stored key may be discarded, and if there are no previous
unreceived tags, the window may be advanced.


DH Ratchet
``````````

Ratchets but not nearly as fast as Signal does.
We separate the ack of the received key from generating the new key.
In typical usage, Alice and Bob will each ratchet immediately in a new session,
but will not ratchet again.



5) Payload
----------

This replaces the AES section format defined in the ElGamal/AES+SessionTags specification.

This uses the same block format as defined in the NTCP2 specification.
Where appropriate, the same block numbers are used.
We do not reuse NTCP2 block numbers for different uses, so
that implementations may share code with NTCP2.


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
         3 for I2NP message (Garlic message only)
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
  | 4  |  size   |    valid data frames   |
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
  | 5  |  size   |STL |STW |STimeout |flg |
  +----+----+----+----+----+----+----+----+
  |              more_options             |
  ~               .   .   .               ~
  |                                       |
  +----+----+----+----+----+----+----+----+

  blk :: 5
  size :: 2 bytes, big endian, size of options to follow, TBD bytes minimum
  STL :: Session tag length
  STW :: Session tag window (max lookahead)
  STimeout :: Session idle timeout
  flg :: 1 byte flags
         bit order: 76543210
         bit 0: 1 to request a ratchet (new key), 0 if not
         bits 7-1: Unused, set to 0 for future compatibility

  more_options :: Format TBD

{% endhighlight %}


Options Issues
``````````````
- Options format is TBD.
- Options negotiation is TBD.



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
  PN :: 2 bytes big endian
  N :: 2 bytes big endian

{% endhighlight %}


Notes
``````
- Maximum PN and N is 65535. Do not allow to roll over. Sender must ratchet the DH key, send it,
  and receive an ack, before the sending chain reaches 65535.

- N is not strictly needed in an existing session message, as it's associated with the Session Tag



Next DH Ratchet Public Key
``````````````````````````
This is in the header in Signal, we put it in the payload,
and it is optional. We don't ratchet every time.

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
  Public Key :: The next public key, 32 bytes

  TBD :: Format TBD

{% endhighlight %}


Issues
``````



Ack
```
This is only if an explicit ack is requested.
Multiple blocks may be present to ack multiple messages.



.. raw:: html

  {% highlight lang='dataspec' %}
+----+----+----+----+----+----+----+
  | 8  |  size   |  key id |   N     |
  +----+----+----+----+----+----+----+

  blk :: 8
  size :: 4
  key ID :: 2 bytes, big endian, from the message being acked
  N :: 2 bytes, big endian, from the message being acked


{% endhighlight %}


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
  size :: varies, typically TBD
  session ID :: reverse session ID, length TBD
  flg :: 1 byte flags
         bit order: 76543210
         bit 0: 1 if signature is present, 0 if not
         bit 1: 1 if explicit ack is requested, 0 if not
         bits 7-2: Unused, set to 0 for future compatibility
  flag :: 1 byte
  Delivery Instructions :: as defined in I2NP spec
  Sig Type :: Type of signature to follow
              Only present if flag is set and delivery instruction type is DESTINATION or ROUTER
  Signature :: Signature of the the next DH ratchet public key,
               by the Destination or RouterIdentity's signing private key
               (online or offline)
               Can only be verified if receiver has the RI or LS.
               Only present if flag is set and delivery instruction type is DESTINATION or ROUTER


{% endhighlight %}


Issues
``````

- Java router does not have signing private key and can't sign anything,
  must be fixed in I2CP to deliver it

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

TODO

I2CP Options
------------

This section is copied from proposal 123.

New options in SessionConfig Mapping:

::

  crypto.encType=nnn      The encryption type to be used.
                          0: ElGamal
                          Other values to be defined in future proposals.
  i2cp.leaseSetType=nnn   The type of leaseset to be sent in the Create Leaseset Message
                          Value is the same as the netdb store type in the table above.




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
