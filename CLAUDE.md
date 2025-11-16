# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Qubestr is a specialized Nostr relay built on the Khatru framework for the HyperQube Network. It handles only two custom event kinds:
- **Kind 33321 (HyperSignal)**: Commands broadcast by HC1 developers to HyperQube nodes (upgrades, reboots)
- **Kind 3333 (QubeManager)**: Status reports from HyperQube nodes acknowledging command execution

The relay enforces NIP-42 authentication for all operations and uses PostgreSQL for persistent event storage.

## Development Commands

### Local Development Setup
```bash
# Install dependencies
go mod tidy

# Start PostgreSQL database only (for local development)
docker-compose up -d db

# Run the relay locally
go run ./cmd/qubestr
```

### Docker Development
```bash
# Build and run both relay and database
docker-compose up --build -d

# View logs
docker-compose logs -f qubestr

# Stop services
docker-compose down

# Stop services and remove volumes (database reset)
docker-compose down -v
```

### Testing
There are no automated tests in this repository yet. Manual testing requires:
1. A Nostr client that supports NIP-42 authentication
2. Authorized pubkeys configured in `.env` for publishing Kind 33321 events
3. Valid event payloads matching the specifications in `hyperqube-events.md`

## Code Architecture

### Entry Point (`cmd/qubestr/main.go`)
- Initializes the Khatru relay instance
- Configures PostgreSQL backend via `eventstore/postgresql`
- Sets up NIP-42 authentication (mandatory on connect)
- Registers validation handlers for event rejection
- Serves HTTP endpoint at root (`/`) with relay information
- Listens on configurable host/port (default: `0.0.0.0:3334`)

### Event Validation Pipeline (`internal/handlers/validation.go`)
The relay uses a rejection-based validation approach with three main validators:

1. **ValidateKind**: Rejects any event that is not Kind 33321 or 3333
2. **ValidateHyperSignalEvent**: For Kind 33321 events:
   - Requires authenticated pubkey to be in `AUTHORIZED_PUBKEYS` env var
   - Enforces `["d", "hyperqube"]` tag (parameterized replaceable event)
   - Requires tags: `version`, `hash`, `network`, `action`
   - `action` must be "upgrade" or "reboot"
   - "reboot" action requires additional `genesis_url` and `required_by` tags
   - Content field must be non-empty

3. **ValidateQubeManagerEvent**: For Kind 3333 events:
   - Requires NIP-42 authentication
   - Requires tags: `a`, `version`, `network`, `action`, `status`, `node_id`, `action_at`
   - `a` tag must match format: `33321:<64_hex_pubkey>:hyperqube`
   - `action` must be "upgrade" or "reboot"
   - `status` must be "success" or "failure"
   - "failure" status requires `error` tag
   - Content field must be non-empty

4. **RequireAuth**: Allows all filter queries (both authenticated and unauthenticated). Logs unauthenticated read requests for security monitoring.

### PostgreSQL Integration
The relay uses the Khatru framework's PostgreSQL backend:
- Database connection built from environment variables (DB_HOST, DB_PORT, etc.)
- Hooks into Khatru's event lifecycle: StoreEvent, QueryEvents, DeleteEvent, ReplaceEvent, CountEvents
- Configurable query limits and recent event caching

### Environment Configuration
All configuration via `.env` file (see `.env.example`):
- **Database**: Connection parameters for PostgreSQL
- **Relay**: PORT, HOST for network binding
- **Auth**: RELAY_ADMIN_PUBKEY for NIP-11, AUTHORIZED_PUBKEYS for Kind 33321 publishing
- **Performance**: DB_QUERY_LIMIT, DB_KEEP_RECENT_EVENTS

## Key Implementation Details

### Addressable Events (Kind 33321)
Kind 33321 is a parameterized replaceable event. The relay stores only the latest event per pubkey with `["d", "hyperqube"]`. This creates a unique "address" (`33321:<pubkey>:hyperqube`) that always points to the most recent command from each authorized developer.

### Regular Events (Kind 3333)
Kind 3333 events are regular events, so the relay preserves all acknowledgments from nodes. This creates an auditable history of node responses to commands.

### Authentication Flow
1. Client connects via WebSocket
2. Relay immediately sends AUTH challenge (NIP-42)
3. **Reading events:** No authentication required (unauthenticated reads are logged)
4. **Writing events:** Authentication required for both event kinds
   - Kind 33321 publishing requires pubkey to be in AUTHORIZED_PUBKEYS whitelist
   - Kind 3333 publishing requires authentication but no whitelist

## Important Files

- `hyperqube-events.md`: Complete specification of custom event kinds, required tags, and validation rules
- `.env.example`: All configurable environment variables with defaults
- `cmd/qubestr/main.go`: Application entry point and relay setup
- `internal/handlers/validation.go`: All business logic for event validation

## Development Notes

- The codebase uses Go 1.24.4 with modules
- Built on Khatru (relay framework) and go-nostr (protocol implementation)
- No make targets or scripts; use `go run` for local dev or `docker-compose` for containerized
- Logging occurs in validation.go for rejected events (includes event ID, kind, pubkey, rejection reason)
- The relay serves a simple HTML page at HTTP root (`/`) describing its purpose
