

## MarketMind Fusion – Product Requirements & Handover

Live app: https://marketmindfusion.github.io

This document gives a concise, handover‑ready overview of the product, architecture, and expectations for future development.

---

### 1. Product Overview

**Product name:** MarketMind Fusion  
**One‑liner:** An AI‑powered research and marketing studio that turns product ideas into research plans, surveys, insights, personas, and campaign assets.

MarketMind Fusion combines Supabase, OpenAI, and a React/Vite frontend to help product and marketing teams go from product description to research and campaign outputs in minutes rather than weeks.

---

### 2. Core Use Cases

- Capture a new product or feature idea and get:
	- Structured product analysis (category, target audience, trends, positioning).
	- Testable marketing hypotheses and decision rules.
	- A full research plan (target audience, sample, methodology, timeline, budget).
- Design research instruments:
	- AI‑generated survey questions with types and options.
	- Email copy to distribute surveys.
- Turn responses into insights:
	- Sentiment and theme analysis from free‑text responses.
	- Personas derived from survey data and hypotheses.
	- Competitive analysis and SWOT.
- Support experimentation and campaigns:
	- A/B test outcome prediction.
	- Marketing copy variants (ads, emails, landing pages, social posts).
	- Campaign imagery for hero banners, ads, product shots.

---

### 3. Users & Personas (High‑Level)

1. **Product Manager / Founder** – wants quick validation and evidence for product decisions.
2. **Marketing / Growth Manager** – needs audience understanding, angles, and assets to run campaigns.
3. **Research‑curious Generalist** – wants structure and guidance to run “good enough” research.

The interface is optimized for these users: guided flows, clear CTAs, and shareable outputs.

---

### 4. Frontend Architecture (React + Vite)

**Tech:** React, TypeScript, Vite, Tailwind/shadcn‑style UI components.

Key directories under `src/`:

- `App.tsx`, `main.tsx` – app shell and routing.
- `pages/` – each major workflow:
	- `Index.tsx` – landing/home.
	- `Auth.tsx` – authentication entry.
	- `Dashboard.tsx` – project overview.
	- `ProductInput.tsx` – product description input.
	- `Hypothesis.tsx` – view/manage hypotheses.
	- `ResearchPlan.tsx` – generated research plans.
	- `SurveyAutomation.tsx` – survey question generation.
	- `SurveyResponse.tsx`, `SurveyResponseHandler.tsx` – response viewing/handling.
	- `DataInsights.tsx` – analysis/insights surface.
	- `ABTestPredictor.tsx` – A/B test prediction experience.
	- `MarketingStudio.tsx`, `Analysis.tsx`, `Report.tsx`, `Instruments.tsx` – additional analysis and campaign tools.
- `components/` – layout and shared UI:
	- `Hero.tsx`, `Features.tsx` – marketing/landing components.
	- `ProtectedRoute.tsx` – route guard using Supabase auth.
	- `ModeToggle.tsx`, `NavLink.tsx` – theme + navigation.
	- `components/ui/` – shadcn‑style primitives (accordion, dialog, dropdown, table, chart, form inputs, toasts, etc.).
- `hooks/` – custom hooks (`use-mobile`, `use-toast`).
- `integrations/supabase/` – `client.ts` (browser Supabase client) and `types.ts` (generated database types).
- `lib/utils.ts` – shared helpers (e.g., className utilities).

Routing is page‑based, with access to certain routes guarded by `ProtectedRoute` once a valid Supabase session is present.

---

### 5. Backend Architecture (Supabase + Edge Functions)

**Supabase project** lives in `supabase/`:

- `config.toml` – project configuration (project ref, connection info).
- `migrations/` – SQL migrations defining tables for projects, personas, contacts, surveys, responses, competitive analyses, etc.
- `functions/` – Supabase Edge Functions (Deno) providing AI‑backed services.

Key Edge Functions (each has an `index.ts`):

- `analyze-product` – turns a product description into structured analysis (category, target audience, trends, positioning, suggested name). 
- `generate-hypotheses` – generates marketing/research hypotheses with methods, decision rules, and confidence.
- `generate-research-plan` – produces a detailed research plan (sample, methodology, timeline, budget).
- `generate-survey-questions` – creates survey questions with types and options.
- `analyze-survey-responses` – runs sentiment and theme analysis on free‑text responses, returning structured insights.
- `generate-personas` – builds personas from research data (demographics, psychographics, pains, goals, channels, buying behavior).
- `generate-competitive-analysis` – generates SWOT‑style analyses and recommendations per competitor.
- `generate-marketing-copy` – outputs multiple variants of marketing copy for different formats.
- `generate-campaign-image` – creates campaign imagery using OpenAI images API.
- `predict-ab-test` – predicts which A/B variant will win and provides rationale and metrics.
- `match-persona-contacts` – scores contacts against a persona and returns ranked matches.
- `duplicate-project` – utility for cloning project setups.
- `send-survey-emails` – back‑end support for survey outreach.

All AI functions:

- Import Deno HTTP server utilities and an XHR polyfill.
- Read `OPENAI_API_KEY` (and optional model/token env vars) from the environment.
- Accept JSON payloads from the frontend, call OpenAI, enforce strict JSON response formats, validate the shape, and return structured results.

Supabase Auth is used for login; the frontend calls these Edge Functions via HTTPS endpoints exposed by Supabase.

---

### 6. Environment & Configuration

**Frontend (`.env` for Vite)**

Typical variables (names may vary in your setup):

- `VITE_SUPABASE_URL`
- `VITE_SUPABASE_ANON_KEY`
- Any other `VITE_*` keys needed by the client.

**Supabase Functions (project settings → Functions → Environment Variables)**

- `OPENAI_API_KEY` – required for all AI Edge Functions.
- Optional per‑function or global knobs (if used):
	- `OPENAI_MODEL`
	- `OPENAI_MAX_TOKENS`
	- `OPENAI_RETRY_DELAY_MS`
	- `OPENAI_FALLBACK_MODEL`
- `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` – for functions such as `match-persona-contacts` that need privileged DB access.

Secrets are **not** checked into the repo. They must be set in Supabase dashboard and/or local `.env` files for CLI testing.

---

### 7. Development & Setup (for Engineers)

**Prerequisites**

- Node.js (LTS) and npm.  
- Supabase CLI (`npm install -g supabase`).  
- An OpenAI API key.  
- A Supabase project.

**Getting Started**

```bash
git clone https://github.com/KUSH23/marketmindfusion.git
cd marketmindfusion

# install dependencies
npm install

# configure frontend env vars (.env)
# set VITE_SUPABASE_URL, VITE_SUPABASE_ANON_KEY, etc.

# link to your Supabase project (one‑time)
supabase link

# apply database migrations
supabase db push
```

**Running locally**

```bash
# start Vite dev server
npm run dev

# run a specific Edge Function locally (from project root)
cd supabase
supabase functions serve analyze-product
```

**Deploying Edge Functions**

```bash
cd supabase
supabase functions deploy analyze-product
supabase functions deploy generate-hypotheses
# ...deploy other functions as needed
```

The frontend is deployed separately via GitHub Pages at https://marketmindfusion.github.io.

---

### 8. Non‑Functional Expectations

- **Performance** – typical AI requests should complete in a few seconds; UI must show clear loading and error states.
- **Reliability** – Edge Functions log errors and return JSON with explicit `error` messages on failure.
- **Security** – no secrets in the frontend bundle or repo; all sensitive keys in Supabase environments; RLS enforced on Supabase tables.
- **Maintainability** – one Edge Function per responsibility; React pages and components organized by feature area.

---

### 9. Handover Notes

- This repository already contains:
	- A working React/Vite UI with multiple research and marketing workflows.
	- A Supabase project structure with migrations.
	- A suite of AI‑backed Supabase Edge Functions.
- Future work can focus on:
	- Hardening error handling and rate limiting around OpenAI.  
	- Extending analytics and reporting (e.g., richer `DataInsights`).  
	- Collaboration features and billing if desired.

For questions or further context, review the `src/pages/` and `supabase/functions/` directories, then explore the live app at https://marketmindfusion.github.io.



