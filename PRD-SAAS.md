# PRD: CPC Exam Prep SaaS — Commercial Product

**Codename:** CPCPro (working title)
**Date:** 2026-03-07
**Based on:** ud4090v/cpc-prep (Dee's personal app — forked, not modified)

---

## Vision

An AI-native CPC exam prep platform that undercuts every existing solution on price while delivering a superior, adaptive learning experience. Target: the ~70,000 AAPC exam candidates annually who are paying $500-2,000 for prep courses that don't adapt to them.

**Core thesis:** Static content goes stale. Pre-built question banks get leaked. AI-generated scenarios are infinite, fresh, and impossible to pirate.

---

## Market

| Competitor | Price | Weakness |
|---|---|---|
| AAPC Official Courses | $500-2,000 | Expensive, not adaptive |
| Practicode | $149/year | Coding only, no study mode |
| Uworld (med) | $299/year | Not CPC-specific |
| Quizlet (medical) | $35/year | No clinical scenarios, no AI |
| **CPCPro** | **$79/year or $29/mo** | AI-native, adaptive, affordable |

**TAM:** ~70,000 CPC exam takers/year × $79 = ~$5.5M if 10% penetration

---

## Product Tiers

### Free Tier
- Medical terminology flash cards (583 cards, all body systems)
- Basic study mode (MC + type answer)
- 10 practice questions/day cap
- No progress persistence across devices
- Goal: hook, convert

### Pro Tier — $29/month or $79/year
- Everything in Free
- Unlimited AI-generated clinical scenarios
- Full 100-question timed exam simulation (4-hour mode)
- Pacing coach (2.4 min/question alerts, pace warnings)
- Comprehensive post-exam analysis with AI coaching
- Progress tracking across devices (Supabase, auth required)
- Weak area identification + targeted drill sessions
- All 17 CPC knowledge domains covered
- Pass/fail at 70% threshold with score breakdown

### Bundle — $149/year
- Everything in Pro
- "AI Tutor" chat mode — ask any coding question, get explained answers
- Book recommendations + affiliate links (CPT, ICD-10-CM, HCPCS)
- Downloadable study guides (PDF)
- Priority support

---

## Core Features

### 1. Terminology Flash Cards (from v1)
- 583 cards across 13 body systems
- MC + type answer
- AI feedback
- Filters by system/category/difficulty
- ✅ Already built — carry over

### 2. AI Clinical Scenario Engine (NEW — the differentiator)

**How it works:**
- User selects topic (E/M, Surgery, Radiology, ICD-10, etc.)
- GPT-4o generates a clinical vignette + 4 answer choices
- Scenarios include realistic patient presentations, procedure descriptions, diagnosis situations
- Each question is unique — never repeated
- After answer: detailed explanation of why each option is right/wrong
- CPT/ICD-10 code validation layer (cross-reference against code database)

**Prompt architecture:**
```
System: You are a CPC exam question writer with 10+ years of experience.
Generate exam-style questions following AAPC format. Always include:
- Realistic clinical scenario (2-3 sentences)
- 4 answer choices (A-D), only one correct
- For coding questions: use real CPT/ICD-10 codes
- Explanation of correct answer + why others are wrong
Output as structured JSON.
```

**Anti-hallucination layer:**
- Maintain a validated code reference (CPT/ICD-10 code list)
- Post-process generated questions: flag any codes not in reference
- If flagged: regenerate or mark as "unverified — check your code book"

**Topics covered:**
- Medical Terminology & Anatomy
- ICD-10-CM Diagnosis Coding
- CPT Surgery (10000-69999 series)
- Evaluation & Management (E/M)
- Anesthesia
- Radiology
- Pathology & Laboratory
- Medicine
- HCPCS Level II
- Modifiers (25, 59, 51, 22, etc.)
- Coding Guidelines & Compliance
- Medical Ethics & Regulations

### 3. Timed Exam Simulation

**Setup screen:**
- Mode: Full Exam (100 q, 4h) | Half Exam (50 q, 2h) | Quick Practice (25 q, 1h)
- Topic mix: Auto (weighted by real exam) | Custom
- Difficulty: Standard | Hard

**During exam:**
- Countdown timer (4:00:00)
- Per-question timer with pacing indicator (green/yellow/red)
- Flag questions for review
- Navigation panel (jump to any question)
- No feedback during exam

**Results screen:**
- Score: X/100 (Pass ≥ 70 / Fail < 70)
- Time breakdown: avg per question, slowest 5 questions
- Per-topic breakdown (which domains strong/weak)
- AI coaching summary: "You struggled with E/M coding. Focus on...")
- Full question review with explanations
- "Retake weak areas" button

### 4. AI Tutor Mode (Bundle tier)
- Chat interface
- "Explain CPT code 99213 to me"
- "What's the difference between modifier 25 and 57?"
- "Walk me through coding this scenario: [paste text]"
- Grounded in CPC knowledge base
- Not a general chatbot — scoped to medical coding

### 5. Auth + Subscriptions
- Supabase Auth (email/password + Google OAuth)
- Stripe for subscriptions (monthly + annual)
- Free tier: no credit card required
- Pro/Bundle: Stripe checkout
- Usage limits enforced server-side

### 6. Book Recommendations / Affiliate
- "You'll need these for the exam" section
- Amazon affiliate links: CPT Professional, ICD-10-CM, HCPCS
- Disclaimer: "The CPC exam is open book. These are the books allowed in the exam room."
- Potential: direct resell at markup (future)

---

## Architecture

### Stack
- **Frontend:** Next.js 14 (App Router), TypeScript, Tailwind CSS
- **Backend:** Next.js API routes (Vercel serverless)
- **Database:** Supabase (PostgreSQL + Auth)
- **AI:** OpenAI GPT-4o (scenarios, analysis, tutor) + GPT-4o-mini (fast feedback)
- **Payments:** Stripe (subscriptions)
- **Email:** Resend (welcome, receipts, study reminders)
- **Deployment:** Vercel (BlackRabbitDev team)

### Supabase Schema (additions over v1)

```sql
-- Users (extends Supabase auth.users)
CREATE TABLE profiles (
  id uuid REFERENCES auth.users PRIMARY KEY,
  tier text DEFAULT 'free',           -- free | pro | bundle
  stripe_customer_id text,
  stripe_subscription_id text,
  subscription_status text,           -- active | canceled | past_due
  created_at timestamptz DEFAULT now()
);

-- Generated scenarios cache (avoid regenerating same scenario)
CREATE TABLE scenarios (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  topic text NOT NULL,
  difficulty text,
  question text NOT NULL,
  options jsonb NOT NULL,             -- [{text, isCorrect, explanation}]
  correct_index int,
  explanation text,
  codes_used jsonb,                   -- CPT/ICD codes referenced
  validated boolean DEFAULT false,
  created_at timestamptz DEFAULT now()
);

-- Exam sessions (extends v1 sessions)
ALTER TABLE sessions ADD COLUMN exam_mode text;  -- terminology | scenario | mixed | full_exam
ALTER TABLE sessions ADD COLUMN tier_at_time text;

-- Usage tracking (for free tier limits)
CREATE TABLE daily_usage (
  user_id uuid REFERENCES auth.users,
  date date,
  scenarios_generated int DEFAULT 0,
  questions_answered int DEFAULT 0,
  PRIMARY KEY (user_id, date)
);
```

### API Routes (additions)
- `POST /api/scenario/generate` — generate clinical scenario (GPT-4o)
- `POST /api/scenario/validate` — validate codes in scenario
- `GET /api/scenario/topics` — list available topics
- `POST /api/tutor` — AI tutor chat (Bundle tier)
- `POST /api/stripe/checkout` — create Stripe checkout session
- `POST /api/stripe/webhook` — handle subscription events
- `GET /api/subscription` — get current user's tier/status
- `POST /api/auth/register` — register new user

### Middleware
- Auth check on protected routes
- Tier check on Pro/Bundle features
- Rate limiting on AI endpoints
- Usage tracking for free tier

---

## Pages / Routes

```
/                    — Landing page (marketing)
/signup              — Register
/login               — Login
/dashboard           — User home (replaces v1 home)
/study               — Terminology flash cards (free)
/scenarios           — AI scenario practice (pro)
/exam                — Timed exam simulation (pro)
/exam/[id]/results   — Exam results
/tutor               — AI tutor chat (bundle)
/progress            — Progress & stats (pro)
/pricing             — Pricing page
/settings            — Account, subscription management
/api/...             — All API routes
```

---

## Landing Page (Critical for conversion)

**Hero:** "Pass your CPC exam on the first try — AI-powered prep at a fraction of the cost"

**Pain points addressed:**
- "AAPC courses cost $1,000+. We cost $79/year."
- "Static question banks get leaked. Our AI generates fresh scenarios every time."
- "The real exam has clinical scenarios. Practice them here, not just flashcards."

**Social proof:** (build as we go — testimonials, pass rate tracking)

**CTA:** "Start free — no credit card required"

---

## Go-to-Market

**Phase 1 (build):** 4-6 weeks to MVP
- Fork + auth + Stripe
- AI scenario engine
- Timed exam mode
- Landing page

**Phase 2 (launch):** 
- Post in r/medicalcoding, r/AAPC subreddits
- AAPC community forums
- Facebook groups for CPC students (large, active)
- TikTok/YouTube short: "I passed CPC for $79 instead of $1,500"

**Phase 3 (grow):**
- SEO: "CPC exam practice questions", "free CPC practice test"
- Affiliate: medical coding bloggers/YouTubers
- Track and publish pass rates (if students consent)

---

## Revenue Model

| Scenario | MRR |
|---|---|
| 100 Pro monthly ($29) | $2,900 |
| 100 Pro annual ($79) | ~$658/mo amortized |
| 50 Bundle annual ($149) | ~$621/mo amortized |
| **500 users mixed** | **~$15,000/mo** |

AI costs at scale: ~$0.10-0.50/scenario (GPT-4o) — manageable at these prices.

---

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| AI hallucinates wrong codes | Code validation layer + "verify with code book" disclaimer |
| AAPC copyright on exam questions | We generate original questions — never copy real exam content |
| Low conversion from free | Strong free tier → clear value wall at Pro features |
| Competition from AAPC itself | We're cheaper, faster, AI-native — they're slow to move |
| Student data privacy (HIPAA-adjacent) | No real patient data, just fictional scenarios — no HIPAA risk |

---

## Completion Criteria (MVP)

- [ ] Auth (signup, login, Google OAuth)
- [ ] Stripe subscriptions (free, pro, bundle tiers)
- [ ] Terminology cards from v1 (free tier)
- [ ] AI scenario generator working (pro tier)
- [ ] Timed 100-question exam mode (pro tier)
- [ ] Post-exam AI analysis
- [ ] Landing page with pricing
- [ ] Deployed to Vercel on custom domain
- [ ] Stripe webhooks updating user tier in Supabase

---

## Repo & Infrastructure

- **Repo:** `ud4090v/cpc-pro` (fork of cpc-prep)
- **Vercel team:** BlackRabbitDev
- **Supabase:** New project (cpc-pro)
- **Stripe:** New product (CPCPro)
- **Domain:** TBD (cpcpro.com? cpcprepai.com?)
- **Dee's version:** `ud4090v/cpc-prep` — untouched, stays as-is
