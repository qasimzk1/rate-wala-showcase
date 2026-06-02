# Rate Wala — UAE Money Transfer Rate Comparison

**[ratewala.org](https://www.ratewala.org)** · Live · Free · No signup required

> UAE's 3.5M+ Indian, Pakistani, Filipino and Egyptian expats collectively send over $30B home every year. Most use whichever exchange house is closest. The rate gap between the best and worst provider on a single AED 1,000 transfer is often 300–3,000 units of local currency. Nobody was showing them this clearly. Rate Wala does.

---

## The Problem

When a UAE expat sends AED 1,000 to Pakistan, the difference between the best available rate and the average provider is typically PKR 500–3,500 — on a single transfer. Sent monthly, that's PKR 6,000–42,000 lost every year by not spending 10 seconds comparing.

Existing tools either required signup, showed rates without fees included, or were built for a Western audience. None were built for the UAE expat specifically — in their corridors, with their providers, in their context.

---

## The Product

Rate Wala compares live AED exchange rates across 10 UAE providers — exchange houses, banks, and transfer apps — for 9 outbound corridors. Users see the total amount the recipient gets, including all fees, so the comparison is always apples to apples.

**Corridors:** AED → INR · PKR · PHP · EGP · BDT · LKR · NPR · GBP · USD

**Providers compared:** Wise · Al Ansari Exchange · Lulu Exchange · Careem Pay · TapTap Send · Western Union · Orient Exchange · Botim Pay · Al Fardan Exchange · Remitly

**Core features:**
- Live rate comparison, updated every 30 minutes
- 30-day rate gap insight card with behavioural framing
- 14-day rate trend badge ("Best in 14 days" / "Rate rising this week")
- Rate alerts via email (notify when a corridor hits a target rate)
- PWA with iOS and Android install prompts
- Push notification opt-in (OneSignal)
- Weekly newsletter signup
- 7-day rate history chart per corridor
- SEO-optimised corridor landing pages

---

## Architecture (High Level)

```
cron-job.org (every 30 min)
  → POST /api/cron/trigger
  → GitHub Actions workflow_dispatch
      ├── seed-mac    (self-hosted UAE Mac runner — required for geo-restricted providers)
      └── seed-fallback (ubuntu-latest — parallel, kicks in if Mac is offline)
  → Scrapers write to Supabase rate_snapshots
  → /api/rates serves data (60s memory cache → Supabase 24h lookback → live cold scrape)
  → Next.js App Router renders corridor pages
```

**Stack:** Next.js 14 (App Router) · TypeScript · Supabase (Postgres) · Vercel · GitHub Actions · PostHog · OneSignal · Brevo

**Why a self-hosted UAE runner?** Two major UAE exchange houses (Lulu Exchange, Al Ansari) block non-UAE IP addresses. A Mac mini on UAE network acts as the primary scraper. Ubuntu cloud runners handle fallback with estimated rates from other providers.

---

## Key Product Decisions

### 1. Loss framing over gain framing
The rate gap card reads *"What not comparing actually **costs** you"* rather than *"What you could save."* Behavioural economics research shows loss framing (Kahneman/Tversky) drives 2x stronger action response. The annual cost is contextualised in local terms — *"That's a month's school fees in Pakistan"* — because abstract numbers don't move people but relatable ones do.

### 2. 30-day history, not today's spread
The insight card uses 30-day historical worst average vs best absolute rate, not today's live spread. This makes the card educational and stable — it tells a monthly story rather than fluctuating with each scrape, and is more defensible as a "this is what actually happens" claim.

### 3. Fallback decoupled from primary runner
Early architecture had the cloud fallback job dependent on the Mac runner completing (`needs: seed-mac`). When the Mac went offline, GitHub's queue timeout is 24 hours — the fallback never fired, causing a 12-hour data blackout. Decoupled to run in parallel: cloud job waits 10 minutes, checks Supabase freshness directly, and scrapes only if needed. Zero dependency on hardware status.

### 4. Corridor pages as SEO entry points, not just tools
Each corridor (`/send/aed-to-pkr`) is a full content page — structured data (FAQPage, BreadcrumbList), hub-and-spoke internal linking, cost comparison guide, corridor statistics. These pages are optimised to rank for "[from] to [to] rate today" queries, which have high intent and no signup friction. The comparison tool is the primary action, not the primary acquisition channel.

### 5. No hardcoded rates
An early version had static monthly rate ranges hardcoded in the codebase. When live data diverged (INR rates moved), the insight card showed ₹22,950 as "best this month" while the live result below showed ₹25,933 — a trust-destroying inconsistency. All values now derived from live Supabase queries.

### 6. ?highlight= URL parameter for provider search
Users searching "Al Ansari Exchange AED to INR rate" land on the corridor page. Adding `?highlight=alansari` to internal links surfaces Al Ansari's card with a blue "Your search" badge and smooth-scrolls to it. Same page, same product — no duplicate content, but context-aware presentation.

---

## Experiments Running

Three concurrent A/B experiments via PostHog feature flags:

| Flag | Variants | Hypothesis |
|------|----------|------------|
| `cta_button_label` | "Open App" / "Send Money →" / "Open [Provider Name]" | Named provider increases click-through by reducing ambiguity |
| `send_form_variant` | Control / V2 (pulsing empty state, compact currency selector, ±100 step buttons) | Clearer affordance increases form interaction rate |
| Insight card punchline | 5 per corridor, served randomly | Fresh copy on each visit reduces habituation |

---

## Operations

- **Uptime monitoring:** Better Stack
- **Rate freshness:** cron-job.org polls `/api/health` every 10 minutes; alerts if `rates_fresh: false`
- **Incident runbook:** 2 documented incidents (INC-001: GitHub cron unreliability → fixed with external HTTP trigger; INC-002: self-hosted runner offline causing 12h blackout → fixed with decoupled fallback architecture)
- **Daily rhythm:** Automated Slack standup at 9am and EOD summary at 6pm via scheduled Claude agents — posts system health, commits shipped, and top 3 backlog priorities
- **KB:** Architecture decisions, incidents, and lessons learned maintained in version-controlled markdown

---

## Roadmap (Active)

- [ ] Affiliate/referral integration with transfer providers
- [ ] KSA expansion — SAR corridors with Al Rajhi, STC Pay
- [ ] Weekly newsletter with best rates per corridor (Brevo)
- [ ] Daily push notifications via OneSignal
- [ ] Provider-specific landing pages for branded search SEO
- [ ] PostHog conversion funnels and cohort analysis

---

## On Code Availability

The source code for Rate Wala is maintained in private repositories. This is intentional for two reasons:

1. **Scraper logic:** The scrapers contain reverse-engineered rate endpoints for UAE exchange providers. Making these public would undermine the data advantage and potentially attract abuse of provider APIs.

2. **Business sensitivity:** Internal architecture documents, incident logs, and pipeline configuration contain operational details (runner setup, monitoring credentials, webhook URLs) that are not appropriate for public access.

The live product at [ratewala.org](https://www.ratewala.org) is the most complete demonstration of the work. Happy to walk through any aspect of the architecture, product decisions, or operational setup in a conversation.

---

## Contact

**Qasim** · [ratewala.org](https://www.ratewala.org)
