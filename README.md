# Dial0

Describe a customer-service problem in a chat box; an LLM agent researches it with
Firecrawl web search, then places a real outbound phone call on your behalf through Vapi
and streams the transcript and call events back into the app.

This was built for the DialZero hackathon and pushed as a single commit. The calling path
is real and does work end to end. The analytics side is not: the company dashboard renders
invented numbers from a fixture file. Read it as a demo, not a product.

## What's here

Next.js 15 App Router, React 19, TypeScript, Tailwind 4, Convex for the backend and live
queries, Better Auth for login. Bun for installs and the lockfile.

- `lib/langgraph/orchestrator.ts` — the real core (~68 KB). A LangGraph multi-agent loop
  with two tools: `firecrawl_search` for research, and `start_call` to trigger the phone
  call (which it reaches by calling this app's own `/api/mcp`).
- `app/api/chat/route.ts` — the only route the UI calls. Loads settings and the issue from
  Convex, runs the orchestrator, streams back over SSE.
- `app/api/vapi/start-call/route.ts` — builds the system prompt and POSTs to
  `api.vapi.ai/call`. Configures Groq `moonshotai/kimi-k2-instruct-0905` for the in-call
  model and Deepgram `nova-3` for transcription, plus listen-in monitor URLs.
- `app/api/vapi/webhook/route.ts` — ingests Vapi call events and appends them to Convex.
- `app/api/mcp/route.ts` — a small JSON-RPC MCP server exposing `start_call`.
- `convex/` — schema (`issues`, `settings`, `orchestrationContexts`, `chatMessages`,
  `callEvents`, `toolCalls`), orchestration functions, Better Auth, Autumn billing, and
  ElevenLabs voice cloning via Vapi.
- `app/` pages — dashboard, activity feed, billing, settings, onboarding, and voice
  creation are all backed by real Convex or Autumn queries.
- `components/`, `hooks/`, `lib/` — UI (shadcn/ui + Radix), chat hook, helpers.

Roughly fifteen top-level markdown files (`FIXES_APPLIED.md`, `DEMO_DASHBOARD_REDESIGN.md`,
`COMPANY_DASHBOARD_IMPLEMENTATION.md`, and so on) are leftover working notes from the
build, not documentation. Three of them are empty files. `SETUP.md`, `MCP_SETUP.md`, and
`diagram.md` are the ones worth reading.

## Running it

```bash
bun install
bun dev        # runs `next dev --turbopack` and `convex dev` together
```

You need accounts and keys for Convex, Vapi (with a phone number provisioned), Firecrawl,
Resend, Autumn, and an OpenAI-compatible LLM endpoint. Env vars the code reads, by name:

`NEXT_PUBLIC_CONVEX_URL`, `CONVEX_SITE_URL`, `SITE_URL`, `NEXT_PUBLIC_SITE_URL`,
`VAPI_PRIVATE_API_KEY`, `VAPI_PHONE_NUMBER_ID`, `VAPI_ORG_ID`, `VAPI_PUBLIC_ASSISTANT_ID`,
`VAPI_WEBHOOK_URL`, `NGROK_WEBHOOK_URL`, `FIRECRAWL_API_KEY`, `FIRECRAWL_API_BASE_URL`,
`OPENAI_API_KEY`, `OPENAI_MODEL`, `OPENAI_BASE_URL`, `OPENAI_TEMPERATURE`,
`INKEEP_API_KEY`, `RESEND_API_KEY`, `RESEND_FROM`, `INTERNAL_EMAIL_PROXY_SECRET`,
`AUTUMN_SECRET_KEY`, `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`, and the
`NEXT_PUBLIC_USER_*` / `NEXT_PUBLIC_SERVICE_*` demo identity values.

Vapi webhooks need a public URL, so local development goes through ngrok. See `SETUP.md`.

## Status / limitations

Known gaps, in rough order of how much they would surprise you:

- **The company dashboard is fake.** `convex/companyDashboard.ts` derives every stat,
  chart, and row from `convex/mockCompanyData.ts`, a hand-written fixture for a real named
  insurer. A comment in the file says as much. Nothing there is measured.
- **The showcase dashboard issues are canned.** IDs beginning `demo-` route to
  `components/demo-issue-dashboard.tsx`, which is hardcoded content for a named carrier.
- **Inkeep is not wired up**, despite the hackathon submission naming it. `lib/inkeep-service.ts`
  is never imported and its `openai` import is not an installed package. The route at
  `app/api/inkeep/route.ts` actually runs Google Gemini 2.5 Pro, reading the key from a
  variable named `INKEEP_API_KEY`, and nothing in the app fetches it.
- `lib/langgraph/orchestrator.ts` hardcodes `baseURL: "https://ai.hackclub.com/proxy/v1"`
  at the call site, so `OPENAI_BASE_URL` is read but ignored there.
- `lib/companyInsights.ts` is labelled AI insight generation; `analyzeSentiment` is a
  keyword counter over two hardcoded word lists.
- `next.config.mjs` sets both `eslint.ignoreDuringBuilds` and `typescript.ignoreBuildErrors`
  to true, so the repo builds whether or not it typechecks.
- The MCP token check in `app/api/mcp/route.ts` treats a null settings lookup as valid, and
  the Vapi webhook will accept its auth token from the query string. Fine for a demo, not
  for anything exposed.
- `.env.example` lists several variables nothing reads (`AGENT_BASE_URL`, `AGENT_API_KEY`,
  `VAPI_PUBLIC_API_KEY`) and omits ones the code needs (`FIRECRAWL_API_KEY`, `OPENAI_*`).
- `tsconfig.tsbuildinfo` is committed and should be ignored.
- `lib/search.ts` (Google Custom Search) is dead code, and `@anthropic-ai/sdk` is a
  dependency nothing imports.

## Attribution

Built for the DialZero hackathon on its sponsor stack: [Vapi](https://vapi.ai) for voice
calls and, through it, [ElevenLabs](https://elevenlabs.io) voice cloning,
[Groq](https://groq.com) inference, and [Deepgram](https://deepgram.com) transcription.
[Convex](https://convex.dev) for the reactive backend, [Better Auth](https://better-auth.com)
for identity, [Resend](https://resend.com) for email, [Autumn](https://useautumn.com) for
usage metering, and [Firecrawl](https://firecrawl.dev) for web research.

Also uses [Next.js](https://nextjs.org), [LangGraph and LangChain](https://langchain.com),
[shadcn/ui](https://ui.shadcn.com) with [Radix](https://radix-ui.com),
[Google Generative AI](https://ai.google.dev/), and the
[Hack Club AI proxy](https://ai.hackclub.com) as the LLM endpoint.
