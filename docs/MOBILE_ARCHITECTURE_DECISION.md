# TRINITY.AI --- Mobile Architecture Decision

**Date:** 2026-08-24

**Status:** APPROVED (architectural decision, no implementation in this
phase)

**Author:** Diego

**Reference phase:** G (post-G-4, pre-G-5)

------------------------------------------------------------------------

## 1. Context and objective

TRINITY.AI will be launched as a multi-client platform: backend API
(Core), desktop application (Electron), web application (Next.js), and,
in the future, a native mobile application (iOS + Android).

This document records the mobile architecture decision **before**
implementation, to ensure that the ongoing phases (G-5 onward) do not
create technical debt that would make the mobile client more difficult
to build.

The analysis covers 10 compatibility areas. No code changes were
necessary --- the current API is mobile-ready.

------------------------------------------------------------------------

## 2. Compatibility audit --- 10 areas

### Area 1: REST API compatibility with mobile clients

**Verdict: ✅ Fully compatible**

  | Aspect | Status |
|---|---|
| Protocol | HTTP/1.1 REST + JSON — universal standard |
| Authentication | `Authorization: Bearer <token>` — compatible with all frameworks |
| Responses | JSON with Pydantic schemas — type-safe |
| CORS | `allowed_origins: list[str]` configurable via env var (`ALLOWED_ORIGINS`) |
| Errors | Standard HTTP status codes (401, 404, 422, 429) |
| Pagination | `SessionListResponse` returns `total` — ready for lists |

**CORS in production:** `config.py:54` defaults to
`["http://localhost:3000", "http://localhost:3001"]`. For native mobile,
CORS does not apply (the client is not a browser). For a PWA, add the
production domain. The production validator (`config.py:116`) rejects
`localhost` in production --- correct behavior.

------------------------------------------------------------------------

### Area 2: JWT authentication for mobile

**Verdict: ✅ Compatible --- Bearer is the correct standard for mobile**

``` text
POST /auth/register  →  { access_token, user_id, email }
POST /auth/login     →  { access_token, user_id, email }
GET  /auth/me        →  { id, email, full_name, plan, is_active, created_at }
```

-   **Algorithm:** HS256 (via `python-jose`) --- `config.py:44`
-   **Expiration:** `jwt_expiration_minutes: int = 60 * 24` (24 hours)
    --- suitable for mobile
-   **Storage on mobile:**
    -   React Native: `expo-secure-store` (iOS Keychain / Android
        Keystore) --- **recommended**
    -   Flutter: `flutter_secure_storage`
    -   PWA: `localStorage` (same pattern as the current desktop/web
        implementation)

**Identified limitation (non-blocking):** No refresh token --- the
session expires after 24 hours. Acceptable workaround for the MVP:
automatically log in again when a 401 is received. Refresh tokens are
planned for Phase 8+.

**GET /auth/me after login:** The desktop app has a bug where `plan` is
hardcoded (fixed in web). On mobile, always call `GET /auth/me` after
login to retrieve the actual plan.

------------------------------------------------------------------------

### Area 3: SSE (Server-Sent Events) on Android and iOS

**Verdict: ⚠️ Compatible with a polyfill --- no architectural blocker**

The Core uses SSE via `StreamingResponse` with
`media_type="text/event-stream"` for:

-   `POST /sessions/{id}/chat` --- chat streaming
-   `POST /sessions/{id}/agent` --- agent streaming
-   `POST /workspaces/code/{id}/execute` --- code execution streaming

**Compatibility by platform:**

  | Platform | Native `EventSource` | `ReadableStream.getReader()` | Solution |
|---|---|---|---|
| iOS Safari (web/PWA) | ✅ Since iOS 9 | ✅ | Native |
| Android Chrome (web/PWA) | ✅ Since 2012 | ✅ | Native |
| React Native | ❌ Not available | ❌ Not available | Polyfill required |
| Flutter | ❌ Not available | N/A (Dart) | `http` package with `Stream` |
| Expo (managed) | ❌ Not available | ❌ Not available | Polyfill required |

**Polyfill for React Native:** `react-native-sse` (NPM, 1.5k stars)
implements `EventSource` according to the W3C specification.
Alternative: a manual parser over `fetch` with `ReadableStream` via the
`react-native-fetch-api` polyfill.

**Current SSE format (no Core changes needed):**

``` text
data: {"type":"text","content":"..."}\n\n

data: {"type":"done","session_id":"...","usage":{...}}\n\n

data: [DONE]\n\n
```

The `data: JSON\n\n` format is the W3C standard --- polyfills read it
correctly.

------------------------------------------------------------------------

### Area 4: TRINITY CODE on mobile

**Verdict: ✅ No blocker --- mobile is a pure client**

`POST /workspaces/code/{workspace_id}/execute` (`workspaces.py:98`)
returns SSE.

-   **Docker:** runs on the server, not on the mobile device --- zero
    impact
-   **Mobile flow:** user enters a command → JSON request → SSE output
    stream
-   **Timeout:** `CodeRunRequest.timeout` is validated in the schema
    (`ge=1, le=120`) --- mobile must respect it
-   **Visual feedback:** `tool_status` → `output` → `done` events map
    naturally to an embedded terminal UI on mobile

**UX recommendation:** use a monospace font (`JetBrains Mono` /
`Courier New`) for output.

------------------------------------------------------------------------

### Area 5: TRINITY WORK --- mobile limitations

**Verdict: ⚠️ Limitation identified --- no blocker for G-5**

The WORK workspace has the `requires_approval` / `approval_policy`
fields (`WorkWorkspace` model). The planned approval flow is: agent
requests an action → human approves/rejects.

**Gap for mobile:**

-   No push-notification mechanism when an action is awaiting approval
-   Mobile would need polling or WebSocket (not implemented) to receive
    alerts
-   The approval endpoint does not yet exist (Phase G+)

**Recommendation:** when implementing TRINITY WORK on mobile, add a
webhook/FCM (Firebase Cloud Messaging) integration in the backend to
send push notifications. This gap does not affect G-5
(AuditLog/PermissionGateway).

------------------------------------------------------------------------

### Area 6: Mobile framework comparison

  | Framework | Code reuse | Performance | SSE | Maturity | Ecosystem |
|---|---|---|---|---|---|
| **React Native + Expo** | ✅✅ High (TS + React) | ✅ Good | Simple polyfill | ✅✅ Mature | ✅✅ Large |
| Flutter | ❌ Low (Dart) | ✅✅ Excellent | Dart Stream | ✅✅ Mature | ✅ Growing |
| Capacitor (wrapper) | ✅✅ Maximum (web = mobile) | ⚠️ WebView | ✅ Native | ✅ Mature | Medium |
| PWA | ✅✅ Maximum (web = mobile) | ⚠️ iOS limitations | ✅ Native | ✅ Stable | N/A |

**Code-reuse cost analysis:**

TRINITY already has TypeScript + React + Zustand + shared types
(`types.ts`). With React Native + Expo:

-   `types.ts` → 100% reuse
-   `useAppStore.ts` → \~90% reuse (replace `localStorage` with
    AsyncStorage)
-   `api/client.ts` → \~85% reuse (replace `ReadableStream` with an SSE
    polyfill)
-   UI components → \~30% reuse (adapt Tailwind to StyleSheet or
    NativeWind)

------------------------------------------------------------------------

### Area 7: Proposed monorepo structure

``` text
packages/
├── core/              (FastAPI — Python)
├── agent/             (Agent executor — Python)
├── model-router/      (Model selector — Python)
├── sandbox/           (Docker sandbox — Python)
├── desktop/           (Electron + React — TypeScript)
├── web/               (Next.js 14 — TypeScript)
└── mobile/            (Expo + React Native — TypeScript)  ← Future phase
    ├── app/
    │   ├── _layout.tsx
    │   ├── (auth)/
    │   │   └── login.tsx
    │   └── (app)/
    │       ├── chat/
    │       ├── code/
    │       └── settings/
    ├── src/
    │   ├── api/client.ts     (based on packages/web/src/api/client.ts)
    │   ├── store/            (Zustand, same as web)
    │   └── types.ts          (copy of packages/web/src/types.ts)
    ├── package.json
    └── app.json
```

**Dependencies to add in `packages/mobile`:**

``` json
{
  "expo": "~51",
  "react-native": "0.74",
  "expo-router": "~3",
  "expo-secure-store": "~13",
  "react-native-sse": "^1.0.0",
  "zustand": "^5"
}
```

**There is no need to modify `packages/core/`.**

------------------------------------------------------------------------

### Area 8: Architectural principles for mobile compatibility

The following Core principles ensure mobile compatibility without
modifications:

1.  **API-first:** backend is completely client-agnostic (no server-side
    rendering)
2.  **Stateless Bearer JWT:** mobile does not depend on cookies or
    server sessions
3.  **Validated JSON schemas (Pydantic):** clients can generate types
    automatically
4.  **W3C SSE format:** `data: JSON\n\n` --- polyfills implement it
    faithfully
5.  **Configurable CORS:** `ALLOWED_ORIGINS` env var --- no `*` in
    production
6.  **Semantic HTTP statuses:** 401/403/404/422/429 --- mobile handles
    them the same way as web

**Principle to reinforce in future phases:** never add logic that
assumes `window`, `localStorage`, `document`, or the DOM in the backend.

------------------------------------------------------------------------

### Area 9: Impact on future phases

  | Phase | Impact on mobile | Required action |
|---|---|---|
| G-5 (AuditLog + PermissionGateway) | None — backend only | None |
| Phase 7 (Web MVP) | Shareable codebase | Keep `types.ts` synchronized |
| Phase 8 (Billing) | Mobile-specific payment sheet | Keep backend client-agnostic (webhook) |
| Phase 9 (Memory) | Client-agnostic | None |
| Mobile phase | Implement `packages/mobile` | See structure in Area 7 |

**Push notifications (WORK approval):** when implementing agent action
approval, add `device_token` to the User model and integrate with
FCM/APNs. The backend will have the endpoint
`POST /users/me/device-token`.

------------------------------------------------------------------------

### Area 10: Conclusion and decision

**Architectural decision: React Native with Expo**

**Reason:** maximum code reuse for existing TypeScript, React, Zustand,
and types. The learning curve is minimal because the team already works
with React on desktop and web.

**When to implement:** after Phase 7 (Web MVP). The web MVP validates
the API client layer and store --- mobile can inherit this tested
foundation.

**Prerequisites before starting mobile:**

-   [ ] `packages/web/` is stable and its tests are passing
-   [ ] `types.ts` is synchronized between `web/` and `mobile/`
-   [ ] API base URL is configurable via env (`EXPO_PUBLIC_API_URL`)
-   [ ] `expo-secure-store` is used for JWT (not `localStorage`)
-   [ ] SSE polyfill: `react-native-sse`

**No backend files need to be modified to support mobile.**

------------------------------------------------------------------------

## 3. Critical issues found

**None.** The current API is mobile-ready without modifications.

Documented limitations (not blockers for G-5):

-   SSE requires a polyfill in React Native (an implementation decision,
    not an architectural one)
-   No refresh token (acceptable for the MVP, Phase 8+)
-   WORK approval has no push notifications (future phase)

------------------------------------------------------------------------

## 4. Test status (confirmed)

The audit was performed by reading the code --- no modifications were
made. Tests run after the audit: **385 passed, 3 skipped, 0 failed**.

Files inspected:

-   `packages/core/src/main.py` --- CORS middleware
-   `packages/core/src/config.py` --- `allowed_origins`, JWT settings
-   `packages/core/src/services/auth_service.py` --- JWT creation/decode
-   `packages/core/src/schemas/auth.py` --- auth schemas
-   `packages/core/src/api/routes/auth.py` --- auth endpoints
-   `packages/core/src/api/routes/sessions.py` --- chat/agent SSE
-   `packages/core/src/api/routes/workspaces.py` --- code execution SSE
-   `packages/core/src/services/code_service.py` ---
    `stream_code_execute`
-   `packages/core/src/schemas/code.py` --- `CodeRunRequest`
-   `packages/desktop/src/renderer/src/api/client.ts` --- SSE
    `ReadableStream` pattern
-   `packages/desktop/src/renderer/src/types.ts` --- shareable types
