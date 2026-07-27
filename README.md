# beautyagent-ai

BeautyAgent AI project workspace.

## Agent Role

#BeautyAgent AI

AI-powered marketing copy generation and compliance review for indie beauty brands.

#Agent Role

BeautyAgent AI turns a short marketing brief into channel-specific draft copy for supported brands, then runs a two-agent pipeline before returning results:

#Brand Voice Agent — evaluates the draft against the brand's enriched voice profile and returns a verdict (ON_VOICE / DRIFTED) plus a confidence score.
#Compliance Agent — audits the copy and marketer brief for FDA/MoCRA cosmetic-claim risk. Only runs if the Brand Voice Agent passes (confidence ≥ 0.75).
Orchestrator — merges both verdicts and routes each channel result to one of three card states: Compliant, Needs a Tweak, or Needs Sign-Off.

A deterministic compliance backstop re-runs on every completed output regardless of agent verdicts — no content ships unaudited.

#Problem It Solves

Beauty teams need faster first drafts without losing brand fit or accidentally making risky cosmetic claims. BeautyAgent AI turns a brief into TikTok, Instagram, and email copy while surfacing off-voice or compliance-sensitive output before it reaches the marketer's clipboard.

#How it Works:
Given a marketing brief, the agent decides whether copy is on-brand and compliant through a sequential two-agent pipeline, then surfaces one of three actionable verdicts: Compliant, Needs a Tweak, or Needs Sign-Off.

#Week 1 → Week 2: What Changed
Area	Week 1 (Agent 1.0)	Week 2 (Agent 2.0)
Card states	PASSED / FAILED	Compliant / Needs a Tweak / Needs Sign-Off
Voice evaluation	None — draft agent self-judged	Dedicated Brand Voice Agent runs first
Routing	Binary compliance check only	Confidence-based orchestrator (0.75 threshold)
Escalation trigger	n/a	"voice" or "compliance" surfaced per card
API response fields	compliance_status, flagged_phrases, explanation, final_safe_output	+ voice_status, voice_confidence, voice_reason, compliance_confidence, escalation_trigger
Model path	OpenRouter → Anthropic	Direct Anthropic API (OpenRouter retained as fallback)
Brand voice configs	Four-adjective placeholders	Enriched per-brand profiles with channel specs, vocabulary guardrails, ON_VOICE / DRIFTED examples
Eval scripts	Red-team compliance suite	+ Brand voice calibration set (/evaluate-voice endpoint)

#BEAUTYAGENT_API_CONTRACT.md is the source of truth for the full request/response schema. Week 2 additions live in Section 9; Sections 1–8 are unchanged.

#Tools Used
Frontend: React/Vite dashboard — owned by Jillian, deployed on Vercel.
Backend: FastAPI /generate and /evaluate-voice endpoints — owned by Christopher, deployed on Render.
Agent tools: draft generation, Brand Voice Agent (check_brand_voice), deterministic compliance checker, compliance LLM audit, orchestrator routing logic, red-team evals, demo smoke tests.
LLM providers: Anthropic (direct, primary path via LiteLLM) for both Sonnet (copy generation + voice evaluation) and Haiku (compliance audit). OpenRouter retained as fallback.
Shared contract files: BEAUTYAGENT_API_CONTRACT.md, docs/, shared/live-ui-samples/.
How to Run

Backend (from repo root):

bash
uvicorn app.main:app --reload --app-dir backend

Frontend (from frontend/):

bash
npm install
npm run dev
Project Structure
beautyagent-ai/
├── frontend/
├── backend/
│   └── app/
│       ├── tools/
│       ├── agent/
│       ├── prompts.py
│       └── data/
│           ├── brand_voice_tower28.md
│           └── brand_voice_halfmagic.md
├── shared/
│   └── live-ui-samples/
├── docs/
├── evals/
│   └── brand_voice_calibration_cases.json
├── BEAUTYAGENT_API_CONTRACT.md
├── DECISIONS.md
├── README.md
├── .gitignore
└── .env.example
Team

Built by two cooperating builders:

Jillian — frontend, UI, brand voice .md files, GitHub PR merges, demo prep.
Christopher — backend, agent logic, tools, prompt engineering, Render deployment, backend evals.

Both builders are repo collaborators with Write access. Branch protection is on main; all changes go through a PR with the non-author approving.

See docs/TEAM_WORKFLOW.md for the full ownership convention. Backend changes that affect the /generate or /evaluate-voice response shape require contract coordination before building — see BEAUTYAGENT_API_CONTRACT.md and DECISIONS.md.

Backend Quick Checks

Run these from the repo root before pushing backend changes:

bash
# Unit tests
python -m unittest discover -s backend/tests -v

# Full red-team compliance eval (mock brand voice, compact output)
python backend/scripts/run_red_team_eval.py --mock-brand-voice --compact

Timeout-friendly eval chunks:

bash
python backend/scripts/run_red_team_eval.py --start 1 --end 5 --mock-brand-voice --compact
python backend/scripts/run_red_team_eval.py --case-id channel_specific_risky_instruction --mock-brand-voice --compact

Full Week 2 demo smoke test:

bash
python backend/scripts/run_demo_smoke.py
# Use --skip-live-brand-voice for a token-free local check

Brand voice calibration evals (required gate before any brand voice file change):

bash
python backend/scripts/run_brand_voice_eval.py --compact
python backend/scripts/run_brand_voice_eval.py --start 1 --end 3 --compact

Optional live LLM smoke tests:

bash
python backend/scripts/smoke_openrouter.py
python backend/scripts/smoke_generate_live.py

Live eval and smoke scripts only run when USE_LLM_DRAFTING=true and either ANTHROPIC_API_KEY or OPENROUTER_API_KEY is set in the backend environment — otherwise they exit as skipped. Token/cost usage is printed per run and accumulated in backend/logs/llm_usage_local.jsonl (gitignored).

Backend Behavior Notes
/generate is one request → one full response. No streaming, polling, websocket, or mid-request progress endpoint.
/evaluate-voice accepts a brand, channel, and draft directly and returns a voice verdict without running the full pipeline. Required for validating DRIFTED cases — the generation flow rewrites the brief before evaluation, so DRIFTED inputs can't be validated through the UI.
Agent sequencing is a product decision. Brand Voice Agent runs before compliance. Running compliance first flattens brand voice; the current order is intentional and documented in DECISIONS.md.
LLM provider failures fall back to deterministic drafting. Fallback drafts still pass through check_compliance plus the final deterministic safety backstop.
TikTok Hook / Script / CTA and Email Subject / Body are formatted inside raw_draft and final_safe_output — they are not separate API fields.
Brief-level compliance violations can return FAILED even when the visible draft is clean. In that case flagged_phrases and explanation point back to risky marketer brief language.
Brand Voice Agent failures (timeout, malformed JSON) default to voice_confidence: 0.0 and route to Needs Sign-Off — fail safe, not fail open. Sibling channels complete normally.
The 0.75 confidence threshold is hardcoded for Agent 2.0. Calibrate using the 6-case near-miss set via run_brand_voice_eval.py before any demo or deployment.
Deployment prep lives in backend/DEPLOYMENT.md. The repo includes render.yaml for a Render web service Blueprint.

For mock-to-live frontend wiring, see docs/LIVE_ENDPOINT_MAPPING.md and the sample responses in shared/live-ui-samples/.

Confirmed Demo Scenarios

Scenario A — Half Magic, three Needs a Tweak cards:

Brand: Half Magic · Product: MAGIC DRIP Glitter Lipgloss · All three channels
Brief: "Open with: clinically proven to boost lip fullness. Lead with that claim as the hook, then describe the sparkle payoff."

Scenario B — Tower 28, mixed card states:

Brand: Tower 28 · Product: SOS Daily Rescue Facial Spray · All three channels · Actives: Hypochlorous Acid
Brief: "An elevated, sophisticated facial mist for the discerning consumer who demands nothing less than clinical perfection in their skincare ritual."
Expected: TikTok Compliant, Instagram Needs Sign-Off, Email Compliant
Scope
Brands: Tower 28, Half Magic
Channels: TikTok, Instagram, Email
Card states: Compliant, Needs a Tweak, Needs Sign-Off
Config: Static JSON / Markdown only — no persistent database
