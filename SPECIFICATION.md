# AT Messaging Proto

## E2EE Messaging using the Messaging Layer Security (MLS) Protocol for AT Protocol applications

`Draft - in progress`

This document proposes a standard for how to combine the [MLS Protocol](https://www.rfc-editor.org/rfc/rfc9420.html) with AT Protocol for open, efficient, E2EE (end-to-end encrypted) direct and group messaging.

## Context

Bluesky's direct messaging is currently a centralized system. The [Bluesky Roadmap](https://docs.bsky.app/blog/2024-protocol-roadmap) outlines plans to “to fully support E2EE DMs as part of atproto itself, **without a centralized service**”. At the March 2025 AtmosphereConf in Seattle, an E2EE working group was established to help define this standardized messaging protocol based on MLS.

### Goals of this messaging protocol are

1. Private *and* Confidential DMs and Group messages  
   1. Private means that an observer cannot tell that Alice and Bob are talking to one another, or that Alice is part of a specific group. This necessarily requires protecting metadata.  
   2. Confidential means that the contents of conversations can only be viewed by the intended recipients.  
2. Forward secrecy and Post-compromise security  
   1. Forward secrecy means that encrypted content in the past remains encrypted even if a key material is leaked.  
   2. Post compromise security means that leaking key material doesn’t allow an attacker to continue to read messages indefinitely into the future.  
3. Scales efficiently for large groups  
4. Users can access (E2EE) messages from multiple application/devices. For example, a user can use either a web browser on a laptop, or an iPhone to send and receive messages in the same group.  
5. Decentralized Delivery and Authentication Service

### Why MLS?

You can think of MLS as an evolution of the Signal Protocol. However, it significantly improves the scalability of encryption operations for large group messaging significantly (linear \-\> log), is built to accommodate federated environments, and also allows for graceful updating of ciphersuites and versions over time. In addition, it’s very flexible and agnostic about the message content that is sent.

It’s beyond the scope of this document to explain the MLS protocol but you can read more about it in its [Architectural Overview](https://www.ietf.org/archive/id/draft-ietf-mls-architecture-13.html) or the [RFC](https://www.rfc-editor.org/rfc/rfc9420). MLS is on track to become an internet standard under the IETF so the protocol itself is extremely well vetted and researched. This also means there is the potential for cross network messaging interoperability in the future as MLS gains more adoption.

### Why do we need a decentralized MLS Authentication & Delivery Service

The purpose of decentralizing the MLS Authentication and Delivery Service is to prevent any one entity having administrative control over all the E2EE messaging network while also ensuring different clients adhering to the protocol defined in this document can message each other. This is a key objective for E2EE messaging between AT Protocol applications.

User accounts on the AT Protocol utilize [W3C-standard](https://www.w3.org/TR/did-core/) Decentralized Identifiers (DIDs), providing secure, stable, and administratively decentralized identifiers. AT Protocol supports the DID PLC and DID Web variants. This specification extends the use of DIDs to serve as the identifier mechanism for the MLS Authentication Service, with the integration process detailed in subsequent sections. Effectively, the AT Protocol DIDs are decentralized, using them as the root identity in the MLS Authentication Service is straight-forward.

The primary complexity of this proposal lies in defining a decentralized Delivery Service. There are two open source implementations of MLS group chat that utilize a decentralized [MLS Delivery Service](https://www.ietf.org/archive/id/draft-ietf-mls-architecture-15.html#name-delivery-service) rather than a centralized service.

- [XMTP](https://xmtp.org) \- An open source decentralized group chat protocol based on MLS, including client libraries and a functioning decentralized MLS Delivery and Authentication Service.  
- [Whitenoise](https://github.com/parres-hq/whitenoise) \- MLS group chats on the Nostr protocol based on [NIP-EE](https://github.com/nostr-protocol/nips/blob/001c516f7294308143515a494a35213fc45978df/EE.md)

Note: There are a few other projects using MLS for E2EE (eg. Germ, Yup-chat), but they do not support decentralizd groups, so they are not included as references.

Our primary aim is to develop a decentralized, MLS-based E2EE messaging system for AT Protocol applications. Additionally, we seek to foster collaboration by encouraging alignment across these E2EE messaging projects toward a unified, decentralized MLS-based protocol, complete with a standardized Authentication and Delivery Service, supported by shared open-source reference implementations. While some questions remain open, this proposal is intended to rally builders around a collective vision and drive coordinated progress.

In addition to leveraging standardized decentralized MLS Authentication & Delivery Service layers, each client must adhere to the other aspects of this protocol definition such as message content formats to interoperate with each other.

Embracing the IETF's ethos of refining protocols through rough consensus and running code, this proposal draws inspiration from both of these open-source protocols, with this document adapted from the NIP-EE standard, and the proposal based on XMTP. XMTP was chosen as a basis for this E2EE protocol because it is the only purpose-built, open source, decentralized E2EE messaging protocol based on MSL. We express our deep appreciation for the foundational work by [@erskingardner](https://github.com/erskingardner) and work done by the XMTP team.

## Core MLS Concepts

From the [MLS Architectural Overview](https://www.ietf.org/archive/id/draft-ietf-mls-architecture-13.html):

> MLS provides a way for clients to form groups within which they can communicate securely. For example, a set of users might use clients on their phones or laptops to join a group and communicate with each other. A group may be as small as two clients (e.g., for simple person to person messaging) or as large as hundreds of thousands. A client that is part of a group is a member of that group. As groups change membership and group or member properties, they advance from one epoch to another and the cryptographic state of the group evolves.
>
> The group is represented as a tree, which represents the members as the leaves of a tree. It is used to efficiently encrypt subsets of the members. Each member has a state called a LeafNode object holding the client’s identity, credentials, and capabilities.

The MLS protocol’s job is to manage and evolve the cryptographic state of a group. This includes managing the membership of a group, the cryptographic state of a group (ratchet tree, keys, and encryption/decryption/authentication of messages), and managing the evolution of the group over time.

### Groups

Groups are created by their first member, who invites one or more other members. Groups evolve over time in blocks called `Epochs`. New epochs are proposed via one or more `Proposal` messages and then committed to via a `Commit` message.

### Clients

The application/device pair (e.g. Bluesky on iOS or AcmeChat on web) with which a user joins the group is represented as a `LeafNode` in the tree. The term `Client` is used to refer to the application/device pair and hold cryptographic material, such as private keys, KeyPackages, and credentials, to perform actions like creating groups, sending messages, or joining groups. The term `Member` is a `Client` that is actively part of a specific MLS group and has access to the group’s shared cryptographic state (e.g., the group key derived from the ratchet tree).

It is not possible to share cryptographic group state across multiple `Members`. If a user joins a group from 2 separate `Clients`, their cryptographic state is separate and they will be tracked as 2 separate `Members` of the group. In most cases, if a user joins a group and has two separate devices, the UX would show them as a single user and not expose that there are two different `LeafNode` for that user.

### Direct Message (DM)

Direct messages in MLS are just a special case of a group where there are only two users in the group. A user may have multiple applications/devices and thus multiple `Members` in the group with two users.

### Messages

There are several different types of messages sent within a group. Some of these are control messages that are used to update the group state over time. These include `Welcome`, `Proposal`, and `Commit` messages. Others are the actual messages that are sent between members in a group. These include `Application` messages.

Messages in MLS are “framed”. Meaning that they are wrapped in a data structure that includes information about the sender, the epoch, the message index within the epoch and the message content. This framing makes it possible to authenticate and decrypt messages correctly, even if they arrive out of order.

MLS is agnostic to the “content” of the messages that are sent. This is a key feature of MLS that allows for the use of MLS for a wide variety of applications.

MLS is also agnostic to the transport protocol that is used to send messages. One goal of this document is to specify the transport protocol (ie. the MLS Delivery Service) to facilitate interoperability between AT Protocol Messaging.

## The focus of this Proposal

This proposal focuses on how applications built on AT Protocol can share common Authentication and Delivery Service functions required by the MLS protocol as well as specifying the message content format. Most clients will choose to use an MLS implementation to handle keys, ratcheting, group state management, and other aspects of the MLS protocol itself. [OpenMLS](https://github.com/openmls/openmls) is the most actively developed library that implements MLS.

This proposal specifies the following:

#### Authentication

1. The required format of the MLS [`Credential`](#mls-credentials) that AT Protocool clients should use to represent ATproto users in a group.

#### Delivery Service

We propose a new MLS Delivery Service for AT Protocol Messaging that is purpose-built for E2EE messaging based on the open source XMTP service.

2. A standardized way that AT Protocool clients should [create MLS groups](#creating-groups).  
3. The structure of [KeyPackage](#keypackage-event-and-signing-keys) that allows AT Protocool users to be added to a group asynchronously.  
4. The manner in which [KeyPackage](#keypackage-event-and-signing-keys) are published to the Delivery Service.  
5. The structure of [Group Messages](#group-events) that represent the evolution of a group's state and the contents of the messages sent in the group.  
6. The manner in which [Group Messages](#group-events) are published to the Delivery Service

#### Message Content

7. A standard way to encode the contents of a message with an identifying content type to support future message types.

## Security Considerations

This is a concise overview of the security trade-offs and considerations of this proposal in various scenarios. This proposal strives to fully maintain MLS security guarantees.

### Forward Secrecy and Post-compromise Security

* As per the MLS spec, keys are deleted as soon as they are used to encrypt or decrypt a message. This is usually handled by the MLS implementation library itself but attention needs to be paid by clients to ensure they’re not storing secrets (especially the [exporter secret](#group-events)) for longer than absolutely necessary.  
* This proposal maintains MLS forward secrecy and post-compromise security guarantees. You can read more about those in the MLS Architectural Overview section on [Forward Secrecy and Post-compromise Security](https://www.ietf.org/archive/id/draft-ietf-mls-architecture-15.html#name-forward-and-post-compromise).

### Leakage of various keys

* This proposal does not depend on a user’s AT Protocol DID identity key for any aspect of the MLS messaging protocol other than its use in the Credential to authenticate the user identity. Compromise of a user’s DID identity key does not give access to past or future messages in any MLS-based group.  
* For a complete discussion of MLS key leakage, please see the Endpoint Compromise section of the [MLS Architectural Overview](https://www.ietf.org/archive/id/draft-ietf-mls-architecture-15.html#name-endpoint-compromise).

### Metadata

* For Welcome Messages only the identity of the invitees is published to the Delivery Service. Welcome messages encrypt the sender metadata and group ID, protecting the social graph.  
* For Welcome Messages a layer of encryption using HPKE (Hybrid Public Key Encryption) is used. This prevents multiple recipients of the same Welcome message from being correlated to the same group.  
* For all other Messages the only group specific metadata published to the Delivery Service is the Message group Mailbox ID. This Mailbox ID is the only value an outside observer can identify on the message. This is a tradeoff to ensure that group participants and group size are obfuscated but still makes it possible to efficiently fan out group messages to all participants. MLS [PrivateMessage](https://www.rfc-editor.org/rfc/rfc9420.html#name-confidentiality-of-sender-d) framing is used to hide the sender and content of group messages. Only group members with up-to-date group state can decrypt and read these messages.  
* A user’s key packages can be used one or more times to be added to groups. There is a tradeoff inherent here: Reusing key packages (initial signing keys) carries some degree of risk but this risk is mitigated as long as a user rotates their signing key immediately upon joining a group. This step also improves the forward secrecy of the entire group.

### Device Compromise

Clients implementing this proposal should take every precaution to ensure that data is stored in a secure way on the device and is protected against unwanted access in the case that a device is compromised (e.g. encryption at rest, biometric authentication, etc.). That said, full device compromise should be viewed as a catastrophic event and any group the compromised device was a part of should be considered compromised until they can remove that member and update their group’s state. Some suggestions:

* Clients should support and encourage self-destructing messages (ensuring that full transcript history isn’t available on a device forever).  
* Clients should regularly suggest to group admins that inactive users be removed.  
* Clients should regularly suggest (or automatically) rotate a user’s signing key in each of their groups.  
* Clients should encrypt group state and keys on the device using a secret value that isn’t part of the group state or the user’s AT Protocol DID key.  
* Clients should use secure enclave storage where possible.

For a full discussion of the security considerations of MLS, please see the Security Considerations section of the [MLS RFC](https://www.rfc-editor.org/rfc/rfc9420.html#name-security-considerations).

##

## Delivery Service Network

We propose the concept of a Delivery Service Network. Within the AT Protocol we call this the AT Protocol Delivery Service. Clients that adhere to this specification can connect to any Delivery Service Network that also adheres to this specification.

Running a Delivery Service Network will necessarily require significant resources. In order to preserve the privacy of the participants, this Delivery Service Network cannot require authentication that identifies users in order to send messages on the network. This makes

The following steps would be performed to connect to a Delivery Service Network

1. Application on a device creates a new Mailbox ID via a Delivery Service network of their choosing. Authentication for this step would depend on the Delivery Service.  
   1. \[Add a note about creating ephemeral Mailboxes for off-the-record / pseudonymous chats here.\]  
2. Application registers this Mailbox ID and the Delivery Service Network in the user’s AT Protocol PDS.

## MLS Credentials

The identity of AT Messaging Proto users within the Delivery Service Network is the Mailbox ID. This Mailbox ID serves as a delegate for the user’s AT Protocol DID. Each `Client` has its own Application ID / Installation ID public key which acts as a delegate of the user’s Mailbox ID for the sole purpose to authenticate the user within this E2EE messaging protocol.

A `Credential` in MLS is an assertion of who the user is coupled with a signing key. When constructing `Credentials` for MLS, clients MUST use the `BasicCredential` type and set the `identity` value as the `Mailbox ID` (format and encoding need to be added). Clients MUST not allow users to change the identity field and MUST validate that all `Proposal` messages do not attempt to change the identity field on any credential in the group. Clients MUST ensure that the Mailbox ID is still a valid delegate E2EE public key for the user’s DID.

A `Credential` also has an associated signing key. The initial signing key for a user is included in the KeyPackage. The signing key MUST be different from the user’s Application ID’s public key and AT Protocol DID public key. This signing key SHOULD be rotated over time to provide improved post-compromise security.

Create a new Mailox ID for an AT Protocol DID

`{`  
   `API/Format - TBD`

   `XMTP: PUBLISH_IDENTITY_UPDATE = "/identity/v1/publish-identity-update"`

`}`

Add Application ID to Mailox ID  
`{`  
   `API/Format - TBD`

   `XMTP: REGISTER_INSTALLATION = "/mls/v1/register-installation"`

`}`

## KeyPackage Signing Keys

Each user that wishes to be reachable via MLS-based messaging MUST first publish at least one KeyPackage. The KeyPackage is used to authenticate users and create the necessary `Credential` to add members to groups in an asynchronous way. Users can publish multiple KeyPackages with different parameters (supporting different ciphersuites or MLS extensions, for example). KeyPackages include a signing key that is used for signing MLS messages within a group. This signing key MUST not be the same as the user’s DID verificationMethod signing key or the `app_id_pubkey`.

KeyPackage reuse SHOULD be minimized. However, in normal MLS use, KeyPackages are consumed when joining a group. In order to reduce race conditions between invites for multiple groups using the same KeyPackage, clients SHOULD use “Last resort” KeyPackages. This requires the inclusion of the `last_resort` extension on the KeyPackage’s capabilities (same as with the Group).

It’s important that `Clients` immediately rotate a user’s signing key after joining a group via a last resort key package to improve post-compromise security. The signing key (the public key included in the KeyPackage) is used for signing within the group. Therefore, clients implementing this proposal MUST ensure that they retain access to the private key material of the signing key for each group they are a member of.

In most cases, it’s assumed that `Clients` implementing this proposal will manage the creation and rotation of KeyPackages.

KeyPackages are published via the Delivery Service. This is a key function of the Deliver Service (no pun intended). This proposal outlines how the decentralized Delivery Service supports the management of client KeyPackages.  

`{`

   `API/Format - TBD`

   `XMTP: UPLOAD_KEY_PACKAGE = "/mls/v1/upload-key-package"`

`}`

### Deleting KeyPackage Events

Clients SHOULD delete the KeyPackage on all the listed Delivery Service networks any time they successfully process a group request event for a given KeyPackages. Clients MAY also create a new KeyPackages at the same time.

If clients cannot process a Welcome message (e.g. because the signing key was generated on another client), clients MUST not delete the KeyPackage and SHOULD show a human-understandable error to the user.

`{`

   `API/Format - TBB`

`}`

### Rotating Signing Keys

Clients MUST regularly rotate the user’s signing key in each group that they are a part of. The more often the signing key is rotated the stronger the post-compromise security. This rotation is done via `Proposal` and `Commit` events and broadcast to the group via a Group Message. [Read more about forward secrecy and post-compromise security inherent in MLS](https://www.rfc-editor.org/rfc/rfc9420.html#name-forward-secrecy-and-post-co).

### Fetch KeyPackages

The Delivery Service MUST provide a way to fetch all KeyPackages for a given did.

`{`

   `API/Format - TBD`

   `XMTP: FETCH_KEY_PACKAGES = "/mls/v1/fetch-key-packages"`

`}`

## Creating groups

MLS Groups are created with a random 32-byte ID value that is effectively permanent. This ID should be treated as private to the group and MUST not be published outside the members of the group in any form.

Clients must also ensure that the ciphersuite, capabilities, and extensions they use when creating the group are compatible with those advertised by the users they’d like to invite to the group. They can check this info via the user’s published KeyPackage Events.

When creating a new group, the following MLS extensions MUST be used.

* [`required_capabilities`](https://docs.rs/openmls/latest/openmls/extensions/struct.RequiredCapabilitiesExtension.html)  
* [`ratchet_tree`](https://docs.rs/openmls/latest/openmls/extensions/struct.RatchetTreeExtension.html)  
* `atproto_group_data (TBD)`

And the following MLS extension is highly recommended (more [here](https://stackedit.io/app#keypackage-event-and-signing-keys)):

* [`last_resort`](https://docs.rs/openmls/latest/openmls/extensions/struct.LastResortExtension.html)

Changes to an MLS group are affected by first creating one or more Proposals Messages and then committing to a set of proposals in a Commit message. However, for the group state to properly evolve the Commit messages (which represent a specific set of proposals \- like adding a new user to the group) must be published to Delivery Service for the other group members to see. See [Group Messages](#group-events) for more information.

### **`Commit` Message race conditions**

The MLS protocol is resilient to almost all messages arriving out of order. However, the order of `Commit` messages is important for the group state to move forward from one epoch to the next correctly. With a decentralized Delivery Service, it is possible for a client to receive 2 or more `Commit` messages all attempting to update to a new epoch at the same time.

Delivery Services Networks conforming to this specification must provide an ordering mechanism to ensure proper group state. For example, the XMTP network achieves this by sequencing all of the group management messages through a blockchain.

## Welcome Message

When a new user is added to a group via an MLS `Commit` message. The member who sends the `Commit` message to the group is responsible for sending the user being added to the group a Welcome Message. This Welcome Message is sent to the user as via the Delivery Service to the user’s Mailbox ID. The Welcome Message gives the new member the context they need to join the group and start sending messages.

Clients creating the Welcome Message SHOULD wait until they have received acknowledgement from relays that their Group Message with the `Commit` has been received before publishing the Welcome Message.

`{`

   `API/Format - TBD`

   `XMTP: SEND_WELCOME_MESSAGES = "/mls/v1/send-welcome-messages"`

`}`

### Large Groups

For groups with a large number of member participants, welcome messages will become larger than the maximum size allowed by the Delivery Service. There is currently work underway on the MLS protocol to support “light” client welcomes that don’t require the full Ratchet Tree state to be sent to the new member. This section will be updated with recommendations for how to handle large groups.

## Sending Group Messages

Group Messages are all the messages that are sent within a group. This includes all “control” messages that update the shared group state over time (`Proposal`, `Commit`) and messages sent between members of the group (`Application` messages).

`{`

   `API/Format - TBD`

   `XMTP: SEND_GROUP_MESSAGES = "/mls/v1/send-group-messages"`

`}`

## Receiving Messages

In order to preserve the privacy of clients, any client can subscribe and receive both group messages and welcome messages for a specific Mailbox ID. The messages themselves can only be decrypted by clients with the proper keys.

Subscribing to all messages

`{`

   `API/Format - TBD`

   `XMTP: SUBSCRIBE_GROUP_MESSAGES = "/mls/v1/subscribe-group-messages"`

   `XMTP: SUBSCRIBE_WELCOME_MESSAGES = "/mls/v1/subscribe-welcome-messages"`

`}`

Querying messages

`{`

   `API/Format - TBD`

   `XMTP: QUERY_GROUP_MESSAGES = "/mls/v1/query-group-messages"`

   `XMTP: QUERY_WELCOME_MESSAGES = "/mls/v1/query-welcome-messages"`

`}`

## Application Messages Content

Application messages are the messages that are sent within the group by members. These are contained within the MLSMessage object. The message body uses a format that supports different content types. The ContentTypeId identifies the type and format of the information contained in the content. It contains enough information to perform the decoding process. More information on this encoding can be found at [https://github.com/xmtp/XIPs/blob/main/XIPs/xip-5-message-content-types.md](https://github.com/xmtp/XIPs/blob/main/XIPs/xip-5-message-content-types.md)

Note: There is also an [IETF draft](https://datatracker.ietf.org/doc/draft-ietf-mimi-content/) defining message media formats. Support for this format could be considered via a specific ContentTypeId.
