# FORGOTTEN NFT ENGINE
## Technical Development Specification for Claude Code
**Version:** 1.0  
**Platform:** C# / .NET / Discord.Net / SQLite / Solana  
**Project:** FORGOTTEN Discord Bot  
**Architecture:** Modular, multi-tier, extensible  
**Primary Objective:** Universal Solana NFT verification, display, ownership management, and gaming integration.

This specification is designed to be given directly to Claude Code. It supports any compatible Solana NFT collection, with Dead Mint Carnival as an optional integration rather than a hardcoded dependency.

# FORGOTTEN
# Universal Solana NFT Verification and Collection Engine

## 1. DEVELOPMENT DIRECTIVE

You are a senior C#/.NET software architect specializing in Discord.Net, blockchain integration, cryptographic authentication, SQLite, and distributed application design.

Your assignment is to design, implement, test, and document a production-quality NFT management subsystem for an existing Discord.Net bot named FORGOTTEN.

The system must allow Discord members to inspect Solana NFTs, verify NFT authenticity, prove wallet ownership, browse collections, and display their NFTs through interactive Discord interfaces.

The subsystem must be blockchain-aware, secure, modular, and extensible.

The NFT engine must operate independently of any specific NFT collection or game.

Future integrations may include Dead Mint Carnival, POW fighting games, NFT tournaments, role-based access control, trading workflows, and collectible progression systems.

**Do not build a demonstration-only application. Implement production-oriented functionality with automated tests, migrations, configuration, structured logging, and documentation.**

---

# 2. PROJECT REQUIREMENTS

## 2.1 Technology Stack

Use the following technologies:

| Component | Technology |
|---|---|
| Programming language | C# |
| Runtime | .NET 10 LTS |
| Discord integration | Discord.Net |
| Database | SQLite |
| ORM | Entity Framework Core |
| Blockchain | Solana |
| Blockchain interface | DAS-compatible RPC API |
| API communication | HttpClientFactory |
| Authentication | Solana Ed25519 signatures |
| Serialization | System.Text.Json |
| Logging | Microsoft.Extensions.Logging |
| Dependency injection | Microsoft.Extensions.DependencyInjection |
| Configuration | Microsoft.Extensions.Options |
| Background processing | IHostedService |
| Testing | xUnit |
| HTTP resilience | Microsoft.Extensions.Http.Resilience |
| Optional caching | IMemoryCache |

Use stable, mutually compatible package versions.

Do not upgrade or replace existing project dependencies without evaluating compatibility.

## 2.2 Existing Project Integration

Before implementation:

1. Inspect the existing repository.
2. Identify the solution structure.
3. Identify the current Discord.Net version.
4. Identify existing dependency injection patterns.
5. Identify database configuration and migrations.
6. Identify existing logging infrastructure.
7. Identify existing Discord command registration.
8. Identify existing member and wallet records.
9. Identify existing POW game integration points.
10. Produce a short architectural assessment.

Extend the existing application rather than creating a competing framework.

If no existing implementation exists, create the architecture specified below.

Do not modify unrelated bot functionality.

---

# 3. ARCHITECTURE

Implement a modular, layered architecture.

```text
FORGOTTEN
│
├── Discord Presentation Layer
│   ├── NFT Commands
│   ├── Wallet Commands
│   ├── Collection Commands
│   ├── Gallery Commands
│   ├── NFT Embeds
│   └── Interactive Components
│
├── Application Layer
│   ├── NFT Inspection Service
│   ├── NFT Verification Service
│   ├── NFT Ownership Service
│   ├── Wallet Connection Service
│   ├── NFT Gallery Service
│   ├── Collection Registry Service
│   ├── NFT Metadata Service
│   └── Ownership Monitoring Service
│
├── Domain Layer
│   ├── NFT Asset
│   ├── NFT Collection
│   ├── Wallet Connection
│   ├── Verification Challenge
│   ├── Ownership Record
│   ├── NFT Attribute
│   └── Verification Result
│
├── Infrastructure Layer
│   ├── Solana DAS Client
│   ├── RPC Provider Adapter
│   ├── Signature Verification
│   ├── Metadata Resolver
│   ├── SQLite Repositories
│   ├── Background Workers
│   └── Caching
│
└── Integration Layer
    ├── Discord Role Provider
    ├── NFT Ownership Events
    ├── External Game Adapter
    └── POW Fighter Adapter
```

### Architectural Rules

- Discord command handlers must remain thin.
- Business logic belongs in application services.
- Domain entities must not depend on Discord.Net.
- Blockchain provider implementations must be replaceable.
- Database persistence must be separated from blockchain queries.
- All external API calls must support cancellation.
- All blockchain responses must be validated before use.
- Services must be testable without Discord or live Solana access.
- No collection-specific logic belongs in the core NFT engine.

---

# 4. NFT VERIFICATION ENGINE

## 4.1 NFT Inspection

Implement:

```csharp
public interface INftInspectionService
{
    Task<NftInspectionResult> InspectAsync(
        string assetAddress,
        CancellationToken cancellationToken = default);
}
```

The service must retrieve:

- NFT asset identifier
- Asset interface/type
- NFT name
- Symbol, when available
- Description
- Image URL
- Animation URL
- Metadata URI
- Collection identifiers
- Collection verification information
- Current owner
- Supply and token-standard information
- Creators and creator verification
- NFT attributes
- Compression status
- Burn status
- Mutability information, when available

Support standard Solana NFTs, programmable NFTs, and compressed NFTs when the selected DAS provider supports them.

Unsupported assets must return an explicit status rather than being incorrectly classified as valid NFTs.

## 4.2 Verification Levels

Implement distinct verification states.

### Level 1: Asset Exists

The asset can be retrieved from the blockchain provider.

### Level 2: NFT Classification

The asset is identified as a supported NFT rather than an ordinary fungible token.

### Level 3: Collection Verification

The NFT's collection identity and verification status are established using trusted on-chain information.

### Level 4: Approved Collection

The collection belongs to a registry explicitly approved by server administrators.

### Level 5: Wallet Ownership

The current owner is identified using authoritative blockchain data.

### Level 6: Discord Member Ownership

The current owner matches a wallet whose control was cryptographically proven by the requesting Discord member.

### Level 7: Application Eligibility

The NFT meets additional requirements imposed by an external application or game.

Verification levels must remain separate.

A valid NFT is not automatically an approved NFT.

A valid NFT is not automatically owned by the requesting member.

An NFT from a verified collection is not automatically authorized for POW battles.

## 4.3 Verification Results

Create:

```csharp
public enum NftVerificationStatus
{
    Verified,
    Unverified,
    Unsupported,
    NotFound,
    Burned,
    InvalidCollection,
    OwnershipMismatch,
    ProviderUnavailable,
    Indeterminate
}
```

Create a result object containing:

```text
AssetAddress
AssetExists
IsSupportedNft
CollectionVerified
CollectionApproved
OwnerAddress
OwnershipVerified
MemberOwnershipVerified
VerificationStatus
VerificationTimestamp
Provider
ErrorCode
ErrorMessage
```

A provider timeout or temporary RPC failure must never be interpreted as proof that an NFT does not exist.

---

# 5. SOLANA BLOCKCHAIN INTEGRATION

## 5.1 Provider Abstraction

Implement:

```csharp
public interface ISolanaAssetProvider
{
    Task<NftAsset?> GetAssetAsync(
        string assetAddress,
        CancellationToken cancellationToken);

    Task<PagedResult<NftAsset>> GetAssetsByOwnerAsync(
        string walletAddress,
        int page,
        int pageSize,
        CancellationToken cancellationToken);

    Task<PagedResult<NftAsset>> GetAssetsByCollectionAsync(
        string collectionAddress,
        int page,
        int pageSize,
        CancellationToken cancellationToken);
}
```

Use the provider's supported DAS methods, including:

```text
getAsset
getAssetsByOwner
getAssetsByGroup
```

Do not assume every Solana RPC endpoint supports DAS.

The implementation must support configuring a compatible DAS endpoint independently from a standard Solana RPC endpoint.

## 5.2 Provider Configuration

```json
{
  "Solana": {
    "Network": "mainnet-beta",
    "DasEndpoint": "",
    "RpcEndpoint": "",
    "RequestTimeoutSeconds": 15,
    "MaxRetries": 3,
    "DefaultPageSize": 20,
    "MaxPageSize": 100,
    "CacheDurationSeconds": 300
  }
}
```

API keys must be supplied through environment variables, user secrets, or another secure configuration provider.

Never commit credentials.

## 5.3 Provider Resilience

Implement:

- Configurable timeouts
- Exponential backoff
- Retry with jitter
- HTTP connection pooling
- Cancellation support
- Rate-limit handling
- Request concurrency limits
- Structured error handling
- Response validation
- Provider health monitoring

Retry only transient failures.

Do not retry permanently invalid asset addresses.

Prevent unbounded pagination and excessively large collection scans.

## 5.4 Blockchain Data Normalization

Normalize provider-specific responses into internal domain objects.

Store the source provider and retrieval timestamp.

Preserve raw provider responses only when necessary for diagnostics, with retention and size limits.

Do not assume every NFT has an image, description, creator, or attribute list.

---

# 6. DISCORD NFT COMMANDS

Implement Discord.Net slash commands using the existing interaction framework.

## 6.1 NFT Inspection

Command:

`/nft inspect <asset_address>`

Behavior:

1. Validate the address format.
2. Query the NFT provider.
3. Normalize metadata.
4. Determine verification status.
5. Build a Discord embed.
6. Display the NFT artwork.
7. Display ownership and collection information.
8. Provide links to a configured Solana explorer.

Example:

```text
🎪 FORGOTTEN NFT EXPLORER

NFT: INFERNA

Collection: Dead Mint Carnival
Collection Verified: YES
Approved Collection: YES

Owner: 7Xh...9aK
Blockchain: Solana

Attributes:
Class: Fire Warrior
Magic: Phoenix Strike
Rarity: Legendary

Status: VERIFIED

[View on Explorer]
[View Collection]
[Show Attributes]
```

The embed must display the actual NFT image when available.

If artwork is missing or unsupported, display a configured placeholder.

Never claim collection verification based solely on the NFT's name or image.

## 6.2 NFT Gallery

Command:

`/nft gallery`

Requirements:

- Require a connected wallet.
- Retrieve NFTs owned by the wallet.
- Support multiple linked wallets.
- Show collection names.
- Display artwork.
- Support pagination.
- Allow collection filtering.
- Allow selecting an individual NFT.

Buttons:

```text
[Previous]
[Next]
[Inspect]
[Collection]
[Refresh]
```

Each gallery interaction must validate the requesting member and enforce session ownership.

Gallery pagination must not download an entire large wallet inventory into memory.

## 6.3 NFT Collection

Command:

`/nft collection <collection_address>`

Display:

- Collection identifier
- Approved registry name, if available
- Collection verification status
- Available collection metadata
- NFTs retrieved through paginated queries
- Supported asset count information

Do not invent a total supply when the provider does not supply one.

## 6.4 NFT Verification

Command:

`/nft verify <asset_address>`

Display:

```text
Asset Exists: YES
Supported NFT: YES
Collection Verified: YES
Collection Approved: YES
Current Owner: 7Xh...9aK
Connected Wallet Match: YES
Member Ownership: VERIFIED
```

Verification must be performed using sufficiently fresh blockchain data.

## 6.5 Additional Commands

Implement:

```text
/nft inspect
/nft verify
/nft gallery
/nft collection
/nft attributes
/nft refresh
/nft help

/wallet connect
/wallet verify
/wallet list
/wallet disconnect
/wallet status
```

Administrator commands:

```text
/nft-admin collection-add
/nft-admin collection-remove
/nft-admin collection-list
/nft-admin provider-status
/nft-admin cache-clear
/nft-admin verification-log
```

Administrative commands must enforce server-specific permissions.

---

# 7. WALLET OWNERSHIP VERIFICATION

## 7.1 Security Requirement

A member must never gain verified wallet ownership by simply entering a wallet address.

Wallet ownership must be proven cryptographically.

## 7.2 Connection Workflow

1. Member executes `/wallet connect`.
2. Bot generates a short-lived verification session.
3. Bot returns a secure wallet-verification URL.
4. Member opens the verification page.
5. Member connects a compatible Solana wallet.
6. Backend generates a unique challenge.
7. Member signs the challenge.
8. Backend verifies the signature.
9. Backend confirms the challenge is valid and unused.
10. Backend associates the wallet with the Discord member.
11. Bot confirms successful verification.

Use a wallet-standard-compatible browser integration.

Do not attempt to access wallet private keys.

## 7.3 Verification Challenge

The signed message must include:

```text
FORGOTTEN Wallet Verification

Domain: <configured-domain>
Discord User ID: <discord-user-id>
Wallet Address: <wallet-address>
Nonce: <random-nonce>
Issued At: <timestamp>
Expires At: <timestamp>
Purpose: Link Solana wallet
```

The challenge must be:

- Cryptographically random
- Single-use
- Short-lived
- Bound to the intended Discord account
- Bound to the intended wallet address
- Bound to the application domain
- Protected against replay attacks

The backend must verify the exact signed message bytes using Ed25519.

Store only a hash of the nonce when practical.

Require HTTPS for the verification website.

Use a secure session-binding mechanism so another user cannot claim a Discord account merely by obtaining a verification URL.

## 7.4 Wallet Management

Support:

- Multiple wallets per member
- Primary wallet selection
- Wallet disconnection
- Reverification
- Wallet verification timestamps
- Ownership synchronization

Define a configurable policy for whether one wallet may be linked to multiple Discord accounts.

Default policy: one verified wallet may belong to only one active Discord identity at a time.

Changing wallet associations must require a fresh ownership proof.

---

# 8. NFT METADATA SYSTEM

## 8.1 Metadata Resolution

Support metadata retrieved from:

- HTTPS
- IPFS
- Arweave

Use configurable gateways for decentralized storage.

Resolve:

```text
name
symbol
description
image
animation_url
external_url
attributes
properties
```

Preserve additional metadata fields when appropriate.

## 8.2 Metadata Security

Metadata is untrusted input.

Implement:

- Maximum response size
- Request timeouts
- Content-type validation
- URL scheme validation
- Redirect limits
- HTML/Markdown escaping
- Discord mention suppression
- Metadata length limits
- SSRF protection

Reject requests to loopback, private, link-local, cloud metadata, and other restricted network destinations.

Revalidate resolved IP addresses across redirects.

Do not execute scripts or arbitrary HTML embedded in NFT metadata.

Never treat metadata claims as authoritative proof of ownership or collection membership.

## 8.3 Image Display

Support:

- PNG
- JPEG
- GIF
- WebP when supported by Discord
- Safe external links for unsupported media

Use Discord embeds for ordinary image display.

Animated media must degrade gracefully when Discord cannot render it.

---

# 9. SQLITE DATABASE

Use EF Core migrations.

Do not automatically delete or recreate an existing database.

## 9.1 Tables

### DiscordWalletConnections

```text
Id
DiscordUserId
WalletAddress
Blockchain
IsPrimary
VerifiedAt
LastVerifiedAt
DisconnectedAt
CreatedAt
UpdatedAt
```

### WalletVerificationChallenges

```text
Id
DiscordUserId
WalletAddress
NonceHash
ChallengeHash
ExpiresAt
ConsumedAt
CreatedAt
Status
```

### NftCollections

```text
Id
Blockchain
CollectionAddress
DisplayName
Description
IsApproved
ApprovedByDiscordUserId
ApprovedAt
CreatedAt
UpdatedAt
```

### NftAssets

```text
Id
Blockchain
AssetAddress
CollectionAddress
Name
Symbol
Description
ImageUrl
AnimationUrl
MetadataUri
OwnerAddress
IsCompressed
IsBurned
LastFetchedAt
CreatedAt
UpdatedAt
```

### NftAttributes

```text
Id
NftAssetId
TraitType
TraitValue
DisplayType
```

### NftOwnershipHistory

```text
Id
NftAssetId
PreviousOwnerAddress
NewOwnerAddress
ObservedAt
Source
TransactionSignature
```

### NftVerificationLogs

```text
Id
DiscordUserId
AssetAddress
VerificationType
VerificationStatus
Provider
CheckedAt
ErrorCode
```

### NftGallerySessions

```text
Id
DiscordUserId
GuildId
WalletAddress
CollectionFilter
CurrentPage
ExpiresAt
CreatedAt
```

## 9.2 Database Rules

- Use unique indexes for blockchain asset identifiers.
- Use unique constraints for active wallet associations.
- Use foreign keys.
- Use UTC timestamps.
- Use parameterized database operations.
- Use transactions for multi-record updates.
- Use indexes for Discord user IDs and wallet addresses.
- Enable appropriate SQLite busy timeouts.
- Configure WAL mode where suitable.
- Avoid unnecessary database writes during gallery navigation.

Ownership history must distinguish observed state changes from confirmed blockchain transfers.

Do not fabricate transaction signatures from polling results.

---

# 10. NFT OWNERSHIP MONITORING

Implement an optional background worker.

Purpose:

Periodically verify ownership of NFTs relevant to connected wallets and registered applications.

## 10.1 Monitoring Requirements

- Configurable polling interval
- Bounded concurrency
- Incremental synchronization
- Provider rate-limit awareness
- Cancellation support
- Failure recovery
- Duplicate event prevention

## 10.2 Ownership Change Detection

When the observed owner changes:

1. Retrieve fresh ownership information.
2. Compare with the previously stored owner.
3. Update the ownership snapshot.
4. Record the observed change.
5. Publish an ownership-change event.
6. Notify subscribed integrations.

Do not infer an exact transfer transaction without transaction-level evidence.

## 10.3 Domain Events

Implement:

```csharp
public record NftOwnershipChangedEvent(
    string AssetAddress,
    string? PreviousOwner,
    string? CurrentOwner,
    DateTimeOffset ObservedAt);
```

Also define:

```text
NftVerifiedEvent
NftCollectionApprovedEvent
WalletConnectedEvent
WalletDisconnectedEvent
NftOwnershipChangedEvent
NftEligibilityChangedEvent
```

Events must be idempotent where appropriate.

---

# 11. DISCORD ROLE INTEGRATION

Implement optional NFT-based role assignment.

Example:

A member owns an NFT from an approved collection.

FORGOTTEN assigns a configured Discord role.

When the member no longer owns a qualifying NFT, the role can be removed.

## Requirements

- Per-guild role configuration
- Collection-to-role mapping
- Fresh ownership verification
- Background reconciliation
- Configurable grace periods for temporary provider outages
- Audit logging
- Discord permission checks

Do not remove roles solely because the blockchain provider is temporarily unavailable.

Do not assign privileged administrative roles automatically.

Role updates must respect Discord's role hierarchy and permissions.

---

# 12. NFT GALLERY USER INTERFACE

The gallery must use Discord.Net interactive components.

Example:

```text
╔══════════════════════════════╗
║       FORGOTTEN NFT          ║
║          GALLERY             ║
╚══════════════════════════════╝

Collection: Dead Mint Carnival

NFT: INFERNA
Owner: @Member
Rarity: Legendary

[ NFT ARTWORK ]

NFT 3 OF 12

[◀ Previous] [Next ▶]

[Inspect NFT]
[View Collection]
[Refresh Ownership]
```

## UI Requirements

- Display one NFT per page by default.
- Allow configurable gallery page size.
- Support filtering by collection.
- Display loading and error states.
- Prevent unrelated users from controlling private gallery sessions.
- Expire inactive sessions.
- Handle deleted Discord messages.
- Handle missing artwork.
- Avoid leaking private wallet connection information through public commands.

Use ephemeral responses for sensitive account-management actions.

Public NFT showcases must require explicit member action.

---

# 13. NFT COLLECTION REGISTRY

Implement an administrator-managed collection registry.

The registry allows server administrators to approve collections for special functionality.

Example:

```text
Collection: Dead Mint Carnival
Blockchain: Solana
Approved: YES
NFT Display: ENABLED
Wallet Verification: ENABLED
Role Access: ENABLED
Game Integration: ENABLED
```

A collection's registry approval must not be confused with its on-chain verification status.

Support multiple collections per Discord server.

Registry entries must include the approving administrator and timestamp.

Collection policies must be scoped by guild.

---

# 14. EXTERNAL GAME INTEGRATION

The NFT engine must expose reusable services for external applications.

Do not tightly couple the NFT engine to the POW game.

## 14.1 Ownership Interface

```csharp
public interface INftOwnershipService
{
    Task<bool> IsOwnedByWalletAsync(
        string assetAddress,
        string walletAddress,
        CancellationToken cancellationToken);
}
```

For security-sensitive operations, provide a richer result type that distinguishes ownership mismatch from provider failure.

## 14.2 Eligibility Interface

```csharp
public interface INftEligibilityService
{
    Task<NftEligibilityResult> CheckEligibilityAsync(
        ulong discordUserId,
        string assetAddress,
        string applicationId,
        CancellationToken cancellationToken);
}
```

## 14.3 POW Integration

Provide an adapter interface for a future POW integration.

Example:

```text
Discord Member
      |
      v
Verified Wallet
      |
      v
Owned NFT
      |
      v
NFT Fighter Mapping
      |
      v
POW Fighter Profile
      |
      v
Battle Engine
```

The NFT engine must not directly implement combat mechanics.

It must provide reliable ownership and eligibility information to the existing game.

## 14.4 Fighter Persistence

Future game integrations may maintain:

```text
NftAssetAddress
FighterId
Level
Experience
Wins
Losses
MagicAttack
LastBattleAt
```

These records belong to the game integration, not the core NFT subsystem.

When an NFT changes owners, fighter progression should remain associated with the NFT unless the game's explicit rules specify otherwise.

Ownership must be rechecked before granting control of a fighter.

---

# 15. CACHING STRATEGY

Implement separate cache policies.

| Data | Suggested TTL |
|---|---|
| NFT metadata | 5 minutes |
| Collection metadata | 15 minutes |
| Gallery results | 60 seconds |
| NFT attributes | 5 minutes |
| Ownership display | 30 seconds |
| Ownership authorization | Fresh provider check |
| Verification challenges | No reusable authorization cache |

These are configurable starting values.

Caching must never allow an expired ownership assertion to authorize sensitive operations.

Provide explicit cache invalidation after ownership changes and administrator actions.

---

# 16. SECURITY REQUIREMENTS

The following are mandatory:

1. Never request wallet private keys.
2. Never request seed phrases.
3. Never store wallet signing secrets.
4. Verify wallet signatures server-side.
5. Prevent challenge replay.
6. Validate all blockchain addresses.
7. Sanitize NFT metadata.
8. Prevent SSRF through metadata URLs.
9. Enforce administrator authorization.
10. Rate-limit Discord commands.
11. Protect API credentials.
12. Use HTTPS for wallet verification.
13. Avoid leaking sensitive account details.
14. Maintain security audit logs.
15. Fail closed when authorization cannot be verified.
16. Treat blockchain providers as external dependencies.
17. Prevent duplicate wallet claims.
18. Protect gallery interaction sessions.
19. Verify NFT ownership before game authorization.
20. Never interpret cached metadata as proof of current ownership.

A failed blockchain request must not result in granting access.

---

# 17. ERROR HANDLING

Implement clear user-facing errors.

Examples:

```text
NFT_NOT_FOUND
INVALID_ASSET_ADDRESS
UNSUPPORTED_ASSET
COLLECTION_NOT_VERIFIED
COLLECTION_NOT_APPROVED
WALLET_NOT_CONNECTED
WALLET_SIGNATURE_INVALID
WALLET_CHALLENGE_EXPIRED
WALLET_ALREADY_LINKED
NFT_NOT_OWNED
BLOCKCHAIN_PROVIDER_UNAVAILABLE
METADATA_UNAVAILABLE
GALLERY_SESSION_EXPIRED
INSUFFICIENT_PERMISSIONS
```

User-facing errors must not expose internal stack traces, provider credentials, or sensitive configuration.

Record technical details through structured logging.

---

# 18. TESTING REQUIREMENTS

Implement automated tests for:

### NFT Validation

- Valid NFT
- Invalid address
- Missing asset
- Burned asset
- Compressed NFT
- Unsupported token
- Unverified collection
- Approved collection
- Malformed metadata

### Wallet Verification

- Valid signature
- Invalid signature
- Expired challenge
- Reused challenge
- Incorrect wallet address
- Incorrect Discord account
- Concurrent verification attempts
- Duplicate wallet association

### Gallery

- Empty wallet
- Single NFT
- Multiple NFTs
- Pagination
- Missing artwork
- Provider timeout
- Unauthorized button interaction
- Expired session

### Ownership

- Correct owner
- Incorrect owner
- Ownership transfer
- Provider unavailable
- Stale cache
- Duplicate ownership events

### Security

- SSRF attempts
- Malicious metadata
- Oversized responses
- Invalid redirects
- Discord mention injection
- Unauthorized administrator commands

Tests must not depend on live blockchain services.

Use mocked provider responses and deterministic fixtures.

Provide optional integration tests against a configured Solana provider.

---

# 19. PERFORMANCE REQUIREMENTS

Design for multiple Discord guilds and concurrent users.

Initial performance objectives:

- Cached NFT inspection: under 500 ms
- Cached gallery navigation: under 500 ms
- Database lookup: under 100 ms under normal local load
- Blockchain requests: configurable timeout
- No unbounded collection enumeration
- No synchronous blocking of Discord gateway events

These are engineering targets, not guaranteed blockchain response times.

Use asynchronous operations throughout.

Defer Discord interactions before potentially slow external calls.

---

# 20. CONFIGURATION

Create strongly typed configuration classes.

Example:

```json
{
  "NftEngine": {
    "Enabled": true,
    "AllowUniversalNftInspection": true,
    "RequireApprovedCollectionForRoles": true,
    "RequireVerifiedWalletForGallery": true,
    "EnableOwnershipMonitoring": true,
    "OwnershipPollingMinutes": 5,
    "GallerySessionMinutes": 15,
    "DefaultGalleryPageSize": 1,
    "MaximumLinkedWallets": 5,
    "EnablePowIntegration": false
  }
}
```

Provide configuration validation at application startup.

Do not silently continue with invalid security-critical settings.

---

# 21. PROJECT STRUCTURE

Recommended structure:

```text
src/
│
├── Forgotten.Bot/
│   ├── Commands/
│   │   ├── NftCommands.cs
│   │   ├── WalletCommands.cs
│   │   └── NftAdminCommands.cs
│   │
│   ├── Components/
│   │   └── NftGalleryComponents.cs
│   │
│   └── Embeds/
│       └── NftEmbedFactory.cs
│
├── Forgotten.Nft.Domain/
│   ├── Entities/
│   ├── Enums/
│   ├── Events/
│   └── ValueObjects/
│
├── Forgotten.Nft.Application/
│   ├── Interfaces/
│   ├── Services/
│   ├── DTOs/
│   └── Validators/
│
├── Forgotten.Nft.Infrastructure/
│   ├── Blockchain/
│   ├── Metadata/
│   ├── Persistence/
│   ├── Security/
│   ├── Caching/
│   └── Workers/
│
├── Forgotten.Nft.WalletWeb/
│   ├── Endpoints/
│   ├── Authentication/
│   └── WalletIntegration/
│
└── Forgotten.Nft.Integrations/
    ├── DiscordRoles/
    └── Pow/
```

Testing:

```text
tests/
│
├── Forgotten.Nft.UnitTests/
├── Forgotten.Nft.IntegrationTests/
└── Forgotten.Nft.SecurityTests/
```

Adapt these directories to the existing repository.

---

# 22. IMPLEMENTATION PHASES

## Phase 1: Foundation

Implement:

- Project architecture
- Dependency injection
- Domain models
- SQLite database
- EF Core migrations
- Provider configuration
- DAS provider adapter
- NFT inspection service
- Metadata normalization
- Unit tests

Deliverable:

`/nft inspect` works against a configured Solana DAS provider.

## Phase 2: Verification

Implement:

- NFT classification
- Collection verification
- Collection approval registry
- Verification result models
- Administrator commands
- Verification audit logs

Deliverable:

`/nft verify` distinguishes NFT validity, collection authenticity, and administrative approval.

## Phase 3: Wallet Integration

Implement:

- Wallet verification web application
- Secure challenge generation
- Signature verification
- Wallet association
- Wallet management commands
- Verification persistence
- Security tests

Deliverable:

Members can securely connect and verify Solana wallets.

## Phase 4: NFT Gallery

Implement:

- Wallet NFT retrieval
- Pagination
- Interactive buttons
- Gallery session management
- Artwork display
- Collection filtering
- Caching

Deliverable:

Members can browse their owned NFTs inside Discord.

## Phase 5: Ownership Monitoring

Implement:

- Background ownership checks
- Ownership snapshots
- Change events
- Audit history
- Role integration
- Cache invalidation

Deliverable:

The bot can react safely to NFT ownership changes.

## Phase 6: Game Integration

Implement:

- External eligibility interface
- Ownership authorization
- POW integration adapter
- NFT-to-fighter mapping interface
- Ownership-change notifications

Deliverable:

The POW game can use verified NFT ownership without depending directly on blockchain APIs.

---

# 23. ACCEPTANCE CRITERIA

The implementation is complete when:

1. The existing Discord bot starts successfully.
2. Existing bot functionality remains operational.
3. NFT inspection works for supported Solana NFTs.
4. NFT artwork displays correctly in Discord.
5. Collection verification is reliable.
6. Unsupported assets are identified correctly.
7. Members can cryptographically verify wallets.
8. Wallet ownership cannot be forged by submitting an address.
9. Members can browse NFT galleries.
10. Gallery navigation works correctly.
11. NFT ownership can be verified.
12. Ownership changes are detected.
13. Administrative collection approval works.
14. SQLite migrations run successfully.
15. All critical automated tests pass.
16. Sensitive credentials remain protected.
17. Blockchain provider failures are handled safely.
18. Game integration interfaces are available.
19. Installation and configuration are documented.
20. The application builds and runs without placeholder implementations in completed phases.

---

# 24. REQUIRED DOCUMENTATION

Generate:

```text
README.md
docs/ARCHITECTURE.md
docs/INSTALLATION.md
docs/CONFIGURATION.md
docs/DISCORD_COMMANDS.md
docs/SOLANA_INTEGRATION.md
docs/WALLET_VERIFICATION.md
docs/DATABASE_SCHEMA.md
docs/SECURITY.md
docs/TESTING.md
docs/POW_INTEGRATION.md
```

Include example configuration files without secrets.

Provide instructions for configuring a DAS-compatible Solana provider.

Document all environment variables.

---

# 25. CLAUDE CODE EXECUTION INSTRUCTIONS

Follow this development process:

### Step 1: Inspect

Analyze the existing repository.

Identify reusable components and potential compatibility problems.

Do not modify files yet.

### Step 2: Plan

Create an implementation plan.

Identify dependencies, database changes, security requirements, and integration points.

Document assumptions.

### Step 3: Implement

Implement each phase incrementally.

Prefer complete vertical slices over large collections of unfinished classes.

Do not introduce unnecessary dependencies.

### Step 4: Test

After each phase:

- Build the solution.
- Execute automated tests.
- Review compiler warnings.
- Validate dependency injection.
- Validate database migrations.
- Review security-sensitive code.

### Step 5: Integrate

Register new Discord commands.

Verify existing commands remain functional.

Test against configured blockchain services when credentials are available.

### Step 6: Document

Update technical documentation.

Provide configuration instructions.

Document any unsupported features or remaining limitations.

### Step 7: Report

At completion, provide:

```text
IMPLEMENTATION SUMMARY

Completed Features:
Files Created:
Files Modified:
Database Migrations:
Discord Commands:
External Dependencies:
Configuration Required:
Tests Executed:
Test Results:
Known Limitations:
Security Considerations:
Next Recommended Phase:
```

Do not claim a feature is complete unless its implementation and relevant tests are present.

Do not claim live blockchain verification was tested if no provider credentials were available.

---

# 26. FINAL ENGINEERING PRINCIPLES

The FORGOTTEN NFT engine must be:

**Universal:** Compatible with multiple Solana NFT collections.

**Modular:** Independent of any specific Discord game.

**Secure:** Cryptographic ownership verification.

**Extensible:** Support future blockchain adapters.

**Persistent:** SQLite-backed records and history.

**Interactive:** Native Discord galleries and embeds.

**Efficient:** Asynchronous operations and bounded caching.

**Reliable:** Explicit handling of provider failures.

**Auditable:** Verification and ownership records.

**Maintainable:** Clean architecture and automated tests.

The system must provide a reusable foundation for NFT-enabled Discord applications.

The core NFT engine must not contain hardcoded Dead Mint Carnival rules.

Dead Mint Carnival and POW integrations must consume the NFT engine through clearly defined application interfaces.

## FINAL DIRECTIVE

Begin by inspecting the existing FORGOTTEN repository.

Produce an architectural assessment and phased implementation plan.

Then implement Phase 1 completely, including tests and documentation.

Proceed through subsequent phases only after the previous phase builds and passes its required tests.

If external credentials, domain configuration, or deployment access are unavailable, implement and test everything possible locally, document the missing prerequisite, and do not fabricate successful integration results.

Prioritize working, maintainable software over unnecessary complexity.

### Implementation notes

Two technical references are especially useful for Claude Code: the [Discord.Net interaction framework](https://docs.discordnet.dev/guides/int_framework/intro?utm_source=chatgpt.com) for slash commands and interactive components, and the [Metaplex DAS documentation](https://www.metaplex.com/docs/dev-tools/das-api?utm_source=chatgpt.com) for Solana asset retrieval.

**One architectural decision I'd strongly recommend:** Keep the NFT engine separate from the POW game. That way FORGOTTEN can validate and display NFTs from any compatible Solana collection, while Dead Mint Carnival can add its own fighter statistics, magic attacks, progression, and ownership rules.

This also leaves the door open for a future standalone NFT verification service that other Discord servers could use.