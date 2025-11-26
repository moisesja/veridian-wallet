# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Veridian Wallet is an open-source mobile application (iOS/Android) developed by the Cardano Foundation that implements Self-Sovereign Identity (SSI) using KERI (Key Event Receipt Infrastructure) on Cardano. Built with React, Ionic, and Capacitor, it provides secure identity management with native biometrics and hardware-backed encryption.

**Tech Stack:**
- Frontend: React 18, TypeScript, Ionic 8, Capacitor 7
- State Management: Redux Toolkit
- Storage: SQLite (encrypted), Ionic Storage, Secure Storage (SE/TEE)
- Identity: KERI via Signify-TS, KERIA cloud agent
- Testing: Jest (unit), WebdriverIO + Appium (E2E)
- Build: Webpack 5

## Development Commands

### Setup
```bash
git clone https://github.com/cardano-foundation/veridian-wallet.git
cd veridian-wallet
make init                 # Configure git hooks (required before first commit)
npm install              # Install dependencies
```

### Running the Application
```bash
# Browser (requires local services)
docker compose up -d --build    # Start KERIA and credential server
npm run dev                      # Dev server at http://localhost:3003/

# Build for production
npm run build                    # Remote environment
npm run build:local              # Local environment
npm run build:release            # Production with Capacitor sync
npm run build:cap                # Build and sync to native platforms
npm run build:e2e                # Local build for E2E tests
```

### Code Quality
```bash
npm run prettier         # Format code
npm run eslint          # Lint TypeScript files
npm test                # Run Jest unit tests with coverage
npm run audit           # Security audit with better-npm-audit
```

### Testing
```bash
# Unit tests (Jest)
npm test                                    # Run all tests with coverage (80% statement/line threshold)
npm test -- path/to/test-file.test.ts      # Run specific test file
npm test -- --watch                         # Run tests in watch mode
npm test -- --coverage --collectCoverageFrom='src/path/**/*.ts'  # Targeted coverage

# E2E tests (WebdriverIO + Appium)
# Prerequisites: Create .env with APP_PATH and KERIA_IP
npm run wdio:android:s24ultra              # Full Android test suite
npm run wdio:ios:15promax                   # Full iOS test suite (iPhone 15 Pro Max)
npm run wdio:ios:16promax                   # Full iOS test suite (iPhone 16 Pro Max)

# Run specific feature or scenario
npm run wdio:ios:15promax -- --spec ./tests/features/passcode.feature
npm run wdio:ios:15promax -- --spec ./tests/features/passcode.feature:18  # Line number

# Generate and view Allure reports
allure generate tests/.reports/allure-results -o tests/.reports/allure-report --clean
allure open tests/.reports/allure-report
```

### Native Platform Builds
```bash
# iOS-specific
npm run build:ios:flags  # Copy xcconfig overwrites to Pods

# Android-specific
npm run build:android:proguard  # Copy ProGuard rules

# Capacitor sync
npx cap sync            # Sync web assets to native platforms
npx cap open ios        # Open in Xcode
npx cap open android    # Open in Android Studio
```

## Architecture Overview

### Core Layer (`src/core/`)

**Agent System** (`src/core/agent/`):
- Central SSI agent implementation coordinating all identity operations
- **Services** (`services/`): Pluggable service layer interfacing between UI and agent
  - `identifierService`: KERI identifier creation/management (single-sig, multi-sig)
  - `credentialService`: ACDC credential lifecycle (issuance, presentation, revocation)
  - `connectionService`: Peer connection management via OOBI
  - `multiSigService`: Group multi-sig operations and member coordination
  - `ipexCommunicationService`: IPEX protocol for credential exchange
  - `keriaNotificationService`: Long-polling notification handling from KERIA
  - `authService`: Authentication and passcode management
- **Records** (`records/`): Data model layer with storage abstractions
  - When creating new record types, set default values in the constructor's `if (props)` block (not at attribute level) due to database serialization
  - Base classes: `BasicRecord`, `ConnectionRecord`, `CredentialMetadataRecord`, `IdentifierMetadataRecord`, etc.
  - Storage classes follow pattern: `*Storage` (e.g., `IdentifierStorage`, `CredentialStorage`)

**Storage** (`src/core/storage/`):
- Three-tier storage strategy:
  - `sqliteStorage/`: Encrypted SQLite (SQLCipher) for sensitive data - primary persistent storage
  - `secureStorage/`: Hardware-backed Secure Enclave/TEE for keys and seeds
  - `ionicStorage/`: IndexedDB wrapper for non-sensitive ephemeral data
- **Migrations** (`sqliteStorage/migrations/`): Database schema evolution
  - Method 1: Direct SQL in `*_sql.ts` files (initial setup)
  - Method 2: TypeScript functions returning SQL + parameters
  - Applied automatically on app start

**Cardano Integration** (`src/core/cardano/`):
- dApp connector using CIP-45 protocol
- Peer-to-peer communication with Cardano dApps

### State Management (`src/store/`)

Redux Toolkit slices managing cached application state:
- `ssiAgent`: Agent connection state and configuration
- `identifiersCache`: KERI identifiers (AIDs)
- `credsCache`/`credsArchivedCache`: Credentials (active and archived)
- `connectionsCache`: Peer connections
- `walletConnectionsCache`: dApp wallet connections
- `notificationsCache`: KERIA notifications
- `stateCache`: Global app state (authentication, onboarding, etc.)
- `seedPhraseCache`: Temporary seed phrase during onboarding
- `biometricsCache`: Biometric authentication state
- `viewTypeCache`: UI view preferences

### UI Layer (`src/ui/`)

- **Pages** (`pages/`): Full-screen Ionic page components mapped to routes
- **Components** (`components/`): Reusable UI components
- **Hooks** (`hooks/`): Custom React hooks for common patterns
- **Utils** (`utils/`): UI-specific utilities
- Styling: SCSS with global styles in `styles/`

### Routing (`src/routes/`)

- React Router v5 with Ionic navigation integration
- Path definitions in `paths.ts`
- Navigation helpers: `nextRoute/`, `backRoute/`

### Services (`services/`)

Separate Node.js services for development/testing:
- **credential-server**: Mock credential issuance server (testing only, no production use)
- **credential-server-ui**: Web UI for credential issuance testing
- **cip45-sample-dapp**: Sample dApp demonstrating CIP-45 integration

### Testing (`tests/`)

E2E test structure:
- `features/`: Gherkin feature files (BDD scenarios)
- `steps-definitions/`: Step implementations
- `screen-objects/`: Page Object Model for UI interactions
- `config/`: WebdriverIO configurations per device
- `helpers/`: Test utilities

## Important Development Notes

### Git Commit Convention
- **REQUIRED**: Use Conventional Commits specification for all commits
- Run `make init` before first commit to configure git hooks
- Format: `<type>(<scope>): <description>` (e.g., `feat(identifiers): add multi-sig support`)
- Automated CHANGELOG generation depends on this

### Testing Requirements
- Unit test coverage thresholds: 80% statements/lines, 50% branches/functions
- Coverage ignores: `/node_modules`, `src/routes/index.tsx`, `/e2e`, `src/core/cardano`
- E2E tests require `.env` file with `APP_PATH` and `KERIA_IP` variables

### Environment Configuration
- Environment variables control build targets: `local`, `remote`, `prod`
- See `.env.example` for all configurable values
- **Never commit** `.env` files with secrets
- Production builds use `.env.production.local` or `.env.production.traefik`
- Key environment variables:
  - `DEV_SKIP_ONBOARDING`: Skip onboarding flow in development (set to `true`)
  - `APP_PATH`: Path to built app for E2E tests (Android `.apk` or iOS `.app`)
  - `KERIA_IP`: KERIA server IP for simulator/emulator network tunneling
  - `APP_CERT_HASH`, `APP_WATCHER_MAIL`, `APP_TEAM_ID`: Required for native builds

### Security Considerations
- Project has undergone security auditing and penetration testing
- Secure Enclave/TEE integration for cryptographic material
- SQLCipher encryption at rest
- Privacy screen and screenshot prevention on mobile
- Biometric authentication support

### Native Platform Notes
- **iOS**: Requires Xcode, Podfile management, xcconfig overwrites for build flags
- **Android**: Requires Android Studio, ProGuard rules must be copied before builds
- Capacitor plugins may require manual native configuration updates
- Test on physical devices for biometrics and secure storage

### KERI/SSI Specifics
- Application communicates with KERIA cloud agent for key event processing
- CESR encoding for efficient over-the-wire data
- Witnesses and Cardano provide dual backing for identifiers
- IPEX protocol for credential exchange flows
- Long-polling for real-time notifications from KERIA

### Known Patterns
- Agent services use dependency injection via `AgentServicesProps`
- Storage operations use session pattern (`SqliteSession`, `IonicSession`)
- Event emitter (`CoreEventEmitter`) for cross-cutting agent events
- `@OnlineOnly` decorator for operations requiring network connectivity
- Network error handling with `isNetworkError()` utility
- Jest configuration: Uses `ts-jest` for TypeScript, special transform for Ionic/Capacitor modules
- Test mocks: Canvas mocks and Swiper mocks in setup files

### Docker Compose Services
The `docker-compose.yaml` provides local development infrastructure:
- KERIA agent (port 3901)
- Credential issuance server (port 3010)
- Credential server UI (port 3011)
- CIP-45 sample dApp (port 3012)
- Required for browser-based development (`npm run dev`)

## Documentation Links
- Project docs: https://docs.veridian.id/
- KERI resources: https://keri.one/
- Capacitor setup: https://capacitorjs.com/docs/getting-started/environment-setup
- Testing guide: [docs/Testing.md](docs/Testing.md)
- Emulator guide: [docs/Running-in-an-Emulator.md](docs/Running-in-an-Emulator.md)
