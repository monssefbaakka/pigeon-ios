# CODEX.md — Pigeon

## Project Summary

Pigeon is an iOS app that sends encrypted text messages between iPhones over Bluetooth Low Energy (BLE) with no internet connection. Each device acts as both a sender and a relay node, forming a mesh network. Messages are end-to-end encrypted using Curve25519 key agreement and AES-256-GCM.

## Build Command

```bash
cd Pigeon
xcodebuild -scheme Pigeon -destination 'platform=iOS Simulator,name=iPhone 17 Pro' build
```

Note: BLE features only work on physical iOS devices. Simulator builds verify compilation only.

## Tech Stack

- Language: Swift 5.9+
- UI: SwiftUI with `@Observable` (Observation framework)
- Persistence: SwiftData
- Networking: CoreBluetooth (BLE)
- Crypto: CryptoKit (Curve25519 + AES-GCM)
- Target: iOS 26.0+
- Dependencies: None (Apple frameworks only)
- Concurrency: `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` (strict)

## Source Tree

All source code is under `Pigeon/Pigeon/`:

```
PigeonApp.swift                  — App entry, creates AppCoordinator + SwiftData container

Models/
  PigeonIdentity.swift           — Curve25519 keypair, pigeonID (8 hex chars of SHA256)
  Message.swift                  — Message DTO (id, plaintext, status, isIncoming)
  Peer.swift                     — Peer DTO (publicKey, pigeonID, rssi, isSaved)
  Conversation.swift             — Conversation DTO (peerPublicKey, unreadCount)

Networking/
  BLEConstants.swift             — Service/characteristic UUIDs, protocol constants
  BLEManager.swift               — BLE orchestrator: Central + Peripheral on bleQueue
  BLEManager+Central.swift       — CBCentralManagerDelegate, CBPeripheralDelegate
  BLEManager+Peripheral.swift    — CBPeripheralManagerDelegate
  MessageProtocol.swift          — MessageEnvelope codec, chunking (480B), reassembly
  MeshRouter.swift               — Actor: outbound queue, store-and-forward, dedup
  ReassemblyBuffer.swift         — Collects BLE chunks, tracks completion + expiry

Crypto/
  CryptoManager.swift            — ECDH key agreement → HKDF → AES-GCM seal/open
  KeyStore.swift                 — Keychain CRUD for identity + peer keys

Persistence/
  PigeonStore.swift              — @MainActor SwiftData facade, DTO ↔ @Model mapping
  StoredMessage.swift            — @Model: persistent message record
  StoredConversation.swift       — @Model: persistent conversation record
  StoredPeer.swift               — @Model: persistent peer record

Services/
  AppCoordinator.swift           — @Observable hub: owns identity, store, BLE, crypto, router
  NotificationManager.swift      — Local notification dispatch

Views/
  MainTabView.swift              — 4 tabs: Messages, Nearby, Identity, Settings
  ConversationListView.swift     — Conversation list with empty state
  ConversationRowView.swift      — Row: avatar, name, preview, timestamp, unread badge
  ChatView.swift                 — Message bubbles + input bar
  MessageBubbleView.swift        — Incoming/outgoing bubble with status icon
  PeerDiscoveryView.swift        — Live nearby peer list with scanning animation
  PeerRowView.swift              — Peer row with signal strength
  IdentityView.swift             — PigeonID display, public key, QR code
  QRCodeSheet.swift              — QR code generator sheet
  SettingsView.swift             — BLE status, mesh toggle, reset identity
  PigeonTheme.swift              — Dark theme: colors (#0A0A0A bg, #8B5CF6 accent), fonts
```

## Architecture

### Dependency Chain

```
PigeonApp → AppCoordinator → BLEManager → MeshRouter (actor)
                           → CryptoManager
                           → PigeonStore → SwiftData ModelContainer
```

`AppCoordinator` is `@Observable` and `@MainActor`. It is injected into SwiftUI views via `.environment(coordinator)`.

### Message Flow (Send)

1. View calls `coordinator.sendMessage(text:in:conversation)`
2. Coordinator: create Message DTO → save to SwiftData → encrypt → enqueue in MeshRouter → BLEManager.sendMessage()
3. BLEManager: chunk envelope → write chunks to peer's BLE characteristic
4. Peer reassembles → decrypts → saves → shows notification
5. Peer sends ACK → sender updates status to `.delivered`

### Message Flow (Relay)

If message `recipientPublicKey != ourPublicKey`:
1. Dedup check via `MeshRouter.markSeen()`
2. Increment hop count, check against TTL (default 5)
3. Store in forwarding cache (1 hour retention)
4. Forward to all connected peers except sender

### Two Model Layers

| Layer | Types | Purpose |
|-------|-------|---------|
| DTOs | `Message`, `Peer`, `Conversation`, `MessageEnvelope` | Networking, crypto, routing. Plain `Codable` structs marked `nonisolated Sendable` |
| Persistence | `StoredMessage`, `StoredConversation`, `StoredPeer` | SwiftData `@Model` classes. Never used outside `PigeonStore` |

`PigeonStore` converts between layers. Never annotate DTOs with `@Model`.

## BLE Protocol Details

| Item | Value |
|------|-------|
| Service UUID | `E1A71B10-9F11-4F1F-93F8-1A0D3A2B0001` |
| Message Characteristic | `...0002` (write, writeWithoutResponse, notify) |
| Identity Characteristic | `...0003` (read) |
| ACK Characteristic | `...0004` (write, writeWithoutResponse, notify) |
| Max chunk payload | 480 bytes |
| Packet header | 22 bytes: UUID(16) + chunkIndex(2) + totalChunks(2) + payloadSize(2), big-endian |
| Default TTL | 5 hops |
| Forwarding retention | 1 hour |
| Reassembly timeout | 30 seconds |

## Concurrency Rules

The project uses `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`. This means:

1. **All types are `@MainActor` by default** unless explicitly opted out
2. **DTOs and utilities** are marked `nonisolated struct/enum ... : Sendable`
3. **MeshRouter** is a Swift `actor` (provides its own isolation)
4. **BLEManager** is `NSObject` with a dedicated `DispatchQueue` (`bleQueue`) for CoreBluetooth
5. **BLEManagerDelegate** methods are `nonisolated` — they bridge to `@MainActor` via `Task { @MainActor in ... }`
6. **PigeonStore** is `@MainActor` — all SwiftData operations use the main model context
7. Properties accessed from extensions in other files cannot be `private` (Swift `private` = file-private)

## Things NOT to Build

Group chats, voice/media, read receipts, cloud backup, LoRa hardware, web/Android, App Store submission, any server component.
