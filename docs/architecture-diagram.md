# ProGear Sales AI Demo — Architecture

```
╔═══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╗
║                                                  PROGEAR SALES AI DEMO — ARCHITECTURE                                                                 ║
╚═══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╝

┌─────────────────────────────┐          ┌──────────────────────────────────────────────────────────────────────────────────┐
│  BROWSER                    │          │  FASTAPI BACKEND  (Render, Python / Uvicorn)                                      │
│  (Vercel, Next.js 14+React) │  HTTPS   │                                                                                  │        ┌──────────────────────────────────────────────────────┐
│                             │          │  main.py                                                                         │  ①    │  OKTA  (Org Authorization Server)                    │
│  NextAuth.js  (OIDC client) │─────────►│  ┌────────────────────────────────────────────────────────────────────────────┐  │───────►│  /oauth2/{orgAS}/v1/authorize                        │
│  /api/auth/[...nextauth]    │  (JWT)   │  │ POST /api/chat   → orchestrate(msg, id_token)                      (→ ②)  │  │◄───────│  /oauth2/{orgAS}/v1/token  (OIDC code exchange)      │
│                             │          │  │ GET  /api/okta/logs → Okta System Logs audit trail                  (→ ①)  │  │        │  ID-JAG token exchange  (RFC 8693)                   │
│  app/page.tsx               │          │  └────────────────────────────────────────────────────────────────────────────┘  │        │  Agent Identity: wlp...  (RS256 private key JWK)     │
│  ┌─────────────────────┐    │          │                                                                                  │        └──────────────────────────────────────────────────────┘
│  │ Chat interface      │    │          │  orchestrator.py  (LangGraph multi-agent workflow)                               │
│  │ Token exchange cards│    │◄─────────│  ┌────────────────────────────────────────────────────────────────────────────┐  │        ┌──────────────────────────────────────────────────────┐
└──┴─────────────────────┴────┘ content  │  │ Node 1: Router     → Claude LLM: intent detection + scope analysis          │  │  ②    │  OKTA  (Custom Authorization Servers)                │
                                 +cards  │  │ Node 2: Token Exch → multi_agent_auth.py: obtain scoped tokens per domain   │  │───────►│  Sales AS  ·  Inventory AS                           │
                                         │  │ Node 3: Agents     → invoke each domain agent in parallel w/ scoped token   │  │◄───────│  Customer AS  ·  Pricing AS                          │
                                         │  │ Node 4: Synthesis  → Claude LLM: synthesize all agent results               │  │        │  Evaluates RBAC group policies per scope request     │
                                         │  └────────────────────────────────────────────────────────────────────────────┘  │        │  Returns access_token  or  access_denied             │
                                         │                                                                                  │        └──────────────────────────────────────────────────────┘
                                         │  multi_agent_auth.py  +  okta_cross_app_access.py                               │
                                         │  ┌────────────────────────────────────────────────────────────────────────────┐  │        ┌──────────────────────────────────────────────────────┐
                                         │  │ STEP 1: id_token → Org AS → id_jag_token                         (→ ①)   │  │        │  OKTA  (User Directory + System Logs)                │
                                         │  │ STEP 2: id_jag   → Custom AS → scoped access_token                (→ ②)   │  │        │  Groups: ProGear-Sales · Warehouse · Finance          │
                                         │  │ client_assertion: RS256 JWT signed by agent private key (JWK)              │  │        │  System Logs API: /api/v1/logs  (audit trail)        │
                                         │  │ In-memory token cache keyed by (user_id, domain)  TTL = exp_in            │  │        └──────────────────────────────────────────────────────┘
                                         │  └────────────────────────────────────────────────────────────────────────────┘  │
                                         │                                                                                  │        ┌──────────────────────────────────────────────────────┐
                                         │  AI Agents  (each receives domain-scoped access_token from Node 2)              │  ③    │  ANTHROPIC API                                       │
                                         │  ┌───────────────────────┐  ┌──────────────────────────┐                        │───────►│  claude-sonnet-4-20250514                            │
                                         │  │ sales_agent.py        │  │ inventory_agent.py        │                        │◄───────│  Node 1: intent detection + scope routing            │
                                         │  │ sales:read            │  │ inventory:read             │                        │        │  Node 4: multi-agent response synthesis              │
                                         │  │ sales:forecast        │  │ inventory:write            │                        │        └──────────────────────────────────────────────────────┘
                                         │  └───────────────────────┘  │ inventory:alert            │
                                         │  ┌───────────────────────┐  └──────────────────────────┘
                                         │  │ customer_agent.py     │  ┌──────────────────────────┐
                                         │  │ customer:read         │  │ pricing_agent.py          │
                                         │  │ customer:write        │  │ pricing:read              │
                                         │  │ customer:history      │  │ pricing:margin            │
                                         │  └───────────────────────┘  │ pricing:discount          │
                                         │                              └──────────────────────────┘
                                         │  Each agent → tool_call → demo_store.py
                                         │
                                         │  demo_store.py  (in-memory data — no external database)
                                         │  ┌────────────────────────────────────────────────────────────────────────────┐
                                         │  │ products · inventory · customers · orders · pricing  (static demo data)    │
                                         │  │ Agents call tools; read/write gated by access_token scope claims           │
                                         │  └────────────────────────────────────────────────────────────────────────────┘
                                         └──────────────────────────────────────────────────────────────────────────────────┘


KEY FLOWS
─────────
① User Authentication (OIDC):  Browser → NextAuth /api/auth/[...nextauth] → Okta /v1/authorize (PKCE + openid profile email)
                                → user login + MFA → authorization code callback → /v1/token exchange → {id_token, access_token}
                                → NextAuth session cookie set.  id_token stored server-side for downstream token exchange.
                                GET /api/okta/logs reads Okta System Logs API to surface the audit trail in the UI.

② Per-Message Token Exchange (RFC 8693, once per required domain per chat message):
                                STEP 1: user id_token → Org AS  →  id_jag_token  (sub = user_id, act = agent_id)
                                STEP 2: id_jag_token  → Custom AS  →  scoped access_token  (RBAC evaluated: user's Okta group vs. policy)
                                If user's group lacks the required scope, Custom AS returns access_denied — surfaced in UI as an
                                authorization boundary, not an error.  Tokens cached per (user_id, domain) until token expiry.

③ LLM Calls (Anthropic Claude):  Node 1 sends conversation to Claude → receives routing JSON {agent: needed, scopes: [...]}.
                                  Node 4 sends all agent results to Claude → receives synthesized natural-language response.

   Full per-message flow:  Browser POST /api/chat
                              → [Node 1]  Claude: intent detection + agent/scope routing
                              → [Node 2]  RFC 8693 token exchange per required domain  (Org AS STEP 1  →  Custom AS STEP 2)
                              → [Node 3]  parallel agent execution  (each agent calls demo_store tools with scoped token)
                              → [Node 4]  Claude: synthesize agent results into natural-language response
                              → JSON {content, agent_flow, token_exchanges} returned to browser
```
