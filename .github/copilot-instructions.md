# Copilot Instructions — Pigeon

## Project

Pigeon is an iOS BLE mesh messenger. Encrypted text messages over Bluetooth between iPhones with no internet. Each device is both client and relay. Built with Swift/SwiftUI/CoreBluetooth/CryptoKit/SwiftData, zero external dependencies.

## Build

```bash
cd Pigeon && xcodebuild -scheme Pigeon -destination 'platform=iOS Simulator,name=iPhone 17 Pro' build
```

## Architecture

- **AppCoordinator** (`@Observable`, `@MainActor`): owns all services, injected via `.environment()`
- **BLEManager** (`NSObject`): dual Central+Peripheral on `bleQueue` DispatchQueue
- **MeshRouter** (Swift `actor`): outbound queue, store-and-forward, message dedup
- **CryptoManager**: Curve25519 ECDH → HKDF → AES-256-GCM
- **PigeonStore** (`@MainActor`): SwiftData CRUD, maps DTO structs ↔ @Model classes

## Key Rules

1. **Strict concurrency**: `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`. All types default to `@MainActor`. Data types crossing isolation boundaries must be `nonisolated` and `Sendable`.

2. **Two-layer models**: DTOs (`Message`, `Peer`, `Conversation`) are `nonisolated Codable Sendable` structs for networking/crypto. `@Model` classes (`StoredMessage`, etc.) are for persistence only. Never merge them.

3. **No private in BLEManager**: Properties accessed by delegate extensions in other files must not be `private` (file-private prevents cross-file access).

4. **Colors via PigeonTheme**: Dark theme (#0A0A0A background, #8B5CF6 violet accent). All colors/fonts through `PigeonTheme` enum. Never hardcode.

5. **BLE chunking**: Messages > 480 bytes get chunked with 22-byte binary headers. `MessageProtocol` handles encode/decode/chunk/reassemble.

6. **Environment access**: Views use `@Environment(AppCoordinator.self) private var coordinator`.

## Source Structure

```
Pigeon/Pigeon/
├── PigeonApp.swift           # Entry: AppCoordinator + ModelContainer
├── Models/                   # DTO structs (nonisolated, Sendable, Codable)
├── Networking/               # BLE transport, mesh routing, protocol codec
├── Crypto/                   # E2E encryption + Keychain storage
├── Persistence/              # SwiftData @Model classes + PigeonStore facade
├── Services/                 # AppCoordinator + NotificationManager
└── Views/                    # SwiftUI views + PigeonTheme
```
