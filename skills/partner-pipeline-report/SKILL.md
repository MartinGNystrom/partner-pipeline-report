---
name: partner-pipeline-report
description: Build a Salesforce partner-pipeline report (partner status, sponsors/named contacts, open and lifetime pipeline) for a list of technology/OEM partner companies, using parallel per-company agents plus a QA verification pass, then publish it as a clean standalone report. Use when the user asks for a partner report, partner pipeline, or "find Salesforce opportunities for these companies" naming a list of vendor/partner companies.
---

# Partner Pipeline Report

## Overview

Given a list of partner/vendor companies (e.g. cybersecurity, AI, or quantum vendors an
organization resells or integrates), produce a report covering, per company: CRM partner status
(tier, program status, partner manager), named sponsors/economic buyers on top deals, and pipeline
(open + lifetime closed-won), then publish it.

First built 2026-07-20 for a six-company report (Claroty, Filigran, Doppel, Command Zero,
Keyfactor, BlackCloak) at World Wide Technology (WWT). See `references/fork-prompt-templates.md`
for the exact prompts used and lessons learned from that run, and
`references/report-template.html` for a ready-to-adapt report template.

## Key CRM facts (Salesforce, via an MCP connector)

These notes describe the schema conventions observed in WWT's Salesforce org. If you're running
this against a different org, verify the equivalent fields/objects before relying on them — but
the general shape (partner status lives on the Account, opportunities link to partners by name
rather than by a dedicated lookup, sponsors live on OpportunityContactRole) is a common Salesforce
CRM pattern worth checking for first.

- Tool names for the CRM MCP connector are dynamic and vary by install. Use `ToolSearch` with a
  query like `"CRM SOQL Salesforce opportunity"` to find the connector's `soqlQuery`, `find` (SOSL),
  `getObjectSchema`, and `getRelatedRecords` tools before calling them.
- **Partner companies are `Account` records with `Type = 'Partner'`.** Look them up with SOSL:
  `FIND {<Company>} IN NAME FIELDS RETURNING Account(Id, Name, Type, Industry, Website)`. Watch
  for spell-correction/fuzzy matches (SOSL will return near-miss company names too — e.g.
  searching "Claroty" also returns "Clarity"-named accounts) and for near-identical unrelated
  companies (e.g. "Doppel" vs "Dopple"). Confirm the right Account by Website/Industry before
  using its Id.
- **Partner status fields live on Account**, not on a separate junction object. In the WWT org:
  `Type`, `Partner_Type__c`, `Partner_Tier__c` (Strategic/Advantage/Select/Approved/Evaluation/
  Active), `Status__c` (onboarding status), `Partner_Manager__c` (a User Id — resolve to Name/
  Title with a follow-up `SELECT Id, Name, Title, Email FROM User WHERE Id IN (...)`),
  `Engaged_Partner_Manager__c`, `WWT_Executive_Partner_Sponsors__c`, `Sponsor_Name__c`,
  `Partner_Priority__c`, `Partner_Certification_Status__c`. In practice the executive-sponsor
  fields are usually blank — say so plainly rather than implying an executive sponsor exists when
  the field is empty.
- **Opportunities do NOT link to the partner via `AccountId`** — `AccountId` is the *customer*.
  A typical Opportunity object has no partner/vendor lookup field at all. Find partner-linked
  opportunities by searching **Name** (and optionally `NextStep`) for the partner's name instead:
  `SELECT StageName, COUNT(Id), SUM(Amount) FROM Opportunity WHERE Name LIKE '%<Company>%' GROUP
  BY StageName` for pipeline aggregates, and `FIND {<Company>} IN NAME FIELDS RETURNING
  Opportunity(Id, Name, StageName, Amount, CloseDate, AccountId, Account.Name)` for the
  individual-deal list. `Opportunity.Description` (a textarea field) typically can't be
  `LIKE`-filtered — `Name` and `NextStep` usually can.
- **"Open pipeline"** = every stage except the closed stages (e.g. `Closed Won`, `Closed Lost`,
  and any "no bid"/disqualified stage — check the org's exact `StageName` picklist values first,
  including odd whitespace in a label).
- **Named sponsors/economic buyers** come from `OpportunityContactRole` (child relationship on
  Opportunity: `Contact.Name`, `Contact.Title`, `Role`, `IsPrimary`) queried per top opportunity —
  not from Account-level sponsor fields, which are typically empty.
- **Watch for shared/bundled opportunities**: one Opportunity can reference two partners in its
  Name (e.g. "Newmont_Doppel & Blackcloak"), so it will show up in both companies' searches. Flag
  it and de-duplicate by Opportunity Id when computing any combined/cross-company total.
- **Watch for duplicate Account records** for the same partner company — check opportunities
  against all matching Account Ids, and flag the duplication itself as a CRM-hygiene finding
  rather than silently picking one.
- Salesforce URLs are safe to hand-construct from a confirmed Id once you have it:
  `https://<org>.lightning.force.com/lightning/r/Opportunity/{Id}/view`.

## Workflow

1. **Resolve accounts.** For each company name given, SOSL-search Account as above and confirm
   the right Id(s) (watch for fuzzy matches and duplicates).
2. **Pull Account partner-status fields** for all resolved Ids in one SOQL query, then resolve any
   Partner_Manager__c (or equivalent) User Ids to names/titles in a follow-up query.
3. **Pull pipeline aggregates** per company (`GROUP BY StageName` SOQL on `Name LIKE`) and a
   sample opportunity list (SOSL `FIND`) to identify top open deals. Do this directly yourself for
   a first pass if the company count is small, or skip straight to step 4 if you'd rather have
   agents do the primary research.
4. **Launch one fork/subagent per company, all in a single message** (see
   `references/fork-prompt-templates.md` for the exact prompt template) to verify the aggregates,
   pull the top 5 open opportunities, look up `OpportunityContactRole` sponsors on those
   opportunities, note any other populated partner-status fields, and write a 2-3 sentence
   public-knowledge company blurb tied to the report's theme. A forked subagent inherits the
   parent's context, so it only needs its own company's already-known facts plus clear scope
   boundaries (explicitly rule out any name-collision or shared-opportunity risk already spotted).
5. **If a subagent's final report doesn't actually contain the requested findings** (e.g. it
   trails off into a status update instead of synthesizing), message it directly asking it to
   report what it already found — don't discard its work and re-run from scratch.
6. **Launch one QA subagent** once all per-company subagents are back (template in
   `references/fork-prompt-templates.md`): paste the fully compiled dataset, have it check each
   company's arithmetic, compute the correct cross-company combined totals with any shared
   opportunity de-duplicated, spot-re-run one live aggregate query per company to catch drift, and
   flag anything internally inconsistent. **Use QA's corrected numbers**, not the pre-QA draft.
7. **Build the report.** Start from `references/report-template.html` — a self-contained,
   theme-aware (light/dark) HTML template with a kicker/header, a headline stating the
   conclusion (not just the topic), a BLUF callout, a combined summary table across all companies,
   one section per company (status line, 2-3 sentence blurb, a callout for anything notable —
   missing partner manager, zero closed-won, duplicate accounts, a whitespace angle, etc. — and a
   top-opportunities table with named contacts), and a footer byline. If the user has their own
   preferred document style/template, follow that instead.
8. **Publish the report.** If the user has access to an internal page-hosting service (e.g. WWT's
   my-pages, reachable via its MCP connector), publish there and share the resulting URL — that's
   the actual deliverable, don't wait to be asked to share it. Otherwise, write the finished HTML
   to a file and tell the user where it is.

## Notes

- This is read-only CRM work — no writes to Salesforce at any point.
- Company count of 1-2 doesn't need the multi-subagent treatment; do the SOQL/SOSL directly. The
  parallel-subagent approach earns its keep once you're covering enough companies (roughly 4+)
  that a QA pass catching one arithmetic slip is worth the overhead.
- If asked to update an existing partner report rather than create a new one, find the existing
  page (e.g. by name/slug on the hosting service) and upload a new version to it instead of
  creating a new one.
