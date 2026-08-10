# Testbench-infra

Deploy configs and ops tooling for Testbench's backend services. This repo does **not**
contain application code — `testbench-api` and `testbench-pipeline` each live in their
own repo and deploy independently to the Heroku apps described below.

## Extraction pipeline — three LLM strategies

`testbench-backend`'s `extraction.strategy.ts` routes each upload to one of three
extractors, based on OCR confidence and document size:

| Condition | Strategy | Provider |
|---|---|---|
| Confidence ≥ threshold, text fits Groq's TPM budget | `FastLLMExtractor` | Groq (`llama-3.1-8b-instant`) |
| Confidence ≥ threshold, text too large for Groq | `OpenAIOverflowExtractor` | Azure OpenAI (`gpt-4.1-mini`) |
| Confidence < threshold | `VisionLLMExtractor` | Azure OpenAI (`gpt-4.1-mini`, vision) |

**History**: the vision path was originally broken (pointed at the Document
Intelligence resource instead of an actual Azure OpenAI resource, 401'd on every
call), and large documents originally failed outright on Groq's TPM rate limit
before being routed elsewhere. Both fixed 2026-08-09 — a real Azure OpenAI/AI
Foundry resource (`testbench-llm-resource`, `gpt-4.1-mini` deployment, 30k TPM) now
backs both the vision fallback and the large-document overflow case. Verified
end-to-end against a real 204-page/94,624-char PDF: 48 questions extracted
correctly in ~40s, single request.

## Services

| App | Repo | Stack | Role |
|---|---|---|---|
| [testbench-api](https://dashboard.heroku.com/apps/testbench-api) | testbench-api | FastAPI (Python) | Public API, owns the DB schema |
| [testbench-pipeline](https://dashboard.heroku.com/apps/testbench-pipeline) | testbench-pipeline | Node/Express | OCR/LLM extraction |

- testbench-api: https://api.testbench.study/ (custom domain; raw Heroku URL
  `https://testbench-api-53f53b05d813.herokuapp.com/` still works too)
- testbench-pipeline: https://testbench-pipeline-0a144388bf78.herokuapp.com/
  (internal only, never called directly by students - no custom domain needed)
- frontend: https://www.testbench.study/ (deployed on Vercel; `testbench.study`
  308-redirects to `www`)

## Dyno tier — read this before touching dyno settings

**Both apps must run on Basic dynos. Never switch either app to Eco.**

Eco dynos sleep after 30 minutes of inactivity, which is disqualifying for this
project (always-on is a hard requirement). Basic dyno = $7/mo per app, always-on,
no sleep.

If you ever see a dyno formation show `Eco` for either app, fix it immediately:
```
heroku ps:type basic -a testbench-api
heroku ps:type basic -a testbench-pipeline
```

## First deploy — dyno formation

Heroku only creates a `web` process type once a Procfile has actually been deployed.
Since this repo holds templates only (not the app code), the dyno type/scale below
could **not** be set at infra setup time — each service owner must run this once,
right after their first `git push heroku main`:

```
heroku ps:type basic -a testbench-api
heroku ps:scale web=1 -a testbench-api

heroku ps:type basic -a testbench-pipeline
heroku ps:scale web=1 -a testbench-pipeline
```

## Deploying

From each service's own repo (not this one):

```
heroku git:remote -a testbench-api        # or testbench-pipeline
git push heroku main
```

Alternatively, set up GitHub auto-deploy per app in the Heroku dashboard
(Deploy tab → GitHub → connect repo → enable automatic deploys).

## Database

A single Heroku Postgres **essential-0** ($5/mo) database is provisioned on
`testbench-api` and shared by both services — it is not reprovisioned on
`testbench-pipeline`. Both services read `DATABASE_URL` from their own config vars;
the value must match across both apps.

To rotate/regenerate credentials, use `heroku pg:credentials:rotate -a testbench-api`
and re-sync `DATABASE_URL` to testbench-pipeline afterward — the shared value will
change.

## Deploy templates

Each service's `app.json`, `Procfile`, and `.env.example` live under
`/deploy/<service-name>/` in this repo, for the service repo to copy or reference:

```
deploy/testbench-api/app.json
deploy/testbench-api/Procfile
deploy/testbench-api/.env.example
deploy/testbench-pipeline/app.json
deploy/testbench-pipeline/Procfile
deploy/testbench-pipeline/.env.example
```

## Config vars

### testbench-api

| Var | Status |
|---|---|
| `DATABASE_URL` | ✅ set (from Postgres provisioning) |
| `JWT_SECRET` | ✅ set (generated, `openssl rand -hex 32`) |
| `R2_ACCOUNT_ID` | ✅ set |
| `R2_ACCESS_KEY` | ✅ set |
| `CLOUDFLARE_WORKER_TRIGGER_URL` | ✅ set — `https://testbench-worker.bdev5592.workers.dev` (see [Cloudflare Worker](#cloudflare-worker) below) |

### testbench-pipeline

| Var | Status |
|---|---|
| `DATABASE_URL` | ✅ set (same value as testbench-api — shared DB) |
| `R2_ACCOUNT_ID` | ✅ set |
| `R2_ACCESS_KEY` | ✅ set |
| `R2_SECRET_KEY` | ✅ set |
| `OCR_API_KEY` | ✅ set — Azure AI Services (Document Intelligence) subscription key |
| `GROQ_API_KEY` | ✅ set |
| `VISION_LLM_API_KEY` | ✅ set — Azure OpenAI/AI Foundry key. **Not** the same value as `OCR_API_KEY` (see note below) |
| `CONFIDENCE_THRESHOLD` | ✅ set to `0.65` — calibrated against a real scanned exam script via Azure Document Intelligence (`prebuilt-layout`). See note below. |
| `AZURE_OCR_ENDPOINT` | ✅ set — not in the original var list, but required by Azure alongside the key. Format: `https://<resource>.cognitiveservices.azure.com/`. Used only for OCR (Document Intelligence) |
| `AZURE_OCR_REGION` | ✅ set — Azure region of the Cognitive Services resource (`southafricanorth`) |
| `AZURE_OPENAI_ENDPOINT` | ✅ set — not in the original var list. Azure AI Foundry Responses API endpoint: `https://testbench-llm-resource.services.ai.azure.com/openai/v1/responses`. Used for vision fallback and large-document overflow (separate resource from `AZURE_OCR_ENDPOINT` — don't conflate them, that was the original vision-path bug) |
| `AZURE_OPENAI_DEPLOYMENT` | ✅ set to `gpt-4.1-mini` — not in the original var list |

**Note on OCR vs. LLM providers:** two separate Azure resources are in play, easy to
confuse (this was the exact bug that broke the vision path originally). `AZURE_OCR_ENDPOINT`
+ `OCR_API_KEY` → Document Intelligence, OCR only. `AZURE_OPENAI_ENDPOINT` +
`VISION_LLM_API_KEY` → Azure OpenAI/AI Foundry, used for both the vision-fallback
extractor and the Groq-overflow text extractor. Don't merge these back into one
resource/key pair.

**Note on `CONFIDENCE_THRESHOLD`:** tested against a real scanned exam script.
Word-level OCR confidence split cleanly into two failure modes — genuine misreads
(handwriting, misoriented symbols) scored below ~0.6, while correctly-read technical
notation (subscripts, exponents, scientific units) often scored 0.7–0.9 despite being
accurate. `0.65` was chosen to catch the former without over-flagging the latter. This
was calibrated on one document — revisit once more real scanned submissions are
available.

### testbench-frontend (Next.js)

Not a Heroku app — these are build-time env vars for whoever owns the Next.js repo,
deployed on Vercel at `testbench.study` (custom domain purchased via Namecheap,
DNS records at Namecheap, not delegated to Vercel/Cloudflare nameservers - see
"Domain setup" below).

```
NEXT_PUBLIC_API_BASE_URL=https://api.testbench.study
NEXT_PUBLIC_R2_PUBLIC_BASE=https://pub-42861a7682c8422c854d3698618e8987.r2.dev
```

`NEXT_PUBLIC_R2_PUBLIC_BASE` still uses the `r2.dev` URL rather than a
`testbench.study` subdomain — R2 custom domains require the zone to be
Cloudflare-managed, which isn't the case here (DNS is on Namecheap). Not worth
migrating for this alone; see "Domain setup" below for the full tradeoff.

**Note on `NEXT_PUBLIC_R2_PUBLIC_BASE`:** the `testbench-uploads` R2 bucket has
public read access enabled (Cloudflare's `r2.dev` public bucket URL) — any object is
readable by anyone with its `storage_key`, no auth required. This matches the
original architecture (a public CDN in front of R2), but means the entire security
model rests on `storage_key` being an unguessable UUID, not on any access control.
Given uploaded files can contain student PII (names, matric numbers visible in
scanned exam photos), don't make `storage_key` predictable or sequential. If a
stronger model is ever needed, the alternative is presigned per-request GET URLs
instead of a permanent public bucket — more secure, more backend work.

## Cloudflare Worker

The glue between the two backend services — triggered by the Floater's
`POST /uploads/:id/start`, fire-and-forwards to Pipeline's `POST /internal/process`,
returns `202` immediately so neither service blocks on the other. Source lives in
this repo under `/worker`:

```
worker/wrangler.toml
worker/src/index.js
```

- **Live URL**: `https://testbench-worker.bdev5592.workers.dev`
- **Account subdomain**: `bdev5592.workers.dev` (registered once via the Cloudflare
  dashboard — a per-account, one-time step; not something `wrangler` can do for a
  brand-new account)
- No custom domain is wired up — `tech.seesunilag.com` is not currently a Cloudflare
  zone under this account, so `workers.dev` is used instead. Fine for this use case:
  it's an internal trigger URL, never exposed to students. Revisit if the domain is
  ever added to Cloudflare.

To redeploy after changing `worker/src/index.js`:

```
cd worker
CLOUDFLARE_API_TOKEN=<token> npx wrangler deploy
```

Token needs Workers Scripts edit permission, scoped to the account (the
"Edit Cloudflare Workers" template on the Cloudflare API Tokens page covers this).

Verified end-to-end (Worker → Pipeline → Postgres write) during setup — this test
also caught and fixed a real bug: Pipeline's Postgres connection wasn't configured
for SSL, which Heroku Postgres requires. See `testbench-backend`'s commit history
(`src/lib/db.ts`) for the fix.

## Domain setup

`testbench.study` was purchased via Namecheap. DNS is managed at **Namecheap**
(BasicDNS, Advanced DNS host records), **not** delegated to Cloudflare or Vercel
nameservers — a deliberate choice, not an oversight. See the reasoning below.

**Current DNS records (Namecheap → Advanced DNS):**

| Type | Host | Value | Purpose |
|---|---|---|---|
| CNAME | `api` | `philosophical-asparagus-hevi7siqfx4397dl16hr7iin.herokudns.com` | Routes to testbench-api |
| A | `@` | `216.198.79.1` | Vercel apex (frontend root) |
| CNAME | `www` | `cname.vercel-dns.com` | Vercel (frontend) |

`testbench.study` 308-redirects to `www.testbench.study` (Vercel's own config, not
a Namecheap redirect record — the domain was briefly misconfigured with a
Namecheap "URL Redirect Record" on `@` during setup, which conflicted with Vercel's
verification; that was removed in favor of the plain A record above).

**Heroku side:** `api.testbench.study` added via `heroku domains:add -a
testbench-api`, with Automatic Certificate Management (`heroku certs:auto:enable`)
providing the SSL cert — ACM is **off by default** on new custom domains, easy to
miss (caused an initial SSL verification failure until enabled).

**Why DNS stays on Namecheap instead of moving to Cloudflare:** the only things
that actually require a Cloudflare-managed zone are custom domains for the
existing Worker (currently on `workers.dev`) and the R2 public bucket (currently
on `r2.dev`) — neither is user-facing (the Worker is an internal trigger endpoint,
the R2 URL only appears in `<img src>` attributes). Migrating DNS is optional
polish, not a functional requirement; revisit only if those URLs specifically need
to look branded later.

## Budget

| Item | Cost |
|---|---|
| testbench-api — Basic dyno | $7/mo |
| testbench-pipeline — Basic dyno | $7/mo |
| Postgres essential-0 (on testbench-api) | $5/mo |
| **Total** | **$19/mo** |

Note: this exceeds the original $13/mo GitHub Student credit estimate, which had only
budgeted for one Basic dyno. Confirmed and accepted at setup time — both apps need to
be always-on (Basic), which requires two dynos. No worker dynos or other paid add-ons
are provisioned beyond what's listed above; keep it that way unless the budget is
revisited.
