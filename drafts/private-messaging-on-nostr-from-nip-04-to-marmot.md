---
layout: post
title: "Private Messaging on Nostr: From NIP-04 to Marmot"
date: 2026-07-13
permalink: /drafts/private-messaging-on-nostr-from-nip-04-to-marmot/
draft: true
---

> This article is based on a talk I gave at LABITCONF in Buenos Aires on November 8, 2025. It preserves the design I presented and explains how Marmot has changed since.

Encrypting a message body solves one requirement. A private group messaging system also has to manage membership, rotate keys, deliver messages while people are offline, and keep clients converged on a canonical group state. It needs separate recovery plans for compromised group secrets, credentials, and devices.

Public Nostr is easier to reason about. Users sign events, publish them to relays, and fetch them from whichever relays they choose. Private communication changes the problem. The relays should carry the data without learning the conversation, and no central provider should own the identities, group state, or message history.

The title slide I delivered read "Private Messaging on Nostr: From NIP-04 to Marmot," with the subtitle "Building unstoppable & secure communication."

## Two Titles, One Transition

The [speaker announcement](https://nostr.com/nevent1qqsvctka4ulapz7l6z0yvsyhqxrxg5xzeyejghpeqhcr9patwvyqtvszyz9j64tndhd58f7f5ejpxj7c592jclf6yt7mq07xa5wg8gmc5tcejwduynz) and [conference-day schedule](https://nostr.com/nevent1qqsdd9c2lnh33mt36ey9dxfrrehs85e5yhya50c92xvkpysdhju789czyz9j64tndhd58f7f5ejpxj7c592jclf6yt7mq07xa5wg8gmc5tcejy9gszr) used the earlier Spanish title "Mensajería privada en Nostr: De NIP-04 a NIP-EE," or "Private Messaging on Nostr: From NIP-04 to NIP-EE." The schedule listed the session at 12:05 p.m. on the second conference day.

By the time I delivered the talk, the title slide said Marmot, although both names belong in the record. [NIP-EE](https://github.com/nostr-protocol/nips/blob/3a126a51a65bcf97a335a6897574254c92334cec/EE.md) had merged into the Nostr NIPs repository in August 2025. A separate [Marmot specification](https://github.com/marmot-protocol/marmot/tree/50c4201cbb2e400ee5d895566441350a9b5f4240) followed in September and already described NIP-EE as the earlier design by the week of the talk.

The final deck used NIP-EE for the historical step and Marmot for the protocol work I was presenting.

## Encrypted Content Is Only the Start

My slides began with a simple observation: centralized service operators and infrastructure are clear pressure points for blocking, monitoring, or coercion. Nostr ties identity to a public key and lets clients distribute event delivery across relays, so users can change relays without changing their identity. This removes one kind of dependency; secure group messaging still requires more.

![The completed LABITCONF slide comparing the complexity of messaging groups with 2, 10, and 10,000 members](/assets/images/labitconf-group-messaging.png)

With two people, each side can derive a shared secret. A group carries more state as members join and leave or keys change after a removal or compromise. Offline clients may fall behind and retain old key material. Honest current members have to converge on the same epoch, membership, and shared state.

The problem becomes painful when a protocol encrypts the same message separately for every recipient. Ten people may be manageable. Ten thousand people expose the limits immediately.

## NIP-04: The Early Baseline

[NIP-04](https://github.com/nostr-protocol/nips/blob/8f8444d05a8842c40211ded5d10af3521541f865/04.md) gave early Nostr clients a direct-message format. A `kind:4` event encrypted its content with AES-256-CBC using a shared secret derived from the sender's private key and the recipient's public key. A public `p` tag identified the recipient so relays knew where to deliver the event.

It was useful because it established a common format. Its scope was much smaller than the private messaging systems users now expect.

The sender public key, recipient `p` tag, and event timestamp were visible. Anyone who could read the event could see the participants, the event's claimed timestamp, and the number and timing of captured events. The protocol defined no group membership state and no key ratchet. Since its conversation secret came from long-term identity keys, compromising either participant's private key could expose captured messages and continue exposing new ones.

The current NIP-04 specification marks it `unrecommended` and says it does not approach the state of the art for encrypted peer communication. I used the deliberately sharp title "The broken beginning" in my slides. A more useful description is an early building block whose security model stopped at encrypted content.

## NIP-17 and Gift Wraps

[NIP-17](https://github.com/nostr-protocol/nips/blob/8f8444d05a8842c40211ded5d10af3521541f865/17.md) improved the design by combining [NIP-44 encrypted payloads](https://github.com/nostr-protocol/nips/blob/8f8444d05a8842c40211ded5d10af3521541f865/44.md) with [NIP-59 Gift Wraps](https://github.com/nostr-protocol/nips/blob/8f8444d05a8842c40211ded5d10af3521541f865/59.md).

The message is built in three layers:

1. A rumor is an unsigned Nostr event containing the real message.
2. A seal encrypts that rumor and is signed by the real sender.
3. A Gift Wrap encrypts the seal and is signed with a fresh ephemeral key.

![The complete NIP-59 diagram showing a rumor inside a seal inside a Gift Wrap](/assets/images/labitconf-gift-wrap.png)

The public wrapper no longer identifies the real sender, and clients should randomize its timestamp up to two days into the past. The Gift Wrap hides the signed seal from public observers, while the recipient can authenticate the sender after decrypting it. This makes public correlation much harder.

Delivery still leaves traces. The outer event needs recipient routing information, and relays can observe which connections publish and fetch events. Encryption cannot hide an endpoint that has already been compromised either.

NIP-17 also supports rooms with more than two participants. Each message must be wrapped separately for every receiver and once for the sender, allowing the sender's other clients to recover that copy. Fan-out therefore grows linearly with the room. The current specification recommends a different scheme for groups larger than ten participants; the [version current during the talk](https://github.com/nostr-protocol/nips/blob/3ec830cd2331ba0ed3113fc05a44c87deb9785be/17.md) set that threshold at one hundred. NIP-17 defines no shared evolving cryptographic group state, and its NIP-44 conversation keys provide no post-compromise security.

NIP-17 solved several real NIP-04 problems. Large, long-lived groups still needed another design.

## Enter MLS

[Messaging Layer Security](https://www.rfc-editor.org/rfc/rfc9420.html), or MLS, is an IETF Standards Track protocol for asynchronous encrypted groups. It models a group as a sequence of epochs. Each epoch has a specific membership and shared cryptographic state. A Commit moves the group to a new epoch, including when someone joins, leaves, or updates their keys.

MLS organizes members in a ratchet tree. Updating the path through that tree grows logarithmically with the number of members. A tree accommodating 10,000 members has a depth of 14, which explains the comparison in my slides. That number describes tree-update work. It is not the total cost of every MLS operation.

The ratchet tree, update mechanism, and deletion rules enable two security properties that NIP-04 and NIP-17 do not provide in the same way:

- Forward secrecy protects older messages after current key material is compromised. All clients retaining the relevant material have to delete old secrets for this guarantee to hold.
- Post-compromise security can restore future message secrecy after ratchet or group secrets are compromised, once the attacker loses access to fresh secrets and members process a relevant Commit with an UpdatePath. A leaked signature key also requires credential rotation or revocation, as the [MLS architecture](https://www.rfc-editor.org/rfc/rfc9750.html#section-8.3) explains.

The phrase "self-healing" fits on a slide. The implementation contract is stricter. Clients need to update keys, process Commits, delete old material, and handle members that stay offline for long periods. An attacker who still controls an endpoint can keep reading what that endpoint can read.

MLS supplies continuous group key agreement, message framing, and state transitions. Applications still need an authentication service, a delivery service, state and retention policies, and a way to resolve conflicting Commits. That boundary is where Nostr and Marmot enter the design.

## From NIP-EE to Marmot

The 2025 design used Nostr public keys for identity and Nostr relays for delivery. MLS provided the shared group state. Marmot specified how those pieces fit together.

![The delivered slide describing MLS as the cryptographic foundation, Nostr as authentication and transport, and Marmot as their integration protocol](/assets/images/labitconf-mls-nostr-marmot.png)

The concrete flow from the talk followed Alice as Bob added her to a group:

1. Alice published a relay-list event telling other clients where to find her MLS `KeyPackage`.
2. Alice published the `KeyPackage`, a signed object containing the public material needed to add her.
3. Bob fetched it and created an MLS Commit that added Alice to the next group epoch.
4. After Bob's Commit was accepted for that epoch, he sent Alice the corresponding Welcome message inside a NIP-59 Gift Wrap.
5. Alice processed the Welcome and ratchet tree, initialized her state for the new epoch, and could read and send new group messages.

![The complete Marmot flow for Bob adding Alice to an MLS group over Nostr](/assets/images/labitconf-marmot-add-member.png)

This flow matters because Alice can be offline when Bob invites her. In the 2025 design, she had already published her KeyPackage to relays, and Bob published the Gift Wrapped Welcome to her inbox relays for her to fetch later. That delivery depends on at least one relay accepting, retaining, and serving each event; Nostr does not guarantee persistence. No central chat server holds the MLS group secrets. Bob and Alice's clients still have to reject wrong-epoch or invalid transitions and converge on one canonical Commit and epoch.

At the time of the talk, the Marmot specifications and the Rust-based Marmot Development Kit were public and explicitly experimental. White Noise's source was also public, with releases still listed as coming soon. Developers could inspect and test the work, but neither the protocol nor the applications were finished.

## What "Unstoppable" Means

"Unstoppable" is easy to turn into a slogan, so I want to give it a narrow definition.

The design has no single messaging provider that controls every account and conversation. Users keep portable public-key identities. Clients can publish through multiple relays. A blocked or failed relay does not have to stop the group, and independent developers can implement the protocol.

Servers still exist, and relays can be blocked, monitored, or operated maliciously. Devices can leak keys, while backups can destroy forward secrecy. Traffic analysis can reveal communication patterns even when event contents are unreadable, and poor recovery UX can make a secure protocol unusable.

An open protocol distributes control and failure across several operators and implementations. It leaves endpoint security, metadata minimization, key storage, interoperability, and usable recovery as engineering work.

## Marmot Today

Marmot has continued to change since November 2025. As of July 13, 2026, the [current specification snapshot](https://github.com/marmot-protocol/marmot/tree/7f2f5fac4b0e8648d820b20d27d670a6c139e717) marks its status as adopted and calls the MIP-era documents from my talk deprecated.

The current design keeps three ideas: Nostr public keys identify accounts, application payloads inside MLS use Nostr event shapes, and MLS manages continuous group key agreement. Transport now has its own boundary. Nostr relays remain a supported durable transport, while the core protocol no longer assumes they are the only possible transport.

The [MDK snapshot from July 13, 2026](https://github.com/marmot-protocol/mdk/tree/894e768bef077be4fac7856af7315a7bf731cd5f) has also grown beyond the early library shown in the talk. It contains an OpenMLS-backed engine, SQLCipher-backed storage, conformance tooling, Nostr transport components, and application bindings.

Those are present-day facts. The diagrams above remain a record of the MIP-era design I presented at LABITCONF.

## The Work That Remains

The progression from NIP-04 to NIP-17, MLS, and Marmot expanded the problem each time. Encrypted content led to metadata protection, small rooms exposed fan-out costs, and group encryption required an evolving state machine. Running that state machine over an eventually consistent relay transport exposed application-level work around ordering, reliable delivery, retention, recovery, and interoperability.

The remaining test is concrete: independent clients must converge on the same canonical sequence of valid Commits and the same resulting group state. They need consistent validation and canonicalization rules, plus reporting and recovery when only some members can detect a malformed Commit. Recovery may require application-defined reconciliation, rejoining, or reinitializing the group, without asking users to understand the cryptography underneath.

That is much harder than encrypting a Nostr event. It is also the work required for private communication without a central gatekeeper.

You can read the [22-page reader edition of the slides](/output/pdf/labitconf-2025-private-messaging-on-nostr-reader-edition.pdf), the [post-event photograph record](https://nostr.com/nevent1qvzqqqqqqypzpzed24ekmk6r5ly6veqnf0v2z4fv05az9lds8lrw68yr5du29uveqywhwumn8ghj7mn0wd68ytnzd96xxmmfdejhytnnda3kjctv9uq3uamnwvaz7tmwdaehgu3dwp6kytnhv4kxcmmjv3jhytnwv46z7qpq6ercj9ydtx5z5cf6wj4hn5uu730r37qfa4003rt7jem74nqx9haslrm39j), and the [Marmot protocol](https://github.com/marmot-protocol/marmot) for the current design.
