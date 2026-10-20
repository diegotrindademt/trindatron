# TRINITY.AI â€” Mobile Architecture Decision

**Data:** 2026-08-24
**Status:** APROVADO (decisÃ£o arquitetural, sem implementaÃ§Ã£o nesta fase)
**Autor:** Diego
**Fase de referÃªncia:** G (pÃ³s-G-4, prÃ©-G-5)

---

## 1. Contexto e objetivo

TRINITY.AI serÃ¡ lanÃ§ado como plataforma multi-cliente: API backend (Core), aplicaÃ§Ã£o desktop
(Electron), aplicaÃ§Ã£o web (Next.js) e, futuramente, aplicativo mÃ³vel nativo (iOS + Android).

Este documento registra a decisÃ£o de arquitetura mobile **antes** da implementaÃ§Ã£o, para garantir
que as fases em andamento (G-5 em diante) nÃ£o criem dÃ­vidas tÃ©cnicas que dificultem o cliente
mÃ³vel.

A anÃ¡lise cobre 10 Ã¡reas de compatibilidade. Nenhuma modificaÃ§Ã£o de cÃ³digo foi necessÃ¡ria â€”
a API atual Ã© mobile-ready.

---

## 2. Auditoria de compatibilidade â€” 10 Ã¡reas

### Ãrea 1: Compatibilidade da API REST com clientes mÃ³veis

**Veredicto: âœ… Totalmente compatÃ­vel**

| Aspecto | Status |
|---|---|
| Protocolo | HTTP/1.1 REST + JSON â€” padrÃ£o universal |
| AutenticaÃ§Ã£o | `Authorization: Bearer <token>` â€” compatÃ­vel com todos os frameworks |
| Respostas | JSON com Pydantic schemas â€” tipagem segura |
| CORS | `allowed_origins: list[str]` configurÃ¡vel via env var (`ALLOWED_ORIGINS`) |
| Erros | HTTP status codes padrÃ£o (401, 404, 422, 429) |
| PaginaÃ§Ã£o | `SessionListResponse` retorna `total` â€” pronto para listas |

**CORS em produÃ§Ã£o:** `config.py:54` default `["http://localhost:3000", "http://localhost:3001"]`.
Para mobile nativo, CORS nÃ£o se aplica (cliente nÃ£o Ã© browser). Para PWA, adicionar domÃ­nio
de produÃ§Ã£o. O validator de produÃ§Ã£o (`config.py:116`) rejeita `localhost` em prod â€”
comportamento correto.

---

### Ãrea 2: AutenticaÃ§Ã£o JWT para mobile

**Veredicto: âœ… CompatÃ­vel â€” Bearer Ã© o padrÃ£o correto para mobile**

```
POST /auth/register  â†’  { access_token, user_id, email }
POST /auth/login     â†’  { access_token, user_id, email }
GET  /auth/me        â†’  { id, email, full_name, plan, is_active, created_at }
```

- **Algoritmo:** HS256 (via `python-jose`) â€” `config.py:44`
- **ExpiraÃ§Ã£o:** `jwt_expiration_minutes: int = 60 * 24` (24 horas) â€” adequado para mobile
- **Armazenamento no mobile:**
  - React Native: `expo-secure-store` (iOS Keychain / Android Keystore) â€” **recomendado**
  - Flutter: `flutter_secure_storage`
  - PWA: `localStorage` (mesmo padrÃ£o do desktop/web atual)

**LimitaÃ§Ã£o identificada (nÃ£o bloqueante):** Sem refresh token â€” sessÃ£o expira em 24h.
Workaround aceitÃ¡vel para MVP: re-login automÃ¡tico quando 401. Refresh tokens sÃ£o Fase 8+.

**GET /auth/me apÃ³s login:** O desktop tem bug de `plan` hardcoded (corrigido no web).
No mobile, sempre chamar `GET /auth/me` apÃ³s login para obter o plano real.

---

### Ãrea 3: SSE (Server-Sent Events) no Android e iOS

**Veredicto: âš ï¸ CompatÃ­vel com polyfill â€” sem bloqueio arquitetural**

O Core usa SSE via `StreamingResponse` com `media_type="text/event-stream"` para:
- `POST /sessions/{id}/chat` â€” streaming de chat
- `POST /sessions/{id}/agent` â€” streaming de agente
- `POST /workspaces/code/{id}/execute` â€” streaming de execuÃ§Ã£o de cÃ³digo

**Compatibilidade por plataforma:**

| Plataforma | `EventSource` nativo | `ReadableStream.getReader()` | SoluÃ§Ã£o |
|---|---|---|---|
| iOS Safari (web/PWA) | âœ… desde iOS 9 | âœ… | Nativo |
| Android Chrome (web/PWA) | âœ… desde 2012 | âœ… | Nativo |
| React Native | âŒ ausente | âŒ ausente | Polyfill necessÃ¡rio |
| Flutter | âŒ ausente | N/A (Dart) | `http` package com `Stream` |
| Expo (managed) | âŒ ausente | âŒ ausente | Polyfill necessÃ¡rio |

**Polyfill para React Native:** `react-native-sse` (NPM, 1.5k stars) implementa
`EventSource` conforme especificaÃ§Ã£o W3C. Alternativa: parser manual sobre
`fetch` com `ReadableStream` via polyfill `react-native-fetch-api`.

**Formato SSE atual (sem mudanÃ§a necessÃ¡ria no Core):**
```
data: {"type":"text","content":"..."}\n\n
data: {"type":"done","session_id":"...","usage":{...}}\n\n
data: [DONE]\n\n
```

O formato `data: JSON\n\n` Ã© o padrÃ£o W3C â€” polyfills leem corretamente.

---

### Ãrea 4: TRINITY CODE em mobile

**Veredicto: âœ… Sem bloqueador â€” mobile Ã© cliente puro**

`POST /workspaces/code/{workspace_id}/execute` (`workspaces.py:98`) retorna SSE.

- **Docker:** roda no servidor, nÃ£o no dispositivo mÃ³vel â€” zero impacto
- **Mobile flow:** usuÃ¡rio digita comando â†’ request JSON â†’ SSE stream de output
- **Timeout:** `CodeRunRequest.timeout` validado no schema (ge=1, le=120) â€” mobile respeitarÃ¡
- **Feedback visual:** eventos `tool_status` â†’ `output` â†’ `done` mapeiam naturalmente
  para uma UI de terminal embutida no mobile

**UX recommendation:** usar fonte monospace (`JetBrains Mono` / `Courier New`) para output.

---

### Ãrea 5: TRINITY WORK â€” limitaÃ§Ãµes em mobile

**Veredicto: âš ï¸ LimitaÃ§Ã£o identificada â€” sem bloqueio para G-5**

WORK workspace tem campo `requires_approval` / `approval_policy` (WorkWorkspace model).
Fluxo de aprovaÃ§Ã£o planejado: agente solicita aÃ§Ã£o â†’ humano aprova/rejeita.

**Gap para mobile:**
- Sem mecanismo de push notification quando hÃ¡ aÃ§Ã£o pendente de aprovaÃ§Ã£o
- Mobile precisaria de polling ou WebSocket (nÃ£o implementado) para receber alertas
- O endpoint de aprovaÃ§Ã£o ainda nÃ£o existe (Fase G+)

**RecomendaÃ§Ã£o:** quando implementar TRINITY WORK em mobile, adicionar webhook/FCM
(Firebase Cloud Messaging) no backend para disparar push notifications.
Este gap nÃ£o afeta G-5 (AuditLog/PermissionGateway).

---

### Ãrea 6: ComparaÃ§Ã£o de frameworks mobile

| Framework | Reuso de cÃ³digo | Performance | SSE | Maturidade | Ecossistema |
|---|---|---|---|---|---|
| **React Native + Expo** | âœ…âœ… Alto (TS + React) | âœ… Boa | Polyfill simples | âœ…âœ… Maduro | âœ…âœ… Grande |
| Flutter | âŒ Baixo (Dart) | âœ…âœ… Excelente | Dart Stream | âœ…âœ… Maduro | âœ… Crescendo |
| Capacitor (wrapper) | âœ…âœ… MÃ¡ximo (web=mobile) | âš ï¸ WebView | âœ… Nativo | âœ… Maduro | MÃ©dio |
| PWA | âœ…âœ… MÃ¡ximo (web=mobile) | âš ï¸ LimitaÃ§Ãµes iOS | âœ… Nativo | âœ… EstÃ¡vel | N/A |

**AnÃ¡lise de custos de reuso:**

TRINITY jÃ¡ possui TypeScript + React + Zustand + tipos compartilhados (`types.ts`).
Com React Native + Expo:
- `types.ts` â†’ reuso 100%
- `useAppStore.ts` â†’ reuso ~90% (trocar localStorage por AsyncStorage)
- `api/client.ts` â†’ reuso ~85% (trocar `ReadableStream` por polyfill SSE)
- UI components â†’ reuso ~30% (adaptar Tailwind para StyleSheet ou NativeWind)

---

### Ãrea 7: Estrutura de monorepo proposta

```
packages/
â”œâ”€â”€ core/              (FastAPI â€” Python)
â”œâ”€â”€ agent/             (Agent executor â€” Python)
â”œâ”€â”€ model-router/      (Model selector â€” Python)
â”œâ”€â”€ sandbox/           (Docker sandbox â€” Python)
â”œâ”€â”€ desktop/           (Electron + React â€” TypeScript)
â”œâ”€â”€ web/               (Next.js 14 â€” TypeScript)
â””â”€â”€ mobile/            (Expo + React Native â€” TypeScript)  â† Fase futura
    â”œâ”€â”€ app/
    â”‚   â”œâ”€â”€ _layout.tsx
    â”‚   â”œâ”€â”€ (auth)/
    â”‚   â”‚   â””â”€â”€ login.tsx
    â”‚   â””â”€â”€ (app)/
    â”‚       â”œâ”€â”€ chat/
    â”‚       â”œâ”€â”€ code/
    â”‚       â””â”€â”€ settings/
    â”œâ”€â”€ src/
    â”‚   â”œâ”€â”€ api/client.ts     (baseado em packages/web/src/api/client.ts)
    â”‚   â”œâ”€â”€ store/            (Zustand, igual ao web)
    â”‚   â””â”€â”€ types.ts          (cÃ³pia de packages/web/src/types.ts)
    â”œâ”€â”€ package.json
    â””â”€â”€ app.json
```

**DependÃªncias a adicionar em `packages/mobile`:**
```json
{
  "expo": "~51",
  "react-native": "0.74",
  "expo-router": "~3",
  "expo-secure-store": "~13",
  "react-native-sse": "^1.0.0",
  "zustand": "^5"
}
```

**NÃ£o Ã© necessÃ¡rio modificar `packages/core/`.**

---

### Ãrea 8: PrincÃ­pios arquiteturais para compatibilidade mobile

Os seguintes princÃ­pios do Core garantem compatibilidade mobile sem modificaÃ§Ã£o:

1. **API-first:** backend completamente agnÃ³stico de cliente (sem server-side rendering)
2. **Bearer JWT stateless:** mobile nÃ£o depende de cookies ou sessÃµes de servidor
3. **JSON schemas validados (Pydantic):** clientes podem gerar tipos automaticamente
4. **SSE format W3C:** `data: JSON\n\n` â€” polyfills implementam fielmente
5. **CORS configurÃ¡vel:** `ALLOWED_ORIGINS` env var â€” sem `*` em produÃ§Ã£o
6. **HTTP status semÃ¢nticos:** 401/403/404/422/429 â€” mobile trata igual ao web

**PrincÃ­pio a reforÃ§ar em fases futuras:** nunca adicionar lÃ³gica que assuma
`window`, `localStorage`, `document` ou DOM no backend.

---

### Ãrea 9: Impacto nas fases futuras

| Fase | Impacto em mobile | AÃ§Ã£o necessÃ¡ria |
|---|---|---|
| G-5 (AuditLog + PermissionGateway) | Nenhum â€” backend only | Nada |
| Fase 7 (Web MVP) | Base de cÃ³digo compartilhÃ¡vel | Manter `types.ts` sincronizado |
| Fase 8 (Billing) | Payment sheet mobile-specific | Backend agnÃ³stico (webhook) |
| Fase 9 (Memory) | AgnÃ³stico de cliente | Nada |
| Fase Mobile | Implementar `packages/mobile` | Ver estrutura Ãrea 7 |

**Push notifications (WORK approval):** quando implementar aprovaÃ§Ã£o de aÃ§Ãµes do agente,
adicionar `device_token` ao User model e integraÃ§Ã£o com FCM/APNs. Backend fica com endpoint
`POST /users/me/device-token`.

---

### Ãrea 10: ConclusÃ£o e decisÃ£o

**DecisÃ£o arquitetural:** **React Native com Expo**

**RazÃ£o:** mÃ¡ximo reuso de cÃ³digo TypeScript, React, Zustand e tipos jÃ¡ existentes.
A curva de aprendizado Ã© mÃ­nima dado que o time jÃ¡ trabalha com React no desktop e web.

**Quando implementar:** apÃ³s Fase 7 (Web MVP). O web MVP valida a API client layer
e a store â€” o mobile herda essa base testada.

**PrÃ©-requisitos antes de iniciar o mobile:**
- [ ] `packages/web/` estÃ¡vel e com testes passando
- [ ] `types.ts` sincronizado entre `web/` e `mobile/`
- [ ] API base URL configurÃ¡vel via env (`EXPO_PUBLIC_API_URL`)
- [ ] `expo-secure-store` para JWT (nÃ£o localStorage)
- [ ] Polyfill SSE: `react-native-sse`

**Nenhum arquivo de backend precisa ser modificado para suportar mobile.**

---

## 3. Problemas crÃ­ticos encontrados

**Nenhum.** A API atual Ã© mobile-ready sem modificaÃ§Ãµes.

LimitaÃ§Ãµes documentadas (nÃ£o bloqueantes para G-5):
- SSE requer polyfill em React Native (decisÃ£o de implementaÃ§Ã£o, nÃ£o arquitetural)
- Sem refresh token (aceitÃ¡vel para MVP, Fase 8+)
- WORK approval sem push notifications (Fase futura)

---

## 4. Estado dos testes (confirmado)

Auditoria realizada com base em leitura de cÃ³digo â€” zero modificaÃ§Ãµes feitas.
Testes executados apÃ³s auditoria: **385 passed, 3 skipped, 0 failed**.

Arquivos inspecionados:
- `packages/core/src/main.py` â€” CORS middleware
- `packages/core/src/config.py` â€” `allowed_origins`, JWT settings
- `packages/core/src/services/auth_service.py` â€” JWT criaÃ§Ã£o/decode
- `packages/core/src/schemas/auth.py` â€” schemas de auth
- `packages/core/src/api/routes/auth.py` â€” endpoints de auth
- `packages/core/src/api/routes/sessions.py` â€” SSE chat/agent
- `packages/core/src/api/routes/workspaces.py` â€” SSE code execute
- `packages/core/src/services/code_service.py` â€” stream_code_execute
- `packages/core/src/schemas/code.py` â€” CodeRunRequest
- `packages/desktop/src/renderer/src/api/client.ts` â€” padrÃ£o SSE ReadableStream
- `packages/desktop/src/renderer/src/types.ts` â€” tipos compartilhÃ¡veis
