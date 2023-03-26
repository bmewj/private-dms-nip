NIP-TBD
=======

Incognito Direct Messages (Draft)
---------------------------------

`draft` `optional` `depends:16` `author:antonleviathan` `author:bartjoyce` `author:styppo` `author:neo ??? TODO`

This is very similar to NIP-59 but with some improvements.

This NIP defines a scheme for users to publish disposable identity key pairs on which to exchange DMs with each other to prevent public inbox peeking. Clients can either entirely abstract away this behavior and present incognito messages identically to regular DMs, or can present them as separate from regular _"public"_ DMs.

This NIP defines a `kind: 30004` parameterized replaceable event where users keep track of their generated throwaway pubkeys.

## Motivation

There is an ongoing discussion regarding NIP-04 and how its privacy problems can be remediated.

To achieve state-of-the-art private messaging on Nostr, we'd have to implement a schema leveraged by existing solutions like Signal, Sphinx, or Nym on Nostr, which we believe to not be in the spirit of Nostr, and are quite complex to implement, for example in the case of a mixer network based approach. For true private messages, users should go somewhere else.

That being said, we can make an effort to improve it today.

## Definitions

- `invite` A regular NIP-04 message that contains an invitation to start an incognito conversation.
- `identity` A standard nostr keypair. We make a distinction between "well known" `identities` and "disposable" `identities`, the former being a keypair that you are known by, and the latter being new identities which are not visibly linked to a "well known" identity, and as such, are anonymous.
- `[name]`: A public keypair identifying the person, and also used as reference to the actor, for example `Bob` or `Alice`
- `sk[name]` A secret key that belongs to a person
- `pk[name]` A public key that belongs to a person
- `[key][name]'`: A key which is disposable. For example, the "well known" npub key of Bob would be `pkBob`, the first disposable identity we generate `pkBob'` and the second disposable identity `pkBob''`. 

## General flow

![Messages Diagram](https://raw.githubusercontent.com/bartjoyce/private-dms-nip/master/message-diagram.jpg)

Bob wants to send incognito messages to Alice.

### 1. Generate disposable identities

Bob generates two new disposable identities: 
* `Bob'` (`pkBob'` & `skBob'`)
* `Bob''` (`pkBob''` & `skBob''`).

### 2. Send invitation to Alice

Bob publishes the following NIP-04 style message:

```json
{
  "id": "...",
  "pubkey": <pkBob'>, // This identity is not tied to `Bob`
  "kind": 4,
  "created_at": 12345678,
  "tags": [
    ["p", <pkAlice>]
  ],
  "content": <ciphertext>
}
```

The `content`'s plaintext will look like this:

> Someone wants to start an incognito conversation with you.
> Your client may not support this behavior, you can find
> a list of clients here who support incognito messaging:
> https://nostrincognito.com/clients
>
>
> `invite:<pkBob>:<pkBob''>:<relay>:<sigBob>`

Alternative with JSON (might make it easier to implement parsing of the setup message):

> Someone wants to start an incognito conversation with you.
> Your client may not support this behavior, you can find
> a list of clients here who support incognito messaging:
> https://nostrincognito.com/clients
>
> ---
> `{"id": "...","pubkey": <pkBob>,"kind": 20004,"created_at": <current timestamp?>,"tags": [["p", <pkAlice>]],"content": "","sig": <sigBob>}`

The message can include any extra human-friendly information to guide users on clients who don't support incognito messages. This preserves backwards compatibility for clients that don't support incognito messages.

Clients who support incognito messages should look for the presence of an `invite:_:_:_:_` code. All three parameters should be hex.

The invite code proves that Bob (`pkBob`) is the real originator of the message, that `pkBob''` is the conversation pubkey that Alice should subscribe to receive incognito messages from Bob, and that Bob possesses `skBob`.

The `relay` is the relay Bob is going to post messages to for this conversation. This is an improvement over NIP-59 where the sender specifies the relay where they are listening for messages, since the counterparty might not be able to write to that relay.

The signature `<sigBob>` is taken from the following throwaway event template:

```json
{
  "id": "...",
  "pubkey": <pkBob>,
  "kind": 0,
  "created_at": 0, // Must be 0
  "tags": [
    ["p", <pkAlice>],
    ["p", <pkBob''>]
  ],
  "content": "",
  "sig": <sigBob>
}
```

In other words, it's a signature of `[0, <pkBob>, 0, 0, [["p", <pkAlice>], ["p", <pkBob''>]], ""]`.

On receipt, Alice can drop in her public key and `pkBob` into the event template and verify the signature.

TODO Multiple setup messages to rotate identity/relay

### 3. Send an incognito message

Once Bob has sent his invitation to Alice he can send incognito messages
using his new disposable identity `pkBob''`.

To send a message he first constructs a regular DM event:

```json
{
  "id": "...",
  "pubkey": <pkBob>,
  "kind": 4,
  "created_at": 12345678,
  "tags": [
    ["p", <pkAlice>]  
  ],
  "content": <ciphertext>,
  "sig": <sig>
}
```

Next, he wraps the event JSON and transmits that as a DM from `Bob''`:

```json
{
  "id": "...",
  "pubkey": <pkBob''>,
  "kind": 4,
  "created_at": 12345678,
  "tags": [
    ["p", <pkRandom>]  
  ],
  "content": <encrypted JSON of regular DM>,
  "sig": <sig>
}
```

For this event the recipient can be any random pubkey and can be
ignored by Alice. All events from `pkBob''` are for her.

**Note:** The `.content` of this message should be encrypted for `pkAlice`,
not for `pkRandom`.

## Implementation Details
The client implements logic, leveraging a library `nostr-incognito` which provides methods needed to upgrade their direct message feature to do the following:

* Recognize the invite event
* Construct the invite event
* Parse the invite event
* Update the replaceable event

This library can also simply be used as a reference by the client to aid them in the implementation.


## Incognito Identity Derivation and State Management

To manage the creation of disposable entities, hierarchical deterministic derivation is used, together with a counter which is stored in a replaceable event, along with 128bits of entropy that's used as a seed for the derivation process.

Two indices are used for the derivation schema, conversation index and identity index. The conversation index starts at 0, and is incremented by 1 whenever an identity for a new conversation is required. If additional identities are required for a conversation, the identity counter is incremented. The identity index is either 0 or 1. 

The replaceable event may be padded in order to prevent information leakage by keeping it at a constant size.



## Potential Improvements

### Ratcheting
It would be possible to adjust the implementation to generate a new incognito identity for each subsequent message. This poses some complexity and privacy implications as the user exposes information when requesting the replaceable events which are used to maintain the state of known conversations.

Removing the 2 identity limit (on the identity index used for hd derivation) would allow us to generate as many identities per conversation as we like, but makes state management trickier as we have to store an additional counter, or derive keys until we find matching messages, which is flakey, computationally demanding and congests the network.

### Increasing the idenitity set by querying randomly selected messages from the relay
When the client requests messages, 

### Delaying messages for anonymiziation


## Client Reference Implementations
- [ ] TODO: link to hamstr PR after it's cleaned up
    - [ ] TODO: message Martti once this is done


## Notes
* we trust relays... how do we fix that?
    * spread messages across relays
    * consider a "delay protocol client side"
    * query more messages than the client cares about to increase anonymity set


### Anton
- [ ] check if we need to specify what relay we are going to be using

* explain high level flow, and diagram for it <-- bart: nice yep

* ~~define scheme to derive ephermaral keys (BIP39)
    * ~~use pubkey for derivation and build schema on that <-- bart: could be worth considering Yelle's solution for this? the npub of the recipient is the derivation path? but that would require a change to nostr-tools and window.nostr to support it
    * ~~the key derivation process consists of combining the pubkey of the message receiver and the priv key of the sender, and using the result to 

* ~~pub key + your priv key as entropy
    * We use randomly generated tokens stored in a replacable event encrypted to ourselves
* define data structure that's passed in the encrypted message
* explain how the client abstracts on this and keeps private messages looking the same
    * consideration: include data on what relays the person publishes too
* the spec can make it optional whether you generate a new identity per message 
    * include random nonce in the public part of the message which is used to derive the key deterministically 
* responder has to take the "second" identity that was supplied, and create a new throwaway identity which then messages the "second" identity that was supplied form the initial message
* omit new identity per message as it adds too much complexity and if we use a setup where Bob responds with a similar message to the one Alice sends, we can establish a direct communicaiton between the Bob' and Alice' keys, which are completely unrelated to their well known identities

* we only need two nonces, 0 and 1, 0 for Alice' and 1 for Alice''/files


---

## Related NIPs
* [NIP-59](https://github.com/nostr-protocol/nips/pull/351)
* [private dm events #17](https://github.com/nostr-protocol/nips/pull/17)

https://github.com/nostr-protocol/nips/pull/224/files#diff-503ae34c31b21445fb1624ece3f7924b5a0b7ab1ab31b19ea065fb7975b3f83bR85
https://github.com/lightning/bolts/blob/master/11-payment-encoding.md