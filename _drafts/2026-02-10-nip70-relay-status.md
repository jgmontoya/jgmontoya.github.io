---
layout: post
title: "The State of NIP-70: Why We're Rolling Back Protected Events in Marmot"
---

At WhiteNoise, we're building secure, private messaging on top of the Marmot protocol (which combines Nostr and **MLS (Messaging Layer Security)**). A fundamental building block of MLS is the **KeyPackage**, a pre-published bundle containing a public `init_key` that allows other users to add you to a group asynchronously.

**Why does this matter for security?**
From a cryptographic perspective, KeyPackages are a liability. If the private counterpart to an `init_key` sits on a device for too long, it increases the risk of a "harvest now, decrypt later" attack. If an attacker records your encrypted "Welcome" messages today and steals your private keys years from now, they could decrypt your past group joins.

To prevent this, **Forward Secrecy** is mandatory. Our implementation (MDK) actively manages this lifecycle:

1. **Consumption:** When a KeyPackage is used to join a group, we zeroize (securely delete) the private `init_key` from the device.
2. **Rotation:** We publish fresh KeyPackages to ensure new invites can always come through.
3. **Cleanup:** Crucially, we attempt to **delete** the old, spent KeyPackages from relays (using [NIP-09](https://github.com/nostr-protocol/nips/blob/master/09.md) Kind 5 deletions) so they don't linger as "ghost" invites that no longer work.

To ensure these deletion requests are secure (and to prevent unauthorized parties from republishing these critical packages), we turned to two specific Nostr Improvement Proposals: **[NIP-70 (Protected Events)](https://github.com/nostr-protocol/nips/blob/master/70.md)** and **[NIP-42 (Authentication)](https://github.com/nostr-protocol/nips/blob/master/42.md)**.

### A Quick Primer

For those less familiar with the depths of these Nostr protocol specs:

- **[NIP-70 (Protected Events)](https://github.com/nostr-protocol/nips/blob/master/70.md):** This standard allows a user to "protect" an event by adding a special `-` tag. It signals to relays that _only_ the original author should be allowed to publish this event, preventing third-party republishing. It's a defense against griefing and unauthorized replacements.
- **[NIP-42 (Authentication)](https://github.com/nostr-protocol/nips/blob/master/42.md):** This is the handshake mechanism. When a relay sees a protected event, it shouldn't just take the client's word for it. It should challenge the client to prove their identity via a cryptographic challenge-response flow.

Ideally, these two work in tandem: NIP-70 sets the rule ("Protect this!"), and NIP-42 provides the key ("I am allowed to edit this").

![The State of NIP-70: Protected Events](/assets/images/nip70-protected-door.jpeg)

## The Reality Check

Recently, we started seeing issues where our MLS key packages (Kind 443) were being rejected by relays. We decided to investigate thoroughly. We built a custom tool to probe relay behavior specifically around NIP-70 and NIP-42.

The source code for our investigation tool is here: [marmot-protocol/mdk (branch: inv/nip70-relay-tests)](https://github.com/marmot-protocol/mdk/tree/inv/nip70-relay-tests/relay-tests).

### What We Found

The results highlighted a gap between our expectations and the spec's design. NIP-70 explicitly states that **the default relay behavior MUST be to reject protected events**. Supporting them via AUTH is opt-in for relay operators. When we tested major public relays (including Damus, Primal, and nos.lol), they were following this default behavior exactly:

1. **Rejection without Recourse:** The relays rejected the events with messages like `blocked: event marked as protected`.
2. **No Auth Challenge:** Crucially, _none_ of these relays initiated the NIP-42 `AUTH` flow. They simply followed the spec's default: reject outright.

Without the `AUTH` challenge, the client has no way to prove it is the owner of the key. The door is simply shut.

### The "Do Nothing" Solution

This leaves us in a difficult position. We want to protect user data, but the relay ecosystem doesn't support the standard mechanism for doing so.

As discussed in our [GitHub issue #168](https://github.com/marmot-protocol/mdk/issues/168), we are forced to disable NIP-70 protection by default in our reference implementation for now.

While the **[Marmot Protocol Specification (MIP-00)](https://github.com/marmot-protocol/marmot/blob/master/00.md)** already treats the protected (`-`) tag as optional to allow for flexibility, the **Marmot DevKit (MDK)** previously enforced it by default to maximize security. We are now aligning the implementation with the reality of the relay ecosystem.

This change is implemented in **[MDK Pull Request #173](https://github.com/marmot-protocol/mdk/pull/173)**. The standard `create_key_package_for_event` function in MDK now returns a `Vec<Tag>` without the protected tag. For users who still want to enable protection (perhaps on relays that support it), we’ve introduced `create_key_package_for_event_with_options`, which accepts a `protected` boolean flag.

![The State of NIP-70: Retreat](/assets/images/nip70-retreat.jpeg)

It feels like a step backward, essentially removing protocol enforcement, but we can't build a reliable messaging protocol if users can't publish their key packages to the relays they actually use.

## A Call to Action for Relay Operators

We believe NIP-70 and NIP-42 together could enable important security properties for Nostr. Ephemeral data and user-controlled deletion are security features, not just nice-to-haves.

The current spec makes AUTH support for protected events optional, and most relays have chosen the simpler path of outright rejection. We'd encourage relay operators to consider implementing the full AUTH flow:

1. See a `protected` event.
2. Send an `AUTH` challenge.
3. Accept the write once the client proves ownership.

Until broader adoption occurs, we have to build around the limitations of the current ecosystem. We're optimistic that as Nostr matures, more relays will see the value in supporting authenticated writes for protected events.
