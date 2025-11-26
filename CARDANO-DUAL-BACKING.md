# Cardano Dual Backing for KERI Identifiers

This document explains how Veridian Wallet implements "dual backing" for KERI identifiers using both traditional KERI witnesses and the Cardano blockchain.

## What is Dual Backing?

Dual backing means that KERI key events are validated and stored in two independent systems:

1. **KERI Witnesses** - Traditional KERI validation nodes
2. **Cardano Blockchain** - Immutable blockchain ledger via cardano-backer

This provides enhanced security, redundancy, and trust through multiple independent verification mechanisms.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      Veridian Wallet                            │
│                                                                 │
│  User creates identifier → Agent creates key event             │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   │ Key Event Log (KEL)
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                         KERIA Agent                             │
│                                                                 │
│  • Processes key events                                         │
│  • Coordinates witness receipts                                │
│  • Manages identifier state                                    │
└─────────┬───────────────────────────────────┬───────────────────┘
          │                                   │
          │ Send to witnesses                 │ Send to backer
          ▼                                   ▼
┌─────────────────────────┐    ┌────────────────────────────────┐
│   KERI Witnesses        │    │   Cardano Backer (Witness)     │
│   (6 nodes)             │    │                                │
│                         │    │  • Acts as KERI witness        │
│  • wit0 (port 5642)     │    │  • Anchors events to Cardano   │
│  • wit1 (port 5643)     │    │  • Provides blockchain proof   │
│  • wit2 (port 5644)     │    │  • Queues events for safety    │
│  • wit3 (port 5645)     │    │  • Waits for confirmations     │
│  • wit4 (port 5646)     │    │                                │
│  • wit5 (port 5647)     │    └────────────┬───────────────────┘
│                         │                 │
│  Validate & receipt     │                 │ Anchor transaction
│  key events             │                 ▼
└─────────────────────────┘    ┌────────────────────────────────┐
                               │      Cardano Blockchain        │
                               │                                │
                               │  • Immutable event log         │
                               │  • Block confirmations         │
                               │  • Public verification         │
                               │  • Network: preprod/mainnet    │
                               └────────────────────────────────┘
                                            │
                                            │ Query via Ogmios
                                            ▼
                               ┌────────────────────────────────┐
                               │         Ogmios                 │
                               │  (Cardano Node Interface)      │
                               │                                │
                               │  • WebSocket/HTTP API          │
                               │  • Query blockchain state      │
                               │  • Submit transactions         │
                               └────────────────────────────────┘
```

## How It Works

### 1. Identifier Creation Flow with Dual Backing

When you create an identifier in Veridian Wallet:

```typescript
// identifierService.ts:223-267
async createIdentifier(metadata) {
  // Step 1: Get available witnesses (including cardano-backer)
  const { toad, witnesses } = await this.getAvailableWitnesses();

  // Step 2: Create identifier with witnesses
  const result = await this.signifyClient.identifiers().create(name, {
    toad,              // Threshold of acceptable duplicity
    wits: witnesses.map((w) => w.eid),  // Witness IDs
  });

  // Key events are now sent to ALL witnesses including cardano-backer
}
```

### 2. Witness Configuration

The witnesses are configured in [keria-config/config.json](keria-config/config.json):

```json
{
  "iurls": [
    "http://witnesses:5642/oobi/BBilc4-.../controller?role=witness",
    "http://witnesses:5643/oobi/BLskRT.../controller?role=witness",
    "http://witnesses:5644/oobi/BIKKuv.../controller?role=witness",
    "http://witnesses:5645/oobi/BM35JN.../controller?role=witness",
    "http://witnesses:5646/oobi/BIj15u.../controller?role=witness",
    "http://witnesses:5647/oobi/BF2rZT.../controller?role=witness"
  ]
}
```

**Note**: In production ([docker-compose.production.cardano-witnesses.yaml](docker-compose.production.cardano-witnesses.yaml)), some or all of these witnesses are replaced with `cardano-backer` instances that anchor to Cardano.

### 3. Witness Discovery and Selection

The `getAvailableWitnesses()` method queries KERIA configuration:

```typescript
// identifierService.ts:749-787
async getAvailableWitnesses() {
  // Get configuration from KERIA
  const config = await this.signifyClient.config().get();

  if (!config.iurls) {
    throw new Error("MISCONFIGURED_AGENT_CONFIGURATION");
  }

  // Parse OOBIs to find witnesses
  const witnesses = [];
  for (const oobi of config.iurls) {
    const role = new URL(oobi).searchParams.get("role");
    if (role === "witness") {
      const eid = oobi.split("/oobi/")[1].split("/")[0];
      witnesses.push({ eid, oobi });
    }
  }

  // Determine threshold based on witness count
  if (witnesses.length >= 12) return { toad: 8, witnesses: witnesses.slice(0, 12) };
  if (witnesses.length >= 10) return { toad: 7, witnesses: witnesses.slice(0, 10) };
  if (witnesses.length >= 9)  return { toad: 6, witnesses: witnesses.slice(0, 9) };
  if (witnesses.length >= 7)  return { toad: 5, witnesses: witnesses.slice(0, 7) };
  if (witnesses.length >= 6)  return { toad: 4, witnesses: witnesses.slice(0, 6) };

  throw new Error("INSUFFICIENT_WITNESSES_AVAILABLE");
}
```

**TOAD (Threshold of Acceptable Duplicity)**: The minimum number of witness receipts required to consider a key event valid. This provides protection against compromised witnesses.

### 4. Cardano Backer Implementation

In production deployments, `cardano-backer` witnesses are configured in [docker-compose.production.cardano-witnesses.yaml](docker-compose.production.cardano-witnesses.yaml):

```yaml
x-cardano-backer: &witness-common
  image: ghcr.io/cardano-foundation/cardano-backer:main
  environment:
    - NETWORK=preprod  # or mainnet
    - OGMIOS_HOST=ogmios
    - OGMIOS_PORT=1337
    - START_SLOT_NUMBER=92593696
    - START_BLOCK_HEADER_HASH=345893332931ff4b...
  entrypoint: ["bash", "-c", "backer start --name wit$WITNESS_NO ..."]
```

Each backer:
1. Acts as a KERI witness (provides receipts)
2. Connects to Cardano via Ogmios
3. Anchors key events to the blockchain
4. Queues events for block confirmations (safety)

### 5. Cardano Infrastructure Stack

The production stack includes:

```yaml
# Cardano Node
cardano-node:
  image: cardano-node:10.4.1
  environment:
    - NETWORK=preprod
    - RESTORE_SNAPSHOT=true  # Uses Mithril for fast sync

# Ogmios (Cardano API)
ogmios:
  image: cardanosolutions/ogmios:v6.13.0
  command: ["--node-socket", "/ipc/node.socket"]

# Cardano Backers (6 instances)
witness-0 through witness-5:
  image: cardano-backer
  # Each connects to Ogmios to write to blockchain
```

## Key Benefits of Dual Backing

### 1. Enhanced Security
- **Multiple validation points**: Events validated by both KERI witnesses and blockchain
- **Redundancy**: If KERI witnesses fail, blockchain records remain
- **Tamper resistance**: Blockchain provides immutable proof

### 2. Trust Distribution
- **Decentralization**: No single point of failure
- **Public verifiability**: Anyone can verify events on Cardano blockchain
- **Witness diversity**: Mix of traditional witnesses and blockchain-backed witnesses

### 3. Resilience
- **Data availability**: Events stored in two independent systems
- **Recovery**: Can reconstruct state from blockchain if needed
- **Fault tolerance**: Threshold signatures (TOAD) protect against witness failures

## Verification Flow

When verifying an identifier's key event history:

```
1. Query KERIA Agent
   └─► Returns Key Event Log (KEL)

2. Verify with KERI Witnesses
   └─► Check witness receipts
   └─► Validate threshold (TOAD)

3. Verify on Cardano (optional, additional security)
   └─► Query Ogmios for blockchain transactions
   └─► Validate cardano-backer anchored events
   └─► Check block confirmations
```

## Configuration Files

### Development (Local)
- **File**: [docker-compose.yaml](docker-compose.yaml)
- **Witnesses**: 6 standard KERI witnesses (no blockchain)
- **Network**: Local development
- **Ports**: 5642-5647

### Production (Cardano-backed)
- **File**: [docker-compose.production.cardano-witnesses.yaml](docker-compose.production.cardano-witnesses.yaml)
- **Witnesses**: 6 cardano-backer instances
- **Network**: Cardano preprod or mainnet
- **Infrastructure**: Requires Cardano node + Ogmios

## Code References

### Identifier Creation with Witnesses
- [src/core/agent/services/identifierService.ts:218-337](src/core/agent/services/identifierService.ts#L218-L337) - `createIdentifier()` method
- [src/core/agent/services/identifierService.ts:749-787](src/core/agent/services/identifierService.ts#L749-L787) - `getAvailableWitnesses()` method

### Configuration
- [keria-config/config.json](keria-config/config.json) - Witness OOBIs for local development
- [docker-compose.yaml](docker-compose.yaml) - Local witness setup
- [docker-compose.production.cardano-witnesses.yaml](docker-compose.production.cardano-witnesses.yaml) - Production cardano-backer setup

### KERIA Docker Image
The KERIA image is configured to use the backer OOBI configuration:
```yaml
volumes:
  - ./keria-config/config.json:/keria/scripts/keri/cf/backer-oobis.json
entrypoint: keria start --config-file backer-oobis --config-dir ./scripts
```

## Summary

Veridian Wallet's "dual backing" architecture provides:

1. **Traditional KERI validation** through witness receipts
2. **Blockchain anchoring** via cardano-backer for immutable proof
3. **Threshold signatures (TOAD)** for fault tolerance
4. **Public verifiability** through Cardano blockchain

This hybrid approach combines the efficiency and flexibility of KERI with the immutability and public trust of blockchain technology, creating a robust foundation for Self-Sovereign Identity.

## External Resources

- [Cardano Backer GitHub](https://github.com/cardano-foundation/cardano-backer) - Source code and documentation
- [KERI Specification](https://keri.one/) - Key Event Receipt Infrastructure
- [Ogmios Documentation](https://ogmios.dev/) - Cardano WebSocket/HTTP API
- [Cardano Node](https://github.com/IntersectMBO/cardano-node) - Cardano blockchain node
