# Pigeon — Project Context

## One-Liner

iOS BLE mesh messenger: encrypted text between iPhones over Bluetooth, no internet, no server.

## How It Works

Each iPhone runs as both a BLE Central (scanner) and Peripheral (advertiser). When two devices come into range, they exchange identities (Curve25519 public keys) and can send encrypted messages. If a recipient isn't directly reachable, messages hop through intermediate devices that relay without seeing plaintext. Delivery ACKs flow back through the mesh.

## Encryption

- **Key Agreement**: Curve25519 ECDH → HKDF (SHA256, salt: "Pigeon-HKDF-Salt-v1") → 256-bit symmetric key
- **Message Encryption**: AES-256-GCM with random 12-byte nonce per message
- **Identity**: No accounts. Keypair IS the identity. PigeonID = first 8 hex chars of SHA256(publicKey)

## BLE Transport

- Service UUID: `E1A71B10-9F11-4F1F-93F8-1A0D3A2B0001`
- 3 characteristics: identity (read), message (write+notify), ACK (write+notify)
- Messages chunked into 480-byte payloads with 22-byte binary headers
- Reassembly timeout: 30s. Forwarding retention: 1hr. Default TTL: 5 hops.

## Tech Stack

Swift 5.9+ / SwiftUI / iOS 26.0+ / CoreBluetooth / CryptoKit / SwiftData / MVVM / @Observable / Zero external deps

## Strict Concurrency

Project uses `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`. All types are MainActor by default. DTOs are explicitly `nonisolated Sendable`. MeshRouter is an `actor`. BLEManager is NSObject on a DispatchQueue. BLEManagerDelegate methods are `nonisolated` and hop to MainActor.

## Architecture Decisions

| Decision | Rationale |
|----------|-----------|
| Separate DTO structs from @Model classes | DTOs are Codable for networking/crypto. @Model can't serve both roles cleanly |
| AppCoordinator as single @Observable | Central state management, single source of truth for UI |
| BLEManager as NSObject (not actor) | CoreBluetooth requires NSObject for delegate conformance and specific queue dispatch |
| MeshRouter as actor | Thread-safe state management for routing queues without manual locking |
| No external dependencies | Reduces attack surface, simplifies builds, everything ships with iOS |

## File Map

32 Swift files across 7 directories:

- **Models/** (4): DTO value types — `PigeonIdentity`, `Message`, `Peer`, `Conversation`
- **Networking/** (7): BLE stack — `BLEManager` (+Central, +Peripheral), `MessageProtocol`, `MeshRouter`, `ReassemblyBuffer`, `BLEConstants`
- **Crypto/** (2): `CryptoManager` (encrypt/decrypt), `KeyStore` (Keychain)
- **Persistence/** (4): `PigeonStore` (facade), `StoredMessage`, `StoredConversation`, `StoredPeer`
- **Services/** (2): `AppCoordinator` (hub), `NotificationManager`
- **Views/** (11): `MainTabView`, `ConversationList/Row`, `Chat/Bubble`, `PeerDiscovery/Row`, `Identity/QR`, `Settings`, `PigeonTheme`
- **Utilities/** (1): `Extensions` (Data hex, UUID bytes)

## Success Demo

Three iPhones, airplane mode, Bluetooth on:
1. Phone A sends "hello" to Phone C
2. A is NOT in BLE range of C
3. Phone B relays: A → B → C
4. C receives decrypted "hello" (B never sees plaintext)
5. C replies: C → B → A
