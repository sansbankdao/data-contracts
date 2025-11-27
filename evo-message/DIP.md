```markdown
  DIP: 00XX (To be assigned)
  Title: EvoMessage: End-to-End Encrypted Messaging via Data Contracts
  Author(s): Shomari
  Status: Draft
  Type: Standard
  Layer: Applications
  Created: 2025-11-27
  License: MIT License
  Requires: DIP-0015
```

# Table of Contents
1. [Abstract](#abstract)
2. [Motivation](#motivation)
3. [Prior Work](#prior-work)
4. [Terminology](#terminology)
5. [Specification](#specification)
    - [Dependencies & The Gatekeeper Model](#dependencies--the-gatekeeper-model)
    - [The EvoMessage Data Contract](#the-evomessage-data-contract)
    - [Binary Data Encoding](#binary-data-encoding)
    - [Multi-Device Routing](#multi-device-routing)
    - [Encryption Protocol (Double Ratchet)](#encryption-protocol-double-ratchet)
    - [Message Chaining](#message-chaining)
6. [Rationale](#rationale)
7. [Backwards Compatibility](#backwards-compatibility)
8. [Copyright](#copyright)

# Abstract
This proposal introduces **EvoMessage**, a standard for End-to-End Encrypted (E2EE) messaging on Dash Platform. It implements the Signal Protocol (Double Ratchet + X3DH) to provide Forward Secrecy, Post-Compromise Security, and Multi-Device support. The system utilizes DashPay (DIP-0015) relationships as a "Gatekeeper" for contact discovery, while EvoMessage handles the secure transport of data via specific Data Contract documents (`preKeyBundle`,`message`,`messagePart`).

# Motivation
While DashPay (DIP-0015) introduced the concept of Contacts, it relies on a static shared secret for encryption, which lacks Forward Secrecy. Furthermore, users require a censorship-resistant method to exchange rich data (text, images, invoices) asynchronously across multiple devices (e.g., Phone and Laptop).

By utilizing Dash Platform, EvoMessage guarantees:
1.  **Censorship Resistance:** Messages are stored on the decentralized Platform state.
2.  **Forward Secrecy:** Compromise of a current key does not reveal past messages.
3.  **Multi-Device Sync:** Users can seamlessly switch between devices while maintaining a unified conversation history.

# Prior Work
*   **DIP-0015 (DashPay):** Defines Identity creation and the`ContactRequest` mechanism used here as the prerequisite "Handshake".
*   **Signal Protocol:** The cryptographic primitives (X3DH, Double Ratchet) used in this proposal.

# Terminology
*   **Session:** A cryptographic state between two specific devices.
*   **Gatekeeper:** The logic whereby strangers must become DIP-0015 Contacts before exchanging EvoMessages.
*   **Sender Scanning:** The privacy-preserving method of retrieving messages by querying the Sender's ID rather than the Recipient's ID.
*   **Device ID:** A unique integer (1, 2, 3...) identifying a specific client instance of an Identity.

# Specification

The EvoMessage system relies on the Dash Platform for storage.

## Dependencies & The Gatekeeper Model
To prevent spam and preserve privacy, EvoMessage does not support direct "Stranger" discovery.
1.  **Discovery:** A stranger must send a **DIP-0015 Contact Request** via the DashPay Data Contract.
2.  **Handshake:** The recipient accepts the request, adding the stranger to their local Contact List.
3.  **Communication:** Once established as a Contact, the pair may begin exchanging **EvoMessages**. Clients **MUST** only scan for EvoMessages from Identities present in their local DashPay Contact List.

## The EvoMessage Data Contract
The contract defines three document types.

### 1. The PreKey Bundle (`preKeyBundle`)
Stores keys required for X3DH (Asynchronous setup).
*   **Indices:** Unique index on`[$ownerId, deviceId]`.
*   **Properties:**
    *`registrationId` (integer): Signal-specific installation ID.
    *`deviceId` (integer): Unique ID (1, 2, 3...).
    *`identityKey` (byteArray): Long-term public key.
    *`signedPreKey` (byteArray): Signed pre-key data.
    *`signedPreKeySignature` (byteArray): Signature of the pre-key.
    *`oneTimePreKeys` (array of objects): Max 100 ephemeral keys.

### 2. The Message (`message`)
A discrete encrypted payload targeted at a *specific* device.
*   **Indices:**`[$ownerId, recipientDeviceId, $createdAt]` (Compound index for Sender Scanning).
*   **Properties:**
    *`recipientId` (byteArray): Hashed/Encrypted ID of recipient (for local verification).
    *`recipientDeviceId` (integer): The specific device this payload is encrypted for.
    *`senderDeviceId` (integer): The device ID of the sender.
    *`messageType` (integer): 1 = Standard, 3 = PreKey (Session Init).
    *`ratchetPubKey` (byteArray): The sender's current ratchet public key.
    *`counter` (integer): Symmetric ratchet index.
    *`previousCounter` (integer): For out-of-order handling.
    *`encryptedPayload` (byteArray): AES-GCM ciphertext (Max ~14KB).
    *`isMultipart` (boolean): True if content continues in`messagePart`.
    *`replyToId` (byteArray): Optional parent ID.

### 3. The Message Part (`messagePart`)
Used for payloads exceeding the document limit.
*   **Indices:** Unique index on`[parentMessageId, sequenceIndex]`.
*   **Properties:**
    *`parentMessageId` (byteArray): Reference to the root`message`.
    *`sequenceIndex` (integer): 1, 2, 3...
    *`encryptedChunk` (byteArray): Partial ciphertext.

## Binary Data Encoding
Dash Platform uses JSON Schema. To ensure strict type safety and storage efficiency, all fields marked as`byteArray` above **MUST** be encoded as an **Array of Integers** (0-255).
*   *Example:*`[255, 0, 128, 64...]`
*   String encoding (Base64/Hex) is **NOT** permitted for cryptographic fields.

## Multi-Device Routing
EvoMessage treats every Device as a separate cryptographic endpoint (Option B).
*   **Sending:** If Alice wants to message Bob (who has Device 1 and Device 2), Alice's client MUST encrypt the payload twice (once for Session A->B1, once for Session A->B2).
*   **Submission:** Alice submits a Batch Transition containing **two** distinct`message` documents.
    *   Doc 1:`recipientDeviceId: 1`
    *   Doc 2:`recipientDeviceId: 2`
*   **Receiving:** Bob's Device 1 queries Platform`WHERE ownerId == Alice AND recipientDeviceId == 1`. It ignores Doc 2, saving bandwidth.

## Encryption Protocol (Double Ratchet)
1.  **Initialization:** Clients perform an X3DH exchange using the`preKeyBundle` to derive a Master Secret.
2.  **Ratcheting:**
    *   **Diffie-Hellman Ratchet:** Updates the Root Key upon every reply (exchange of`ratchetPubKey`).
    *   **Symmetric Ratchet:** Updates the Message Key for every message sent using the`counter`.
3.  **Payload:** Content is encrypted using`AES-256-GCM`.

## Message Chaining
For payloads >14KB:
1.  Split plaintext into chunks.
2.  Chunk 0 goes into`message.encryptedPayload`.
3.  Set`message.isMultipart = true`.
4.  Remaining chunks go into`messagePart` documents.
5.  Recipient assembles and decrypts as a single stream.

# Rationale

### Why the "Gatekeeper" Model?
Allowing strangers to send encrypted messages directly via EvoMessage would require a public index of Recipients (`toUserId`). This would leak the social graph (metadata) to the entire network. By relying on DashPay Contact Requests as the "Gatekeeper," we utilize an existing anti-spam mechanism where the Recipient ID is public *only* for the initial handshake. Once contact is established, EvoMessage takes over using "Sender Scanning" (Querying by`$ownerId`), ensuring metadata privacy for the actual conversation.

### Why Multi-Device (Option B)?
Modern users expect seamless transition between mobile and desktop. Treating Identity as a single device forces a "log out/wipe keys" workflow when switching devices, which destroys message history. By implementing distinct`deviceId` routing, we allow independent cryptographic sessions, ensuring robust history synchronization across all user devices.

### Why Sender Pays?
Storing data on a decentralized blockchain incurs permanent costs. The "Sender Pays" model via Dash Credits acts as a natural economic spam deterrent. There is no centralized "free tier" to abuse.

# Backwards Compatibility
This DIP utilizes the Identity system defined in DIP-0011 and DIP-0015 but operates as an opt-in Application Layer standard. It does not modify the Consensus Layer.

# Copyright
Copyright (c) 2025 Sansbank DAO. [Licensed under the MIT License](https://opensource.org/licenses/MIT).
