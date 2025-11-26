# Veridian Wallet - Architecture Guide

This document provides a comprehensive overview of the Veridian Wallet architecture, data flows, and system interactions.

## Table of Contents
- [System Overview](#system-overview)
- [Network Architecture](#network-architecture)
- [Application Architecture](#application-architecture)
- [Data Flow](#data-flow)
- [Storage Architecture](#storage-architecture)
- [Agent System Deep Dive](#agent-system-deep-dive)
- [Event System](#event-system)
- [Initialization Sequence](#initialization-sequence)

## System Overview

Veridian Wallet is a Self-Sovereign Identity (SSI) mobile wallet that implements KERI (Key Event Receipt Infrastructure) on the Cardano blockchain. The application follows a layered architecture with clear separation between:

1. **UI Layer** - React + Ionic components
2. **State Management** - Redux Toolkit for application state
3. **Core Agent** - SSI operations and business logic
4. **Storage Layer** - Three-tier storage strategy (SQLite, Secure Storage, Ionic Storage)
5. **External Services** - KERIA cloud agent, witnesses, credential servers

### Key Architectural Principles

- **Singleton Agent Pattern**: The `Agent` class is implemented as a singleton, ensuring a single point of coordination for all SSI operations
- **Event-Driven Architecture**: Core events flow through `CoreEventEmitter` to decouple agent operations from UI
- **Offline-First**: Application supports offline mode with intelligent sync when connectivity returns
- **Platform Abstraction**: Storage and security features abstract native platform differences (iOS/Android)

## Network Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     Veridian Wallet App                      │
│  ┌────────────┐  ┌─────────────┐  ┌──────────────────────┐  │
│  │ UI Layer   │──│ Agent Core  │──│ Storage (SQLite/SE)  │  │
│  │ (React)    │  │ (Signify-TS)│  │                      │  │
│  └────────────┘  └─────────────┘  └──────────────────────┘  │
└────────────┬─────────────────────────────────────────────────┘
             │
             ├──────────► KERIA Cloud Agent (port 3901-3903)
             │            • Key event processing
             │            • Identifier management
             │            • Notification long-polling
             │
             ├──────────► KERI Witnesses (ports 5642-5647)
             │            • 6 witness nodes for key event validation
             │            • Provides redundancy and availability
             │
             ├──────────► Cardano Blockchain
             │            • Dual backing for identifiers
             │            • Immutable event log
             │
             ├──────────► Credential Issuance Server (port 3001)
             │            • ACDC credential issuance
             │            • IPEX protocol implementation
             │
             └──────────► dApps (via CIP-45)
                          • Cardano dApp connector
                          • Peer-to-peer communication
```

### Docker Compose Services

The local development environment (`docker-compose.yaml`) provides:

1. **KERIA** (`idw-keria`) - Port 3901-3903
   - Main agent service for identity operations
   - Handles key events, identifiers, credentials
   - Configured with CORS enabled for browser development
   - Stores data in `keria-data` volume
   - Configuration: `keria-config/config.json`

2. **Witnesses** (`idw-witnesses`) - Ports 5642-5647
   - 6 KERI witness nodes running in demo mode
   - Validates and witnesses key events
   - Essential for KERI security model

3. **Credential Issuance Server** (`cred-issuance`) - Port 3001
   - Mock server for testing credential flows
   - **Testing only** - not for production use
   - Communicates with KERIA at `http://keria:3901`

4. **Credential Server UI** (`cred-issuance-ui`) - Port 3000
   - Web interface for issuing test credentials
   - Useful for development and testing

## Application Architecture

### Layer Breakdown

```
┌─────────────────────────────────────────────────────────────┐
│                         UI Layer                            │
│  src/ui/                                                    │
│  ├── pages/        - Full-screen Ionic pages               │
│  ├── components/   - Reusable React components             │
│  ├── hooks/        - Custom React hooks                    │
│  └── App.tsx       - Root component with initialization    │
└────────────────────────┬────────────────────────────────────┘
                         │ dispatch actions, selectors
┌────────────────────────▼────────────────────────────────────┐
│                   State Management Layer                    │
│  src/store/                                                 │
│  ├── identifiersCache    - KERI identifiers (AIDs)         │
│  ├── credsCache           - Active credentials             │
│  ├── connectionsCache     - Peer connections               │
│  ├── notificationsCache   - KERIA notifications            │
│  ├── stateCache           - Global app state               │
│  └── biometricsCache      - Biometric auth state           │
└────────────────────────┬────────────────────────────────────┘
                         │ Agent API calls
┌────────────────────────▼────────────────────────────────────┐
│                      Agent Core Layer                       │
│  src/core/agent/                                            │
│  ├── agent.ts       - Singleton coordinator                │
│  ├── services/      - Business logic services              │
│  │   ├── identifierService      - AID management           │
│  │   ├── credentialService      - ACDC lifecycle           │
│  │   ├── connectionService      - Peer connections         │
│  │   ├── multiSigService        - Multi-sig operations     │
│  │   ├── ipexCommunicationService - Credential exchange    │
│  │   ├── keriaNotificationService - Long-polling           │
│  │   └── authService            - Authentication           │
│  ├── records/       - Data models & storage abstractions   │
│  └── event.ts       - Event emitter                        │
└────────────────────────┬────────────────────────────────────┘
                         │ Storage operations
┌────────────────────────▼────────────────────────────────────┐
│                     Storage Layer                           │
│  src/core/storage/                                          │
│  ├── sqliteStorage/   - Encrypted SQLite (SQLCipher)       │
│  │   ├── migrations/  - Schema evolution                   │
│  │   └── sqliteSession.ts                                  │
│  ├── secureStorage/   - Hardware SE/TEE (keys, seeds)      │
│  └── ionicStorage/    - IndexedDB (non-sensitive data)     │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│                  External Services                          │
│  • KERIA (Signify-TS client)                               │
│  • Witnesses (KERI validation)                             │
│  • Cardano (blockchain backing)                            │
│  • Credential Servers (ACDC issuance)                      │
└─────────────────────────────────────────────────────────────┘
```

## Data Flow

### Example: Creating a New Identifier

```
1. User Action (UI)
   └─► pages/CreateIdentifier → dispatch(createIdentifier())

2. Redux Action
   └─► Thunk middleware → Agent.agent.identifiers.createIdentifier()

3. Agent Core
   └─► identifierService.createIdentifier()
       ├─► signifyClient.identifiers().create() → KERIA
       ├─► identifierStorage.save() → SQLite
       └─► eventEmitter.emit(IdentifierAdded)

4. Event Propagation
   └─► AppWrapper event handler
       └─► dispatch(setIdentifiersCache())

5. UI Update
   └─► React re-render with new identifier
```

### Example: Receiving a Credential (IPEX Flow)

```
1. Background Polling
   └─► keriaNotificationService.pollNotifications() (every N seconds)

2. Notification Detected
   └─► KERIA returns new notification
       └─► Process notification type

3. IPEX Grant Notification
   └─► ipexCommunicationService.processCredentialNotification()
       ├─► Fetch credential details from KERIA
       ├─► credentialStorage.save() → SQLite
       └─► eventEmitter.emit(AcdcStateChanged)

4. Event Handler (AppWrapper)
   └─► acdcChangeHandler()
       ├─► dispatch(updateOrAddCredsCache())
       └─► dispatch(setToastMsg(NEW_CREDENTIAL_ADDED))

5. UI Updates
   └─► Credentials page refreshes
   └─► Toast notification appears
```

## Storage Architecture

### Three-Tier Storage Strategy

#### 1. SQLite Storage (Primary Persistent Storage)
- **Location**: `sqliteStorage/`
- **Implementation**: SQLCipher for encryption at rest
- **Platform**: Native (iOS/Android) only
- **Session**: `SqliteSession`
- **Use Cases**:
  - Identifier metadata
  - Credential metadata
  - Connection records
  - Notification history
  - Pending operations

**Key Classes**:
- `SqliteStorage<T>` - Generic storage implementation
- `IdentifierStorage`, `CredentialStorage`, `ConnectionStorage` - Domain-specific storage
- `BasicStorage` - Key-value store for miscellaneous data

**Migrations**:
- Located in `sqliteStorage/migrations/`
- Two formats supported:
  1. Direct SQL in `*_sql.ts` files (initial setup)
  2. TypeScript functions returning `{ sql: string, params: any[] }`
- Applied automatically on app start in `sqliteSession.open()`

#### 2. Secure Storage (Hardware-Backed)
- **Location**: `secureStorage/`
- **Implementation**:
  - iOS: Secure Enclave
  - Android: Keystore with TEE
- **Platform**: Native only
- **Use Cases**:
  - App passcode
  - Signify BRAN (seed)
  - Optional password for sensitive operations
  - Biometric authentication tokens

**Key Methods**:
```typescript
await SecureStorage.set(KeyStoreKeys.APP_PASSCODE, passcode);
await SecureStorage.get(KeyStoreKeys.SIGNIFY_BRAN);
await SecureStorage.keyExists(KeyStoreKeys.APP_PASSCODE);
await SecureStorage.wipe(); // Delete all keys
```

#### 3. Ionic Storage (Browser-Compatible)
- **Location**: `ionicStorage/`
- **Implementation**: IndexedDB wrapper
- **Platform**: Browser development
- **Session**: `IonicSession`
- **Use Cases**:
  - Same structure as SQLite but unencrypted
  - Development and browser testing only

### Platform-Specific Storage Selection

```typescript
// agent.ts:206-209
private constructor() {
  this.storageSession = Capacitor.isNativePlatform()
    ? new SqliteSession()
    : new IonicSession();
}
```

### Storage Records Pattern

All record types inherit from `BasicRecord` and follow this pattern:

```typescript
class IdentifierMetadataRecord extends BasicRecord {
  constructor(props?: IdentifierMetadataRecordProps) {
    super();
    if (props) {
      // Set default values HERE (not at attribute level)
      // This is critical for database serialization
      this.id = props.id;
      this.displayName = props.displayName;
      this.createdAtUTC = props.createdAtUTC;
      // ... other properties
    }
  }
}
```

**Important**: Default values must be set in the `if (props)` block, not at the attribute declaration level. This is due to how the database serialization works.

## Agent System Deep Dive

### Agent Singleton (`src/core/agent/agent.ts`)

The `Agent` class is the central coordinator for all SSI operations. It:

1. **Manages Signify Client**: Initializes and maintains connection to KERIA
2. **Coordinates Services**: Lazy-loads and provides access to all services
3. **Handles Lifecycle**: Boot, connect, recover, delete account
4. **Online/Offline Management**: Tracks connection state and triggers sync

### Key Agent Methods

#### Initialization Flow

```typescript
// 1. Setup local dependencies (storage, event emitter)
await Agent.agent.setupLocalDependencies();

// 2. Boot KERIA agent (first time)
await Agent.agent.bootAndConnect({
  url: "http://keria:3901",
  bootUrl: "http://keria:3903"
});

// 3. Start with existing connection
await Agent.agent.start("http://keria:3901");

// 4. Recover from seed phrase
await Agent.agent.recoverKeriaAgent(seedPhrase, connectUrl);
```

#### Service Getters (Lazy Loading)

```typescript
// Services are initialized on first access
const identifiers = Agent.agent.identifiers; // IdentifierService
const credentials = Agent.agent.credentials; // CredentialService
const connections = Agent.agent.connections; // ConnectionService
const multiSigs = Agent.agent.multiSigs;     // MultiSigService
```

### Agent Services

#### 1. IdentifierService (`identifierService.ts`)
**Purpose**: Create and manage KERI Autonomic Identifiers (AIDs)

**Key Operations**:
- `createIdentifier()` - Create single-sig or multi-sig identifier
- `getIdentifiers()` - Fetch all identifiers from storage
- `syncKeriaIdentifiers()` - Sync from KERIA to local DB
- `removeIdentifiersPendingDeletion()` - Cleanup after online
- `getAvailableWitnesses()` - Check witness availability

**Multi-sig Support**: Can create group multi-sig identifiers with threshold signatures

#### 2. CredentialService (`credentialService.ts`)
**Purpose**: Manage ACDC (Authentic Chained Data Container) credentials

**Key Operations**:
- `getCredentials()` - Fetch active or archived credentials
- `syncKeriaCredentials()` - Sync from KERIA
- `archiveCredential()` / `restoreCredential()` - Archive management
- `removeCredentialsPendingDeletion()` - Cleanup

**Credential Lifecycle**: PENDING → CONFIRMED → (optionally) REVOKED or ARCHIVED

#### 3. ConnectionService (`connectionService.ts`)
**Purpose**: Manage peer-to-peer connections via OOBI (Out-Of-Band Introduction)

**Key Operations**:
- `connectByOobiUrl()` - Establish connection via OOBI URL
- `getConnections()` - Fetch connection list
- `syncKeriaContacts()` - Sync contacts from KERIA
- `resolvePendingConnections()` - Process connections pending confirmation

**Connection Types**:
- Regular connections (peer-to-peer)
- Multi-sig connections (group member connections)

#### 4. MultiSigService (`multiSigService.ts`)
**Purpose**: Coordinate group multi-signature operations

**Key Operations**:
- `createMultisigIdentifier()` - Initiate multi-sig group
- `joinMultisigGroup()` - Join as member
- `processGroupsPendingCreation()` - Complete multi-sig setup
- Multi-member coordination via KERIA

**Threshold Signatures**: Supports m-of-n signature schemes

#### 5. IpexCommunicationService (`ipexCommunicationService.ts`)
**Purpose**: Handle IPEX (Issuance and Presentation Exchange) protocol

**Key Operations**:
- `processCredentialNotification()` - Handle incoming credential offers
- `grantAcdcFromApply()` - Accept credential issuance
- `offerAcdcFromApply()` - Offer credential to others

**IPEX Flow**: Apply → Offer → Grant → Credential issued

#### 6. KeriaNotificationService (`keriaNotificationService.ts`)
**Purpose**: Long-polling for real-time KERIA notifications

**Key Features**:
- Polls KERIA every N seconds (configurable interval)
- Processes notifications: credentials, multi-sig invites, connection requests
- Manages long-running operation status
- Automatic reconnection on failure

**Polling Methods**:
```typescript
keriaNotifications.startPolling();  // User logged in
keriaNotifications.stopPolling();   // User logged out
keriaNotifications.pollNotifications(); // Manual poll
```

#### 7. AuthService (`authService.ts`)
**Purpose**: Authentication and passcode management

**Key Operations**:
- `storeSecret()` - Store sensitive data in Secure Storage
- `getLoginAttempts()` - Track failed login attempts
- `incrementLoginAttempt()` - Failed login tracking

### Service Dependency Injection

All services receive `AgentServicesProps`:

```typescript
interface AgentServicesProps {
  signifyClient: SignifyClient;  // KERIA connection
  eventEmitter: CoreEventEmitter; // Cross-cutting events
}
```

Services can also depend on:
- Storage instances (IdentifierStorage, CredentialStorage, etc.)
- Other services (passed in constructor)

### @OnlineOnly Decorator

Operations requiring network connectivity use the `@OnlineOnly` decorator:

```typescript
@OnlineOnly
async deleteAccount() {
  // Only executes if Agent.isOnline === true
  // Throws error if offline
}
```

## Event System

### CoreEventEmitter (`src/core/agent/event.ts`)

The event emitter enables loose coupling between agent operations and UI updates.

### Key Event Types (`event.types.ts`)

```typescript
enum EventTypes {
  KeriaStatusChanged,        // Online/offline state
  IdentifierAdded,           // New identifier created
  IdentifierRemoved,         // Identifier deleted
  ConnectionStateChanged,    // Connection status update
  AcdcStateChanged,          // Credential status update
  MultiSigMemberAdded,       // Multi-sig group member added
  NotificationAdded,         // New notification
  NotificationRemoved,       // Notification dismissed
  OperationComplete,         // Long operation succeeded
  OperationFailure,          // Long operation failed
}
```

### Event Flow Example

```typescript
// 1. Service emits event
this.eventEmitter.emit<IdentifierAddedEvent>({
  type: EventTypes.IdentifierAdded,
  payload: { identifier: newIdentifier }
});

// 2. AppWrapper subscribes to event
Agent.agent.identifiers.onIdentifierAdded((event) => {
  identifierAddedHandler(event, dispatch);
});

// 3. Handler updates Redux
function identifierAddedHandler(event, dispatch) {
  dispatch(updateOrAddIdentifierCache(event.payload.identifier));
  dispatch(setToastMsg(ToastMsgType.IDENTIFIER_CREATED));
}
```

### Event Subscription Setup

All event subscriptions are registered in `AppWrapper.tsx` → `setupEventServiceCallbacks()`:

```typescript
// Online/offline status
Agent.agent.onKeriaStatusStateChanged((event) => {
  setOnlineStatus(event.payload.isOnline);
});

// Connection changes
Agent.agent.connections.onConnectionStateChanged((event) => {
  connectionStateChangedHandler(event, dispatch);
});

// Credential changes
Agent.agent.credentials.onAcdcStateChanged((event) => {
  acdcChangeHandler(event, dispatch);
});

// Notifications
Agent.agent.keriaNotifications.onNewNotification((event) => {
  notificationStateChanged(event, dispatch);
});
```

## Initialization Sequence

### App Startup Flow

```
┌─────────────────────────────────────────────────────────────┐
│ 1. App.tsx Renders                                          │
│    ├─► Check system compatibility (OS version, keystore)   │
│    ├─► Initialize FreeRASP (security monitoring)           │
│    └─► Render AppWrapper                                    │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 2. AppWrapper.initApp()                                     │
│    ├─► Agent.agent.setupLocalDependencies()                │
│    │   ├─► Open storage session (SQLite or Ionic)          │
│    │   ├─► Initialize storage services                     │
│    │   └─► Setup event emitter                             │
│    │                                                         │
│    ├─► Check initialization state                          │
│    │   └─► If first install: wipe secure storage           │
│    │                                                         │
│    ├─► DEV_SKIP_ONBOARDING check                           │
│    │   └─► devPreload() if enabled                         │
│    │                                                         │
│    ├─► loadCacheBasicStorage()                             │
│    │   ├─► Check passcode exists                           │
│    │   ├─► Check seed phrase exists                        │
│    │   ├─► Load KERIA connection URL                       │
│    │   ├─► Load user preferences                           │
│    │   └─► Update authentication state                     │
│    │                                                         │
│    └─► setupEventServiceCallbacks()                        │
│        └─► Register all event handlers                     │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 3. Set Initialization Phase                                 │
│    ├─► PHASE_ZERO: Initial loading                         │
│    ├─► PHASE_ONE: Show splash + lock page                  │
│    └─► PHASE_TWO: Main app ready                           │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 4. User Authentication (if configured)                      │
│    ├─► LockPage prompts for passcode/biometric             │
│    └─► On success: authentication.loggedIn = true          │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 5. Start Agent (if KERIA URL configured)                   │
│    ├─► Agent.agent.start(keriaConnectUrl)                  │
│    │   ├─► signifyReady() - Initialize Signify-TS         │
│    │   ├─► Get BRAN from Secure Storage                    │
│    │   ├─► Create SignifyClient                            │
│    │   └─► signifyClient.connect() → KERIA                 │
│    │                                                         │
│    └─► recoverAndLoadDb()                                   │
│        ├─► Check cloud recovery status                     │
│        ├─► syncWithKeria() if needed                       │
│        └─► loadDatabase()                                   │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 6. Load Database into Redux                                 │
│    ├─► connections.getConnections()                        │
│    ├─► identifiers.getIdentifiers()                        │
│    ├─► credentials.getCredentials()                        │
│    ├─► keriaNotifications.getNotifications()               │
│    └─► dispatch to Redux caches                            │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 7. Mark Agent Online & Start Polling                        │
│    ├─► Agent.agent.markAgentStatus(true)                   │
│    │   ├─► Process pending operations                      │
│    │   └─► Emit KeriaStatusChanged event                   │
│    │                                                         │
│    └─► keriaNotifications.startPolling()                   │
│        ├─► Poll notifications every N seconds              │
│        └─► Process incoming requests                        │
└─────────────────────────────────────────────────────────────┘
                     │
                     ▼
              App Ready for User
```

### Initialization Phases

The app uses three initialization phases to manage the startup sequence:

**PHASE_ZERO**:
- Initial loading
- Shows loading spinner only
- Storage not yet initialized

**PHASE_ONE**:
- Storage initialized
- Shows splash screen + lock page overlay
- Waiting for user authentication

**PHASE_TWO**:
- Fully initialized
- Agent connected to KERIA (or in offline mode)
- Main app navigation available
- Background polling active

### Offline Mode Handling

If KERIA connection fails during startup:

```typescript
// AppWrapper.tsx:308-318
try {
  await Agent.agent.start(authentication.ssiAgentUrl);
  await recoverAndLoadDb();
} catch (e) {
  if (e.message === Agent.KERIA_CONNECT_FAILED_BAD_NETWORK) {
    // Show offline mode immediately
    dispatch(setInitializationPhase(InitializationPhase.PHASE_TWO));

    // Background reconnection attempt
    Agent.agent.connect().then(() => recoverAndLoadDb());
  }
}
```

The app continues to function with local data, and automatically syncs when connectivity returns.

### Recovery Flow

If the user is recovering their wallet from a seed phrase:

```typescript
await Agent.agent.recoverKeriaAgent(seedPhrase, connectUrl);
// This triggers:
// 1. Derive BRAN from seed phrase
// 2. Connect to KERIA with recovered identity
// 3. Sync all data: identifiers, credentials, connections
// 4. Mark recovery complete
```

## Summary

Veridian Wallet's architecture is designed for:

- **Security**: Hardware-backed storage, encrypted databases, secure key management
- **Resilience**: Offline-first design, automatic reconnection, event-driven sync
- **Modularity**: Clear separation of concerns, dependency injection, platform abstraction
- **Developer Experience**: TypeScript types, comprehensive testing, well-documented patterns

The key to understanding the codebase is recognizing the flow:

1. **UI** triggers actions
2. **Redux** manages cached state
3. **Agent** coordinates SSI operations
4. **Services** implement business logic
5. **Storage** persists data
6. **Events** propagate changes back to UI

This architecture enables complex SSI operations while maintaining a responsive user experience and supporting both online and offline modes.
