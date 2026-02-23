# Pigeon — AI Agent Context

Pigeon is a BLE mesh messaging app for iOS. Two or more iPhones exchange end-to-end encrypted text messages over Bluetooth Low Energy with no internet, no cell service, no server. Messages hop device-to-device through nearby Pigeon nodes.

## Build & Run

```bash
cd Pigeon
xcodebuild -scheme Pigeon -destination 'platform=iOS Simulator,name=iPhone 17 Pro' build
```

BLE cannot be tested in the simulator. All BLE testing requires physical devices with Bluetooth enabled. The simulator build verifies compilation only.

Deployment target: iOS 26.0. Uses Swift 5.9+ with `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` (strict concurrency). All value types crossing isolation boundaries must be `nonisolated` and `Sendable`.

## Architecture

MVVM with `@Observable` (Observation framework). No external dependencies — only Apple frameworks: CoreBluetooth, CryptoKit, SwiftData, SwiftUI.

### Object Graph

```
PigeonApp
  └── AppCoordinator (@Observable, @MainActor)
        ├── PigeonIdentity (Curve25519 keypair)
        ├── PigeonStore (@MainActor, owns ModelContainer)
        ├── CryptoManager (stateless encrypt/decrypt)
        ├── MeshRouter (actor, outbound queue + forwarding store)
        └── BLEManager (NSObject, dual Central+Peripheral)
              ├── CBCentralManager (on bleQueue)
              └── CBPeripheralManager (on bleQueue)
```

`AppCoordinator` is injected into the SwiftUI environment via `.environment(coordinator)`. Views access it with `@Environment(AppCoordinator.self)`.

### Data Flow: Sending a Message

1. `ChatView` calls `coordinator.sendMessage(text:in:)`
2. `AppCoordinator` creates `Message` DTO → saves to SwiftData → encrypts with `CryptoManager` → enqueues in `MeshRouter` → calls `BLEManager.sendMessage()`
3. `BLEManager` chunks the envelope via `MessageProtocol` → writes chunks to the peer's `messageCharacteristic` over BLE
4. Peer receives chunks → `ReassemblyBuffer` collects them → `MessageProtocol.reassemblePayload()` → `handleReassembledData()` → decrypts → calls delegate
5. Recipient sends ACK → sender updates status to `.delivered`

### Data Flow: Mesh Relay

When a message arrives that is NOT addressed to this device:
1. `MeshRouter.markSeen()` deduplicates
2. `MessageProtocol.incrementHopCountIfAllowed()` checks TTL
3. `MeshRouter.storeForForwarding()` holds for 1 hour
4. Forward to all connected peers except the sender

### Two-Layer Model Architecture

Plain Codable structs (`Message`, `Peer`, `Conversation`, `MessageEnvelope`) serve as DTOs for networking, crypto, and routing. Separate SwiftData `@Model` classes (`StoredMessage`, `StoredConversation`, `StoredPeer`) handle persistence. `PigeonStore` maps between them. Never add `@Model` to the DTO structs.

## Key Files

| File | What it does |
|------|-------------|
| `PigeonApp.swift` | Entry point. Creates PigeonStore + AppCoordinator, injects into environment |
| `Services/AppCoordinator.swift` | Owns all services. Implements BLEManagerDelegate. Drives UI state |
| `Networking/BLEManager.swift` | Core BLE orchestrator. Central + Peripheral simultaneously on `bleQueue` |
| `Networking/BLEManager+Central.swift` | CBCentralManagerDelegate + CBPeripheralDelegate. Scanning, connection, reads |
| `Networking/BLEManager+Peripheral.swift` | CBPeripheralManagerDelegate. Advertising, identity reads, message writes |
| `Networking/MessageProtocol.swift` | Envelope/ACK/Packet encoding. Chunking (480-byte payloads). Reassembly |
| `Networking/MeshRouter.swift` | Actor. Outbound queue keyed by recipient pubkey. Forwarding store with TTL. Dedup cache |
| `Networking/ReassemblyBuffer.swift` | Collects chunks by messageID, tracks completion and expiry |
| `Networking/BLEConstants.swift` | Service UUID, characteristic UUIDs, protocol constants |
| `Crypto/CryptoManager.swift` | Curve25519 key agreement → HKDF → AES-256-GCM encrypt/decrypt |
| `Crypto/KeyStore.swift` | Keychain storage for private key and known peer public keys |
| `Models/PigeonIdentity.swift` | Keypair generation, pigeonID (first 8 hex of SHA256), display name |
| `Models/Message.swift` | DTO: id, conversationID, sender/recipient pubkeys, plaintext, status, isIncoming |
| `Models/Peer.swift` | DTO: publicKey, pigeonID, displayName, rssi, firstSeen, lastSeen, isSaved |
| `Models/Conversation.swift` | DTO: id, peerPublicKey, peerDisplayName, peerPigeonID, unreadCount |
| `Persistence/PigeonStore.swift` | SwiftData CRUD. All ops @MainActor. Accepts/returns DTOs |
| `Persistence/StoredMessage.swift` | @Model for messages. Maps to/from Message DTO |
| `Persistence/StoredConversation.swift` | @Model for conversations. Maps to/from Conversation DTO |
| `Persistence/StoredPeer.swift` | @Model for peers. Maps to/from Peer DTO |
| `Views/PigeonTheme.swift` | Dark theme colors and typography constants |

## BLE Protocol

Service UUID: `E1A71B10-9F11-4F1F-93F8-1A0D3A2B0001`

| Characteristic | UUID suffix | Properties | Purpose |
|---------------|------------|------------|---------|
| Identity | `...0003` | Read | Returns JSON `PeerIdentityPayload` (publicKey + pigeonID + displayName) |
| Message | `...0002` | Write, WriteWithoutResponse, Notify | Sends/receives chunked `MessageEnvelope` data |
| ACK | `...0004` | Write, WriteWithoutResponse, Notify | Delivery acknowledgments |

Messages > 480 bytes are chunked with a 22-byte binary header: `[UUID messageID (16)] [UInt16 chunkIndex] [UInt16 totalChunks] [UInt16 payloadSize]`. Big-endian.

## Concurrency Model

- `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` — everything is MainActor by default
- DTOs and utilities are explicitly `nonisolated` + `Sendable`
- `BLEManager` is `NSObject` (not actor) — CoreBluetooth delegates run on `bleQueue` (DispatchQueue)
- `MeshRouter` is a Swift `actor`
- `PigeonStore` is `@MainActor` — all SwiftData ops on main context
- `AppCoordinator` is `@MainActor` + `@Observable`
- BLEManagerDelegate methods are `nonisolated` and hop to `@MainActor` via `Task { @MainActor in ... }`

## Design System

Dark theme. Forced `.preferredColorScheme(.dark)`.

| Token | Value |
|-------|-------|
| Background | `#0A0A0A` |
| Surface | `#1A1A1A` |
| Surface Secondary | `#2A2A2A` |
| Accent (violet) | `#8B5CF6` |
| Accent Dark | `#7C3AED` |
| Text Primary | `#F5F5F5` |
| Text Secondary | `#8A8A8A` |
| Success | `#4CAF50` |
| Error | `#EF5350` |
| Outgoing bubble | `#7C3AED` |
| Incoming bubble | `#2A2A2A` |

All colors and fonts go through `PigeonTheme`. Never hardcode colors in views.

## Out of Scope (Do NOT Build)

- Group chats
- Voice/media (text only)
- Read receipts
- Cloud backup / server
- LoRa / hardware dongles
- Web or Android
- App Store submission (TestFlight only)
