# CLAUDE.md — Pigeon

## What is this project?

Pigeon is an iOS BLE mesh messenger. Encrypted text messages between iPhones over Bluetooth — no internet, no server. Each device acts as both client and relay node. Messages hop across nearby devices until they reach their destination.

## Quick Reference

```bash
# Build (from Pigeon/ directory containing .xcodeproj)
cd Pigeon
xcodebuild -scheme Pigeon -destination 'platform=iOS Simulator,name=iPhone 17 Pro' build

# Clean build
xcodebuild -scheme Pigeon -destination 'platform=iOS Simulator,name=iPhone 17 Pro' clean build
```

BLE only works on physical devices, not simulators.

## Tech Stack

- Swift 5.9+, SwiftUI, iOS 26.0+
- CoreBluetooth (BLE mesh), CryptoKit (E2E encryption), SwiftData (persistence)
- MVVM with `@Observable` (Observation framework)
- Zero external dependencies
- Strict concurrency: `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`

## Project Layout

```
Pigeon/Pigeon/
├── PigeonApp.swift              # Entry point, creates AppCoordinator
├── Models/                      # Plain Codable DTO structs (nonisolated, Sendable)
│   ├── PigeonIdentity.swift     # Curve25519 keypair + pigeonID
│   ├── Message.swift            # Message DTO with status enum
│   ├── Peer.swift               # Peer DTO with RSSI
│   └── Conversation.swift       # Conversation DTO with unread count
├── Networking/                  # BLE transport + mesh routing
│   ├── BLEManager.swift         # Core BLE orchestrator (Central + Peripheral)
│   ├── BLEManager+Central.swift # CBCentralManagerDelegate + CBPeripheralDelegate
│   ├── BLEManager+Peripheral.swift # CBPeripheralManagerDelegate
│   ├── BLEConstants.swift       # UUIDs, protocol constants
│   ├── MessageProtocol.swift    # Envelope/packet codecs, chunking, reassembly
│   ├── MeshRouter.swift         # Actor: outbound queue, forwarding, dedup
│   └── ReassemblyBuffer.swift   # Multi-chunk message assembly
├── Crypto/
│   ├── CryptoManager.swift      # Curve25519 ECDH + AES-256-GCM
│   └── KeyStore.swift           # Keychain storage
├── Persistence/                 # SwiftData layer
│   ├── PigeonStore.swift        # CRUD facade (@MainActor)
│   ├── StoredMessage.swift      # @Model for messages
│   ├── StoredConversation.swift # @Model for conversations
│   └── StoredPeer.swift         # @Model for peers
├── Services/
│   ├── AppCoordinator.swift     # @Observable owner of all services
│   └── NotificationManager.swift # Local notifications
└── Views/
    ├── MainTabView.swift        # 4-tab navigation
    ├── ConversationListView.swift
    ├── ConversationRowView.swift
    ├── ChatView.swift           # Message thread + input
    ├── MessageBubbleView.swift  # Bubble UI with status
    ├── PeerDiscoveryView.swift  # Nearby BLE peers
    ├── PeerRowView.swift
    ├── IdentityView.swift       # PigeonID + QR code
    ├── QRCodeSheet.swift
    ├── SettingsView.swift
    └── PigeonTheme.swift        # Colors + typography
```

## Critical Patterns

### Concurrency
All types are `@MainActor` by default (project setting). Data types used across isolation boundaries (`Message`, `Peer`, `Conversation`, `MessageEnvelope`, `CryptoManager`, etc.) are explicitly `nonisolated` and `Sendable`. The `MeshRouter` is a Swift `actor`. `BLEManager` is `NSObject` using a dedicated `DispatchQueue` for CoreBluetooth callbacks. `BLEManagerDelegate` methods are `nonisolated` and dispatch to `@MainActor` via `Task { @MainActor in }`.

### Two-Layer Models
DTOs (`Message`, `Peer`, `Conversation`) are plain Codable structs used by networking and crypto. `@Model` classes (`StoredMessage`, `StoredConversation`, `StoredPeer`) are SwiftData persistence. `PigeonStore` maps between them. Do NOT merge these layers.

### Environment Injection
`AppCoordinator` is passed via `.environment(coordinator)`. All views access it with `@Environment(AppCoordinator.self)`.

### BLE Dual Role
Every device runs as both CBCentralManager (scanner) and CBPeripheralManager (advertiser) simultaneously on the same `bleQueue`. This is how mesh works.

## Common Tasks

**Add a new view**: Create in `Views/`, access coordinator via `@Environment(AppCoordinator.self)`, use `PigeonTheme` for all colors/fonts.

**Add a new persisted field**: Update both the DTO struct in `Models/` AND the `@Model` class in `Persistence/`. Update the `toDTO()` and `init(from:)` mapping methods. Update `PigeonStore` if new queries are needed.

**Modify BLE behavior**: Core logic is in `BLEManager.swift`. Delegate callbacks are split into `+Central.swift` and `+Peripheral.swift`. Properties accessed from extensions must not be `private` (file-private in Swift prevents cross-file extension access).

## Known Constraints

- iOS Simulator does NOT support BLE. All BLE testing requires 2+ physical iPhones.
- BLE MTU is ~512 bytes. Messages are chunked into 480-byte payloads with 22-byte headers.
- iOS allows ~7 simultaneous BLE connections.
- Background BLE operation requires `UIBackgroundModes` with `bluetooth-central` and `bluetooth-peripheral` (configured in `Info.plist`).
- Mesh relay TTL defaults to 5 hops. Forwarded messages are held for 1 hour before expiry.
