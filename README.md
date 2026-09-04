# GTM Hiring Signal Engine

Turns a company name into a verified hiring signal with the right person, their email, and a Slack alert — no Clay, no custom code, single n8n workflow.

## Project Overview

Finding companies that are actually hiring for GTM roles is slow. Teams scrape careers pages, read 20 postings, guess if its intent, then hunt for the hiring manager.

This automation solves that by taking a company name + optional careers URL, discovering the careers page via Tavily, searching current GTM openings, classifying the single strongest signal with an LLM, then finding and verifying the right person via a cheap-first waterfall. Only valid email + positive intent reaches HubSpot/Slack.

## The Problem

Without an automated engine:

- Teams manually open careers pages
- Generic AI personalization misses real intent
- Wrong person contacted (recruiter vs hiring manager)
- Unverified emails bounce and hurt deliverability
- No quarantine reason when intent/person/email fails

## The Solution

This n8n workflow automatically:

1. Receives Tally webhook {company_name, domain, careers_url?}
2. Discovers careers page: Tavily Search -> Choose best URL -> Tavily Extract (or direct Extract if URL supplied)
3. Searches current GTM jobs via Tavily Search (include_raw_content: true, 24k chars max)
4. Classifies with OpenRouter minimax (8 signals: RevOps/SalesOps/MarketingOps/GTM/CRM/Sales Enablement/Demand Gen/Business Systems) — score >=0.55, confidence >=0.55, evidence required
5. Gate: has_intent === true else quarantine (no_hiring_intent)
6. Finds person — Tier 1: Hunter domain search (limit 10) + function scoring (RevOps/GTM/SalesOps/MarketingOps/CRM, threshold >=100) -> Select Best Person
7. Fallback Tier 2: Tavily LinkedIn search site:linkedin.com/in + company + target_terms -> Parse -> score >=0.75 (current employees only)
8. Gets email — Prospeo via LinkedIn URL -> Dropcontact fallback (45s wait) -> ZeroBounce verify (only valid proceeds)
9. Valid -> HubSpot upsert contact + Slack Block Kit to #all-fafo | Invalid -> quarantine with reason

## Workflow

Tally (company) -> Tavily Discover (Search+Extract) -> Tavily Job Search -> LLM Classify (minimax) -> Gate -> Hunter Domain Search -> Select Best Person -> Hunter Found? --NO--> Tavily LinkedIn Search -> Prospeo (LinkedIn URL) -> Dropcontact fallback -> ZeroBounce -> Valid? --YES--> HubSpot + Slack | --NO--> Quarantine

## Technologies Used

- n8n — 27 nodes, single workflow WzvSZX5hAYXtZygU
- Tavily — Discovery, extraction, job search, LinkedIn person search
- OpenRouter minimax/m3:free — LLM classification (strict JSON)
- Hunter — Domain search (primary person, 50/mo free)
- Prospeo — Email from LinkedIn URL (primary email, 75/mo)
- Dropcontact — Fallback enrichment (50 trial)
- ZeroBounce — Verification gate (100 free, api-us.zerobounce.net)
- HubSpot — Sandbox 247212224, 11 custom props
- Slack — Block Kit to #all-fafo (C0BN1L6KDBR)

## HubSpot Fields

11 custom props: hiring_signal_type, hiring_signal_score, hiring_signal_confidence, hiring_signal_evidence, person_source (hunter/tavily), email_source (prospeo/dropcontact), email_status (valid), plus standard name/email/company.

## Example Output

Valid signal:

```
{
  "has_intent": true,
  "tool_mentioned": "HubSpot",
  "inferred_pain_point": "CRM migration bandwidth",
  "hiring_signal_type": "RevOps",
  "score": 0.82,
  "confidence": 0.78
}
```

Quarantine: `quarantined: true, quarantine_reason: no_person_found / email_not_verified / no_hiring_intent`

## Follow-Up Logic

- No intent / score <0.55 -> quarantine, no HubSpot/Slack.
- Hunter score <100 and Tavily LinkedIn <0.75 -> quarantine no_person_found.
- Email not valid on ZeroBounce -> quarantine email_not_verified.

## Example Scenario

Input: {company_name: "Linear", domain: "linear.app"} -> Tavily finds careers + 5 GTM jobs -> LLM classifies RevOps intent 0.82 -> Hunter finds 10 emails -> Select Best Person scores RevOps 120 -> Prospeo finds email -> ZeroBounce valid -> HubSpot contact created -> Slack card posted.

## Demo

- Tavily careers discovery + job aggregation (24k cap)
- LLM strict schema classification
- Hunter + Tavily LinkedIn fallback
- Prospeo/Dropcontact + ZeroBounce waterfall
- HubSpot upsert + Slack quarantine gates

Add Loom + screenshots here.

## Benefits

- Finds real hiring intent, not keyword fluff
- Targets hiring manager, not generic recruiter
- Cheap-first waterfall protects free-tier credits
- Strict gates prevent bad CRM writes
- Quarantine reasons for debugging

## Security

Never upload credentials. Tavily, OpenRouter, Hunter, Prospeo, Dropcontact, ZeroBounce, HubSpot, Slack tokens live in n8n only. Exports in workflows/ scrubbed.

## Possible Improvements

- Add Apollo as Tier 0 for broader coverage
- Add multi-signal detection (not single strongest)
- Add HubSpot Company + Deal association
- Add Slack buttons for qualify/disqualify

## Project Status

Completed — Demo / Portfolio Version — Live workflow WzvSZX5hAYXtZygU on Railway, webhook gtm-hiring-signal-v3.

## Author

Chiranjeev Sahu — GTM Engineering
Skills Demonstrated: n8n Tavily LLM Classification Hunter Prospeo ZeroBounce HubSpot
