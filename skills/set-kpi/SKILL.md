---
name: set-kpi
description: >
  Create KPI docs, compensation structures, screening questions, and hiring templates for any
  contractor role. Use this skill whenever the user wants to hire someone, set KPIs, define
  compensation, create a job post, write screening questions, or build a hiring doc — for ANY
  role including community managers, YouTube managers, producers, marketers, developers, VAs,
  or any contractor. Also trigger when the user says "set KPI", "hiring doc", "job post",
  "comp structure", "screening questions", or "Fiverr post".
---

# Set KPI — Contractor Hiring Doc Generator

Create complete, visual hiring docs for any contractor role. The output is a ready-to-use
KPI doc with ASCII charts, comp scenarios, screening questions, and messaging templates.

## Why this skill exists

Setting KPIs for contractors is high-leverage but tedious. Bad KPIs lead to misaligned
incentives, wasted time, and contractors who game metrics instead of driving results.
This skill ensures every role gets KPIs that are reverse-engineered from real business
targets — not arbitrary numbers pulled from thin air.

## The Process

Follow these 6 steps in order. Use AskUserQuestion at each step to gather input.

### Step 1: Understand the Role

Ask:
- What role are you hiring for? (title, responsibilities)
- Is this a new role or updating an existing one?
- What problem does this person solve for you? What are you currently doing that you want off your plate?

Check if `docs/hiring/about-bookzero.md` exists — if so, the shared company page is already
built and new role docs should reference it.

### Step 2: Business Context & Targets

Ask:
- What's the business target this role supports? (e.g., 1,000 users by October)
- What's the current baseline? (e.g., 43 users now)
- What % of the target does this role contribute to?
- What's the timeline?

Build a month-by-month growth table from current state to target. Identify key inflection
points (tax season, product launches, seasonal demand).

### Step 3: Compensation Structure

Ask:
- Fixed salary, commission-based, or hybrid?
- What's the budget range?
- Any performance bonuses or milestone payouts?

Then generate 3 comp scenarios (slow month, average month, strong month) as ASCII box diagrams:

```
  SCENARIO 1: Slow Month
  ───────────────────────────────────────
  Base                          $XXX
  X deliverables     × $XX   = $XXX
                                ─────
  TOTAL                        $XXX/mo
```

### Step 4: Define KPIs

Reverse-engineer KPIs from the business target:
- Start with the end goal (e.g., 1,000 users)
- Work backward: what monthly numbers are needed to hit that?
- Break into leading indicators (activities) and lagging indicators (results)
- Set Month 1 (ramp), Month 2-3 (growth), Month 4+ (mature) targets

Create ASCII visualizations:
- Pipeline/funnel diagram showing the flow
- Bar chart showing monthly ramp-up targets
- Conversion math showing how activities → results

KPIs should be weighted by importance. Primary metric gets 25%+ weight.

### Step 5: Screening & Hiring

Generate:
- 5-7 screening questions tailored to the role's domain
- Questions that test: domain knowledge, past results, strategic thinking, commitment
- A "What You Must Submit" section requiring a written plan (Week 1 day-by-day, strategy, milestones)
- Red flags that trigger contract end

At least one question should require the applicant to visit bookzero.ai and demonstrate understanding.

### Step 6: Messaging Templates

Generate two Fiverr-ready messages:
1. **Reply to applicants** — Thanks, links to docs, specific deliverables required to proceed
2. **New job post** — Role summary, comp, hours, links to docs, "must include" requirements

Both should link to the Google Doc versions if URLs are provided.

## Output Format

All docs go in `docs/hiring/`. Use short file names (e.g., `outbound-kpi.md`, `community-mgr-kpi.md`).

### Visual Style Guide

Use ASCII box diagrams for everything. These render in Google Docs, Notion, email, and any
markdown viewer without needing Mermaid or special tools.

**Boxed tables:**
```
  ┌──────────────────────────────────────┐
  │  HEADER                              │
  ├──────────────────────────────────────┤
  │  Row 1           │  Value            │
  │  Row 2           │  Value            │
  └──────────────────┴───────────────────┘
```

**Pipeline/funnel:**
```
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  STEP 1  │ →  │  STEP 2  │ →  │  STEP 3  │
  └──────────┘    └──────────┘    └──────────┘
```

**Bar charts:**
```
  30+ ·····  ████████████████████████████████
  20  ·····  ████████████████████████
  10  ·····  ████████████████
   0  ───────┬──────────┬──────────┬────────
          Month 1     Month 2    Month 3
```

**Timeline/handoff:**
```
  Month 1          Month 2          Month 3
  ┌────────┐       ┌────────┐       ┌────────┐
  │ PHASE  │  →    │ PHASE  │  →    │ PHASE  │
  │ Name   │       │ Name   │       │ Name   │
  └────────┘       └────────┘       └────────┘
```

**Comp scenarios:**
```
  SCENARIO: Description
  ───────────────────────────────────────
  Component 1                    $XXX
  Component 2       × $XX     = $XXX
                                 ─────
  TOTAL                         $XXX/mo
```

### Doc Structure

Every role KPI doc follows this structure:

```
# ROLE TITLE
### BookZero | KPIs & Expectations

> Reference to about-bookzero.md

## THE ROLE (table + daily time breakdown chart)
## COMPENSATION (scenarios as ASCII boxes)
## YOUR PIPELINE/FUNNEL (ASCII flow diagram)
## KPIS (tables + ASCII bar charts + conversion math)
## GROWTH TIMELINE (phase boxes with handoff plan)
## WHAT YOU MUST SUBMIT (numbered list)
## SCREENING QUESTIONS (numbered table)
## RED FLAGS (checklist)
```

Keep each doc under 2 pages when pasted into Google Docs. Use ALL CAPS for section headers.
No emojis. Every line earns its space.

### Fiverr Messages

Append to `docs/hiring/fiverr-messages.md` (create if doesn't exist). Each message goes
inside a code fence for easy copy-paste.

## Example Roles

This skill works for any role. Common ones:

| Role | Key KPIs | Comp Model |
|------|----------|------------|
| LinkedIn Outbound | Demos booked, close rate, lead quality | Low base + commission |
| QA + Growth PM | Funnel metrics, bugs caught, sprint velocity | Fixed with raise gate |
| Community Manager | Engagement rate, member growth, retention | Fixed or hybrid |
| YouTube Manager | Views, subs growth, CTR, upload cadence | Fixed + per-video bonus |
| YouTube Producer | Videos delivered, quality score, turnaround | Per-video rate |
| Content Writer | Posts published, traffic, backlinks | Per-piece or monthly |
| VA / Admin | Tasks completed, response time, accuracy | Hourly or monthly |
