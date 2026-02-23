# Pigeon — AI Context

## What Is This?

iOS BLE mesh messenger. Encrypted text messages between iPhones over Bluetooth. No internet. No server. Each device is both client and relay node. End-to-end encrypted with Curve25519 + AES-256-GCM.

## Build

```
cd Pigeon && xcodebuild -scheme Pigeon -destination 'platform=iOS Simulator,name=iPhone 17 Pro' build
```

## Stack

Swift 5.9+ / SwiftUI / iOS 26.0+ / CoreBluetooth / CryptoKit / SwiftData / @Observable MVVM / Zero external deps

## Architecture Overview

```
PigeonApp.swift
  └─ AppCoordinator (@Observable, @MainActor)
       ├─ PigeonIdentity ─── Curve25519 keypair, pigeonID
       ├─ PigeonStore ────── SwiftData CRUD (@MainActor)
       ├─ CryptoManager ──── ECDH + AES-GCM encrypt/decrypt
       ├─ MeshRouter ─────── actor: outbound queue, forwarding, dedup
       └─ BLEManager ─────── NSObject: Central + Peripheral on bleQueue
```

Views access AppCoordinator via `@Environment(AppCoordinator.self)`.

## Concurrency (CRITICAL)

`SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` — everything is @MainActor by default.

- DTOs (`Message`, `Peer`, `Conversation`, `MessageEnvelope`): `nonisolated Sendable Codable` structs
- `MeshRouter`: Swift `actor`
- `BLEManager`: `NSObject` on DispatchQueue `bleQueue`
- `PigeonStore`: `@MainActor`
- BLEManagerDelegate methods: `nonisolated`, hop to MainActor via `Task { @MainActor in ... }`

## Model Architecture

Two layers. Do not merge them.

| Layer | Types | Location | Used By |
|-------|-------|----------|---------|
| DTOs | `Message`, `Peer`, `Conversation`, `MessageEnvelope` | `Models/` + `MessageProtocol.swift` | Everything |
| Persistence | `StoredMessage`, `StoredConversation`, `StoredPeer` | `Persistence/` | Only `PigeonStore` |

`PigeonStore` maps between them via `init(from:)` and `toDTO()`.

## BLE Protocol

- Service: `E1A71B10-9F11-4F1F-93F8-1A0D3A2B0001`
- Characteristics: identity (read `...0003`), message (write+notify `...0002`), ACK (write+notify `...0004`)
- Chunking: 480B payloads, 22B binary header (UUID + chunkIndex + totalChunks + payloadSize)
- TTL: 5 hops, forwarding retention: 1hr, reassembly timeout: 30s

## File Map (32 Swift files)

```
PigeonApp.swift              — Entry point

Models/ (4)
  PigeonIdentity.swift       — Keypair, pigeonID, display name
  Message.swift              — Message DTO + MessageStatus enum
  Peer.swift                 — Peer DTO (pubkey, rssi, saved)
  Conversation.swift         — Conversation DTO (unread, preview)

Networking/ (7)
  BLEConstants.swift         — UUIDs, protocol constants
  BLEManager.swift           — Central+Peripheral orchestrator
  BLEManager+Central.swift   — CBCentralManagerDelegate
  BLEManager+Peripheral.swift — CBPeripheralManagerDelegate
  MessageProtocol.swift      — Envelope codec, chunking, reassembly
  MeshRouter.swift           — Actor: routing, forwarding, dedup
  ReassemblyBuffer.swift     — Chunk collection + timeout

Crypto/ (2)
  CryptoManager.swift        — Curve25519 → HKDF → AES-GCM
  KeyStore.swift             — Keychain storage

Persistence/ (4)
  PigeonStore.swift          — SwiftData CRUD facade
  StoredMessage.swift        — @Model message
  StoredConversation.swift   — @Model conversation
  StoredPeer.swift           — @Model peer

Services/ (2)
  AppCoordinator.swift       — @Observable service hub
  NotificationManager.swift  — Local notifications

Views/ (11)
  MainTabView.swift          — 4 tabs
  ConversationListView.swift — Message threads
  ConversationRowView.swift  — Thread row
  ChatView.swift             — Chat + input
  MessageBubbleView.swift    — Bubble + status
  PeerDiscoveryView.swift    — Nearby peers
  PeerRowView.swift          — Peer row + signal
  IdentityView.swift         — ID + QR
  QRCodeSheet.swift          — QR display
  SettingsView.swift         — App settings
  PigeonTheme.swift          — Colors + fonts

Utilities/ (1)
  Extensions.swift           — Data hex, UUID bytes
```

## Design

Dark theme. `PigeonTheme` enum has all tokens. Background `#0A0A0A`, accent `#8B5CF6` (violet), outgoing bubble `#7C3AED`, incoming `#2A2A2A`. Use `PigeonTheme.x` for all colors/fonts.

## Out of Scope

No group chats, no media, no read receipts, no cloud, no LoRa, no web/Android.
