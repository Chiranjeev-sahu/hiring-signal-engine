# Project 4 — GTM Hiring Signal Engine

Turns a company name into a verified hiring signal with the right person, their email, and a Slack alert — no Clay, no custom code, single n8n workflow.

## Overview

```
Tally webhook (company + optional careers_url)
  -> Tavily: discover careers page + search current GTM jobs
  -> LLM (OpenRouter minimax): classify single strongest GTM hiring signal
     [RevOps, SalesOps, MarketingOps, GTM, CRM, Sales Enablement, Demand Gen, Business Systems]
  -> Hunter domain search (primary) -> score by function match (RevOps/GTM/SalesOps/MarketingOps/CRM)
  -> Tavily LinkedIn fallback: find current employees in target functions
  -> Prospeo (via LinkedIn URL) -> Dropcontact fallback -> ZeroBounce verify
  -> HubSpot upsert contact (full enrichment) + Slack Block Kit #all-fafo
```

Only records with **valid email + positive intent** reach HubSpot/Slack. Everything else quarantined with reason.

## Results

| Metric | Value |
|---|---|
| Input | Tally webhook `POST /webhook/gtm-hiring-signal-v3` `{company_name, domain, careers_url?}` |
| Career discovery | Tavily Search (if URL not supplied) + Tavily Extract |
| Job search | Tavily Search targeted GTM role queries (`include_raw_content: true`) |
| Classification | OpenRouter `minimax/minimax-m3:free` — strict schema, 8 signal types, score ≥0.55, confidence ≥0.55, evidence required |
| Person finding | Primary: Hunter domain search (limit 10) + function scoring threshold ≥100<br>Fallback: Tavily LinkedIn search `site:linkedin.com/in/ + company + target_terms`, score ≥0.75 |
| Email | Primary: Prospeo via LinkedIn URL + name + company<br>Fallback: Dropcontact enrichment (45s wait) |
| Verification | ZeroBounce — only `status: valid` proceeds |
| HubSpot | Upsert contact (portal `247212224`) with signal, person, email, source |
| Slack | Block Kit to `#all-fafo` (C0BN1L6KDBR) |
| Quarantine | Gates at: no intent, no person found, email not verified — tagged with reason |

## Architecture

**One workflow** (`WzvSZX5hAYXtZygU`, 27 nodes) with dual-path discovery and multi-tier fallbacks:

- **Intake**: Tally-compatible webhook → normalize
- **Discover**: Supplied URL → Tavily Extract **OR** Tavily Search careers → Choose best URL → Tavily Extract
- **Aggregate**: Tavily Extract career page + Tavily Search current GTM jobs → Prepare Career Content (24k chars max)
- **Classify**: LangChain InformationExtractor + OpenRouter minimax → Normalize Hiring Signal (validation gate)
- **Gate**: `has_intent === true` + score/confidence/evidence thresholds
- **Person (Tier 1)**: Hunter domain search → Select Best Person (function scoring RevOps/GTM/SalesOps/MarketingOps/CRM, threshold 100)
- **Person (Tier 2)**: Tavily LinkedIn search → Parse → filter current employees in target functions (score ≥0.75)
- **Email**: Prospeo (LinkedIn URL) → Dropcontact fallback → ZeroBounce
- **Output**: Valid → HubSpot upsert contact + Slack Block Kit | Invalid/No-match → Quarantine

## Stack

| Tool | Purpose | Tier |
|---|---|---|
| Tavily | Careers page discovery, extraction, job search, LinkedIn person search | Primary |
| OpenRouter (minimax) | LLM classification | Free |
| Hunter | Domain search → person emails | Primary person |
| Prospeo | Email from LinkedIn URL | Primary email |
| Dropcontact | Email enrichment fallback | Fallback |
| ZeroBounce | Email verification gate | Gate |
| HubSpot | Contact upsert (portal 247212224) | CRM |
| Slack | Block Kit alert to `#all-fafo` | Notification |
| n8n | Orchestration (single workflow) | Platform |

All free tier or trial. No Clay. No custom Node/Python services.

## Repo Map

```
workflows/           (empty — workflow lives in n8n, ID: WzvSZX5hAYXtZygU)
tests/fixtures/      (to be populated: tavily-careers.json, tavily-jobs.json, hunter-results.json, 
                      prospeo-email.json, zerobounce.json, hiring-signal.json)
src/                 (validators, parsers — to be added)
docs/
  schema.md          field maps for each stage
  runbook.md         rebuild-from-zero guide
  decisions.md       D1–D8 locked
  credentials.md     signup → credential → scopes for each tool
  linkedin-post.md   announcement copy
  loom-script.md     4-min demo script
```

## Verification

- **HubSpot portal 247212224**: Contacts show `hiring_signal_type`, `hiring_signal_score`, `hiring_signal_confidence`, `hiring_signal_evidence`, `person_source` (hunter/tavily), `email_source` (prospeo/dropcontact), `email_status` (valid)
- **Slack `#all-fafo`**: Block Kit message with company, signal type, score, person name/title/email, source tags
- **Quarantine**: Code nodes tag `quarantined: true` + `quarantine_reason` (no_hiring_intent | no_person_found | email_not_verified)

## Next

- Populate `tests/fixtures/` with live run payloads for each stage
- Add `src/validateSignal.mjs` + `src/parseHunter.mjs` + `src/parseTavilyPerson.mjs` with TDD
- Document credential acquisition for Tavily, Hunter, Prospeo, Dropcontact, ZeroBounce in `docs/credentials.md`