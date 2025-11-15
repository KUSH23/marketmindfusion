

## MarketMind Fusion – Product Requirements & Handover

Live app: https://marketmindfusion.github.io

This document gives a concise, handover‑ready overview of the product, architecture, and expectations for future development. All installation, environment, and deployment details are collected after the app summary.

---

### 1. Product Overview

**Product name:** MarketMind Fusion  
**One‑liner:** An AI‑powered research and marketing studio that turns product ideas into research plans, surveys, insights, personas, and campaign assets.

MarketMind Fusion combines Supabase, OpenAI, and a React/Vite frontend to help product and marketing teams go from product description to research and campaign outputs in minutes rather than weeks.

---

### 2. Installation & Detailed Setup

This section explains how to get a fresh environment running, with links to the tools we rely on.

#### 2.1. Prerequisites

- Node.js (LTS) – download from https://nodejs.org  
- npm (bundled with Node.js)  
- Git – https://git-scm.com  
- Supabase account and project – https://supabase.com  
- Supabase CLI – https://supabase.com/docs/guides/cli  
- OpenAI API key – https://platform.openai.com

Optional but recommended:

- VS Code – https://code.visualstudio.com  
- GitHub account for collaboration – https://github.com

#### 2.2. Clone and Install Dependencies

```bash
git clone https://github.com/KUSH23/marketmindfusion.git
cd marketmindfusion

# install JS/TS dependencies
npm install
```

If you are using `bun`, you can alternatively run:

```bash
bun install
```

#### 2.3. Frontend Environment Variables (`.env`)

Create a `.env` file in the project root (same level as `package.json`) and define:

```bash
VITE_SUPABASE_URL=your_supabase_project_url
VITE_SUPABASE_ANON_KEY=your_supabase_anon_public_key
```

How to find these:

- In the Supabase dashboard, go to **Project Settings → API**.  
- Copy the **Project URL** into `VITE_SUPABASE_URL`.  
- Copy the **anon public** key into `VITE_SUPABASE_ANON_KEY`.

You can add additional `VITE_*` keys later if needed by the frontend.

#### 2.4. Link to Your Supabase Project & Run Migrations

From the project root:

```bash
supabase login
```

```bash
supabase link
```

Follow the CLI instructions (you will need your project reference from the Supabase dashboard). Once linked, apply the existing database schema:

```bash
supabase db push
```

This runs the SQL migrations in `supabase/migrations/` against your project and sets up the tables expected by the app.

For more details, see: https://supabase.com/docs/guides/cli/local-development

#### 2.5. Configure Supabase Function Environment Variables

In the Supabase dashboard:

1. Go to **Project → Edge Functions → Secrets**.  
2. Add the following variables:

	- `OPENAI_API_KEY` – your key from https://platform.openai.com/account/api-keys
	- (Optional) `OPENAI_MODEL`, `OPENAI_MAX_TOKENS`, `OPENAI_RETRY_DELAY_MS`, `OPENAI_FALLBACK_MODEL`  
	- `RESEND_API_KEY` – for using send survey email (https://resend.com/)  

3. Save and redeploy functions after changes.

You can also use a local `.env` inside `supabase/` when running functions locally with `supabase functions serve`. See the Supabase docs: https://supabase.com/docs/guides/functions

#### 2.6. Running the Frontend Locally

```bash
npm run dev
```

This starts the Vite dev server (by default on `http://localhost:5173`). The app will use the Supabase project configured by your `VITE_*` variables.

#### 2.7. Running Supabase Edge Functions Locally

From the `supabase/` directory:

```bash
cd supabase
supabase functions serve analyze-product
```

You can swap `analyze-product` for any other function folder name (`generate-hypotheses`, `generate-survey-questions`, etc.). The CLI will expose a local URL you can point the frontend to during development if needed.

#### 2.8. Deploying Edge Functions

```bash
cd supabase
supabase functions deploy analyze-product
supabase functions deploy generate-hypotheses
# ...deploy other functions as needed
```

Documentation: https://supabase.com/docs/guides/functions/deploy

#### 2.9. Building the Frontend for Production

To create a static production build (used for GitHub Pages):

```bash
npm run build
```

This outputs a static bundle (by default into `dist/`). You can then publish this to GitHub Pages or any static hosting. The current live deployment is at https://marketmindfusion.github.io.

For Vite deployment details, see: https://vitejs.dev/guide/static-deploy.html

---

### 3. Prompt Library

This section collects the prompt templates and system/user message patterns used in the Supabase Edge Functions. Keep these templates in sync with the `supabase/functions/*/index.ts` files. Use the variables shown in each template when invoking via the frontend.

Full canonical prompts and raw templates: `prompts.md`

- **analyze-product**
	- System prompt (role: system): marketing intelligence analyst asking for a strict JSON object with keys: `category`, `targetAudience`, `trends`, `positioning`, `suggestedName`.
	- User prompt (role: user): the raw `productDescription` text.
	- Response format: valid JSON, example structure:
		{
			"category": "...",
			"targetAudience": "...",
			"trends": "...",
			"positioning": "...",
			"suggestedName": "..."
		}
	- Notes: Always validate the JSON and required fields (`category`, `targetAudience`) before persisting.

- **generate-hypotheses**
	- System prompt: two modes (`expert` vs default). Both require EXACT JSON with `hypotheses` array; each hypothesis contains `statement`, `method`, `methodTechnical`, `decisionRule`, `confidence`.
	- User prompt pattern:
		`Product: ${productName}\n\nDescription: ${productDescription}\n\nGenerate 3 hypotheses.`
	- Notes: Mode controls tone/rigor. Function normalizes fields (e.g., `idea` -> `statement`) after parsing.

- **generate-survey-questions**
	- System prompt: expert survey designer with allowed question types: `text`, `multiple_choice`, `rating` and guidelines for question ordering and option counts.
	- User prompt includes `surveyTitle`, optional `surveyDescription`, optional `personaInfo`, and `questionCount`.
	- Response format: JSON `{ "questions": [ { "text", "type", "required", "options"? } ] }`.
	- Notes: For `multiple_choice` include 4–6 options; for `rating` include endpoints.

- **generate-research-plan**
	- System prompt: marketing research strategist asking for strict JSON plan with `targetAudience`, `sample`, `methodology`, `timeline`, `budget`.
	- User prompt composes `productName`, `productDescription`, and aggregated `hypotheses` statements.
	- Notes: This function may use larger token limits and shared OpenAI helper; check token/finish_reason for truncated responses.

- **generate-competitive-analysis**
	- System prompt: competitive intelligence analyst expecting JSON with `analyses` array where each item has SWOT lists, `positioning`, and `recommendations` object.
	- User prompt provides `productName`, `productDescription`, and `competitors` (or instructs AI to pick top competitors).
	- Notes: Uses tool-calling style; ensure `recommendations` exists for every analysis row before DB insertion.

- **generate-marketing-copy**
	- System prompt: marketing copywriter; requires JSON `variants` array with `variant_name`, `headline`, `body_text`, and `cta`.
	- User prompt includes `productName`, `productDescription`, `targetAudience`, and `contentType` (ad_copy, email, landing_page, social_post, blog_post).
	- Notes: Each variant should use distinct angles (e.g., problem-focused, benefit-focused, social-proof).

- **generate-personas**
	- System prompt: market researcher expecting `personas` array with detailed demographic and psychographic fields.
	- User prompt includes `productName`, `productDescription`, and `researchData` (JSON string of research findings).
	- Notes: The function sanitizes personas to ensure required fields have defaults.

- **analyze-survey-responses**
	- System prompt: sentiment analysis expert; returns overall sentiment, themes, pain_points, positive_highlights, actionable_insights, and per-response sentiment.
	- User prompt consists of enumerated `textResponses` to analyze.
	- Response format: JSON with keys `overall_sentiment`, `sentiment_score`, `key_themes`, `pain_points`, `positive_highlights`, `actionable_insights`, `individual_sentiments`.

- **predict-ab-test**
	- System prompt: marketing analyst/data scientist. Expects JSON with `predicted_winner`, `confidence_score`, `predicted_metrics` for both variants, `key_factors`, and `recommendations`.
	- User prompt contains `testName`, serialized `variantA`, `variantB`, `targetAudience`, and optional `historicalData`.

- **generate-campaign-image**
	- Prompt (images API): combination of `imageTypePrompts` (hero, ad_banner, social_media, product_shot) and `styleModifiers` (modern, vibrant, professional, creative), plus optional `brandColors`.
	- Returned payload includes the original `prompt` (stored for audit) and Base64 image data.

- **match-persona-contacts**
	- System prompt: matching expert with clear scoring rules (prioritize psychographics/interests over demographics). Expect an array of `{ contact_id, match_score, reasons }`.
	- User prompt includes full persona details and a list of contacts (each with demographics, interests, behavior).
	- Notes: Function filters and enriches matches; keep `minMatchScore` configurable.

General notes for the Prompt Library:

- Store canonical templates in code and in this README to make future audits easier.  
- Respect the strict JSON response contracts — the Edge Functions parse and JSON.parse the AI content directly.  
- If you need to change formats, update both the function and the README template together.

---

### 4. Core Use Cases

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

### 5. Users & Personas (High‑Level)

1. **Product Manager / Founder** – wants quick validation and evidence for product decisions.
2. **Marketing / Growth Manager** – needs audience understanding, angles, and assets to run campaigns.
3. **Research‑curious Generalist** – wants structure and guidance to run “good enough” research.

The interface is optimized for these users: guided flows, clear CTAs, and shareable outputs.

---

### 6. Frontend Architecture (React + Vite)

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

### 7. Backend Architecture (Supabase + Edge Functions)

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

### 8. Known Issues & Limitations

- **Edge Function returned a non-2xx status code**: Many Supabase Edge Functions depend on OpenAI. If the model name is invalid, the account is rate-limited, or the response exceeds token limits, OpenAI can return 4xx/5xx responses. These surface in the frontend as a generic non-2xx error.
	- Recommended actions:
		- Check that `OPENAI_API_KEY` is valid and has quota.
		- Confirm the model configured (for example, `gpt-4.1-mini`, `gpt-4o-mini`, `gpt-image-1`) exists and is enabled for your account.
		- Reduce prompt size or complexity if you hit token length issues.
		- Inspect Supabase Edge Function logs for the exact `OpenAI API error: <status>` message.
- **Strict JSON contracts**: Most functions call `JSON.parse` on the model output and expect specific fields. If the model returns extra text, comments, or malformed JSON, the function will throw a 500 error.
	- Recommended actions:
		- Keep prompts unchanged unless you also update the parsing logic.
		- If you change the expected JSON shape, update both the function code and `prompts.md` / Prompt Library.
		- When debugging, log the raw `content` string from the model to see what broke.
- **Schema constraints vs AI output**: Some tables (for example, competitive analysis, personas) expect non-null fields such as `recommendations`. If the AI omits them, inserts/updates may fail.
	- Recommended actions:
		- Add server-side defaulting/sanitization before inserts (for example, ensure `recommendations` is at least an object with default keys).
		- Relax DB constraints only if you are comfortable with partial data.

Users are advised to review the model names, token limits, and JSON contracts in the Supabase functions before deploying to production to avoid these errors.

---

### 9. Non‑Functional Expectations

- **Performance** – typical AI requests should complete in a few seconds; UI must show clear loading and error states.
- **Reliability** – Edge Functions log errors and return JSON with explicit `error` messages on failure.
- **Security** – no secrets in the frontend bundle or repo; all sensitive keys in Supabase environments; RLS enforced on Supabase tables.
- **Maintainability** – one Edge Function per responsibility; React pages and components organized by feature area.

---

### 10. Handover Notes

- This repository already contains:
	- A working React/Vite UI with multiple research and marketing workflows.
	- A Supabase project structure with migrations.
	- A suite of AI‑backed Supabase Edge Functions.
- Future work can focus on:
	- Hardening error handling and rate limiting around OpenAI.  
	- Extending analytics and reporting (e.g., richer `DataInsights`).  
	- Collaboration features and billing if desired.

For questions or further context, review the `src/pages/` and `supabase/functions/` directories, then explore the live app at https://marketmindfusion.github.io.




