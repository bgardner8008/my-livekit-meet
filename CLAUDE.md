# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

LiveKit Meet is an open source video conferencing application built with Next.js 15, React 18, and the LiveKit Components library. It provides video conferencing capabilities with features like end-to-end encryption (E2EE), recording, and multi-region support.

## Development Commands

**Prerequisites:**
- Node.js >= 18
- pnpm 10.18.2 (specified in package.json)

```bash
# Install dependencies
pnpm install

# Development server
# Runs Next.js dev server on http://localhost:3000 with:
# - Hot module replacement (HMR) for instant updates
# - Fast Refresh for React components
# - Source maps enabled for debugging
# - TypeScript compilation on-demand
pnpm dev

# Production build
pnpm build

# Start production server
pnpm start

# Linting
pnpm lint
pnpm lint:fix

# Testing
pnpm test                                      # Run all tests with Vitest
pnpm vitest run lib/getLiveKitURL.test.ts      # Run specific test file

# Formatting
pnpm format:check
pnpm format:write
```

## Environment Setup

**Important:** This application does not run a LiveKit media server. It's a client application that connects to an existing LiveKit server instance (either LiveKit Cloud or a self-hosted server).

Copy `.env.example` to `.env.local` and configure:

**Required variables:**
- `LIVEKIT_API_KEY` - API key from LiveKit Cloud Dashboard (or your self-hosted server)
- `LIVEKIT_API_SECRET` - API secret from LiveKit Cloud Dashboard (or your self-hosted server)
- `LIVEKIT_URL` - WebSocket URL of your LiveKit server (e.g., `wss://my-livekit-project.livekit.cloud`)

**Optional variables:**
- S3 credentials for recording: `S3_KEY_ID`, `S3_KEY_SECRET`, `S3_ENDPOINT`, `S3_BUCKET`, `S3_REGION`
- `NEXT_PUBLIC_SHOW_SETTINGS_MENU=true` - Enables settings menu with Krisp noise filters
- `NEXT_PUBLIC_LK_RECORD_ENDPOINT=/api/record` - Recording API endpoint
- Datadog logging: `NEXT_PUBLIC_DATADOG_CLIENT_TOKEN`, `NEXT_PUBLIC_DATADOG_SITE`

## Architecture

### App Structure (Next.js 15 App Router)

```
app/
├── page.tsx                           # Home page with demo/custom tabs
├── rooms/[roomName]/                  # Dynamic room route
│   ├── page.tsx                       # Server component wrapper
│   └── PageClientImpl.tsx             # Client component with room logic
├── custom/                            # Custom connection flow
│   ├── page.tsx
│   └── VideoConferenceClientImpl.tsx
└── api/
    ├── connection-details/route.ts    # Generates participant tokens
    └── record/
        ├── start/route.ts             # Start room recording
        └── stop/route.ts              # Stop room recording
```

### Core Libraries and Components

**lib/** - Shared utilities and custom hooks:
- `client-utils.ts` - Client-side utilities (room ID generation, E2EE passphrase encoding, device detection)
- `types.ts` - TypeScript interfaces (ConnectionDetails, SessionProps, etc.)
- `useSetupE2EE.ts` - E2EE worker setup hook
- `usePerfomanceOptimiser.ts` - CPU-constrained device optimization (reduces video quality on low-power devices)
- `getLiveKitURL.ts` - Handles multi-region URL construction for LiveKit Cloud
- UI components: `SettingsMenu.tsx`, `RecordingIndicator.tsx`, `Debug.tsx`, `KeyboardShortcuts.tsx`, `CameraSettings.tsx`, `MicrophoneSettings.tsx`

### Key Flows

**Room Join Flow:**
1. User enters name in PreJoin component (in `PageClientImpl.tsx`)
2. Fetch connection details from `/api/connection-details` with roomName and participantName
3. Server generates JWT token with 5-minute TTL using `livekit-server-sdk`
4. Client connects to LiveKit server with token and optional E2EE setup
5. VideoConference component renders with LiveKit Components

**LiveKit WebSocket Protocol:**
- Uses **Protocol Buffers (protobuf)** for all signaling messages, not JSON
- Client sends `SignalRequest` messages, server responds with `SignalResponse` messages
- WebSocket connects to `/rtc` endpoint with JWT access token
- Establishes up to two WebRTC peer connections:
  - **Subscriber connection** - for receiving tracks from other participants
  - **Publisher connection** - for sending your own tracks (only if publishing)
- Signaling includes: participant join/leave, track subscriptions, ICE candidates, SDP offer/answer
- See [LiveKit Protocol Repository](https://github.com/livekit/protocol) and [Client Protocol Docs](https://docs.livekit.io/reference/internals/client-protocol/)

**E2EE Flow:**
- Passphrase encoded in URL hash fragment (`#passphrase`)
- `useSetupE2EE()` hook extracts passphrase from `location.hash`
- Creates E2EE worker from `livekit-client/e2ee-worker`
- ExternalE2EEKeyProvider decodes passphrase and sets encryption key
- Note: E2EE disables certain codecs (AV1, VP9) and RED for compatibility

**Recording Flow:**
- Uses LiveKit Egress API for room composite recordings
- Requires S3 configuration in environment variables
- `/api/record/start` - Starts recording with EgressClient, uploads to S3
- `/api/record/stop` - Stops active recording
- **Security Note:** Recording endpoints have no authentication in this example

### Performance Optimizations

**Low CPU Mode (`useLowCPUOptimizer`):**
- Detects CPU-constrained devices via `ParticipantEvent.LocalTrackCpuConstrained`
- Automatically reduces publisher video quality (`track.prioritizePerformance()`)
- Reduces subscriber video quality to LOW for all remote tracks
- Can disable video processing if needed

**Next.js Configuration:**
- `reactStrictMode: false` - Disabled for LiveKit compatibility
- `productionBrowserSourceMaps: true` - Source maps enabled
- Source map loader for `.mjs` files (LiveKit dependencies)
- COOP/COEP headers set to `same-origin`/`credentialless` for SharedArrayBuffer support (required for E2EE worker)

### Token Generation

Access tokens created in `/api/connection-details/route.ts`:
- Identity format: `${participantName}__${randomPostfix}` (random postfix stored in cookie)
- TTL: 5 minutes
- Grants: `roomJoin`, `canPublish`, `canPublishData`, `canSubscribe`
- Cookie expiration: 2 hours (120 minutes)

### Region Support

Multi-region routing via `getLiveKitURL()`:
- Inserts region prefix for LiveKit Cloud URLs: `myproject.livekit.cloud` → `myproject.eu.production.livekit.cloud`
- Preserves staging environment: `myproject.staging.livekit.cloud` → `myproject.eu.staging.livekit.cloud`
- Non-LiveKit Cloud URLs returned unchanged

## TypeScript Configuration

- Path alias: `@/*` maps to project root
- Target: ES2020
- Strict mode enabled
- Source maps enabled for debugging
