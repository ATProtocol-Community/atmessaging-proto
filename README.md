# AT Messaging Proto

This repository hosts the **AT Messaging Proto** specification, a draft standard for end-to-end encrypted (E2EE) direct and group messaging in AT Protocol applications using the Messaging Layer Security (MLS) Protocol.

## Purpose

The goal of this repository is to collaboratively develop and refine the AT Messaging Proto specification, enabling open, efficient, and secure E2EE messaging for AT Protocol applications like Bluesky. We welcome community contributions to improve the spec, address technical challenges, and ensure it meets the needs of decentralized, privacy-focused messaging.

## Context

Bluesky's current direct messaging system is centralized. As outlined in the [Bluesky Roadmap](https://blueskyweb.xyz/roadmap), the goal is to support E2EE DMs natively in the AT Protocol without relying on a centralized service. At the March 2025 AtmosphereConf in Seattle, an [E2EE working group](https://wiki.atprotocol.community/en/working-groups/e2ee) was formed to define this standard, leveraging MLS for robust security.

## Goals of AT Messaging Proto

- Private *and* Confidential DMs and Group messages  
  - Private means that an observer cannot tell that Alice and Bob are talking to one another, or that Alice is part of a specific group. This necessarily requires protecting metadata.  
  - Confidential means that the contents of conversations can only be viewed by the intended recipients.  
- Forward secrecy and Post-compromise security  
  - Forward secrecy means that encrypted content in the past remains encrypted even if a key material is leaked.  
  - Post compromise security means that leaking key material doesn’t allow an attacker to continue to read messages indefinitely into the future.  
- Scales efficiently for large groups  
- Users can access (E2EE) messages from multiple application/devices. For example, a user can use either a web browser on a laptop, or an iPhone App to send and receive messages in the same group.  
- Decentralized MLS Delivery and Authentication Service

## Contributing

We encourage contributions to the AT Messaging Proto specification\! To get started:

1. Read the [draft specification](./SPECIFICATION.md).  
2. Check [open issues](https://github.com/ATProtocol-Community/atmessaging-proto/issues) for ongoing discussions or propose new ones.  
3. Submit pull requests with proposed changes, following the contribution guidelines (to be added).  
4. Join the conversation on Discord at [https://discord.gg/deTNUM76](https://discord.gg/deTNUM76)
