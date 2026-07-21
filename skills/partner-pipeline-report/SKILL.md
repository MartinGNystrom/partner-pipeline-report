---
description: Build a Salesforce partner-pipeline report (partner status, sponsors/named contacts, open and lifetime pipeline) for a list of technology/OEM partner companies, using parallel per-company agents plus a QA verification pass, then publish it as a clean standalone report. Use when the user asks for a partner report, partner pipeline, or "find Salesforce opportunities for these companies" naming a list of vendor/partner companies.
---

# Partner Pipeline Report

## Overview

Given a list of partner/vendor companies (e.g. cybersecurity, AI, or quantum vendors an
organization resells or integrates), produce a report covering, per company: CRM partner status
(tier, program status, partner manager), named sponsors/economic buyers on top deals, pipeline
(open + lifetime closed-won), and — where those sources are available — a fuller narrative pulled
from ZoomInfo firmographics/signals, internal documents, meeting transcripts, and Slack, then
publish it.

First built 2026-07-20 for a six-company report (Claroty, Filigran, Doppel, Command Zero,
Keyfactor, BlackCloak) at World Wide Technology (WWT); extended 2026-07-20 to pull in ZoomInfo,
documents, meetings, and Slack alongside CRM. See `references/fork-prompt-templates.md` for the
exact prompts used and lessons learned from that run, and `references/report-template.html` for a
ready-to-adapt report template.

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

## Enrichment sources beyond CRM (ZoomInfo, documents, meetings, Slack)

CRM tells you the deal facts (status, pipeline, named buyers) but not the *story* — momentum,
relationship health, competitive pressure, what internal stakeholders actually think. Layer these
in when available to turn a numbers table into a fuller narrative per company. None of these are
required for the skill to work; treat each as optional and degrade gracefully.

- **Discover what's actually connected first.** Tool availability varies by install. Run
  `ToolSearch` for each category (e.g. `"ZoomInfo company"`, `"Glean search"`, `"Slack search"`,
  `"Webex meeting"`, `"Google Drive"`) before assuming a source exists. Skip any category with no
  matching tools rather than failing the whole report over a missing connector — note the gap
  briefly in the report instead (e.g. "no Slack connector available for this run") only if it
  materially matters, not as boilerplate on every section.
- **ZoomInfo** (`mcp__*ZoomInfo*` tools): use `search_companies`/`enrich_companies` to confirm the
  right company record, then `enrich_company_signals`, `enrich_news`, and `enrich_scoops` for
  recent firmographic and buying-intent signals (funding rounds, leadership changes, expansion,
  layoffs, product launches). `get_gtm_context` can surface how the org's own GTM data already
  frames the account. Pull 2-4 bullet points of what's genuinely notable — not a data dump.
- **Internal documents** (commonly via Glean, `mcp__*Glean*__search` / `read_document`, or a
  Google Drive connector): search for the partner's name across internal docs — sales collateral,
  competitive positioning, QBR/EBR decks, partnership agreements. Cite what you find by title
  (and link/ID if the platform provides one) rather than quoting large blocks of text; summarize
  the relevant point in one line.
- **Meeting transcripts/summaries** (Glean `meeting_lookup`, a Webex Meetings connector's
  `list-recordings`/`get-meeting-summary`, or similar): look for recent meetings involving the
  partner. Summarize key takeaways, decisions, and action items at a high level — attendee names
  are fine to include when they're already the named CRM contacts/partner managers in this report,
  but don't transcribe verbatim exchanges.
- **Slack** (`mcp__*Slack*__slack_search_public_and_private` / `slack_search_channels` /
  `slack_read_channel` / `slack_read_thread`): search for recent internal mentions of the partner
  name. This is usually the best source for relationship-health signals that never make it into
  CRM — deal blockers, escalations, internal frustration or enthusiasm, a partner manager flagging
  a renewal risk. **Summarize the substance, don't quote people verbatim** unless the message is
  clearly fine to reproduce in a shared report (e.g. an official program announcement) — internal
  chat is written assuming an internal, ephemeral audience, and a published report has a longer
  shelf life and a different (sometimes external) readership than the original channel. When in
  doubt, paraphrase and attribute loosely ("the partner manager flagged renewal risk in Slack") or
  leave a detail out rather than embarrass or misrepresent someone.
- **All of this is read-only** — searching/reading only, never posting to Slack, editing docs, or
  writing to any of these systems.
- **Fold enrichment into the per-company subagent** (workflow step 4) rather than running it as a
  separate pass — each subagent already owns one company end-to-end, so it's the natural place to
  gather CRM facts and the wider narrative together and hand back one coherent picture.

## Certifications, awards, labs, and service capabilities

A separate enrichment pass covering "what can this partner actually do, and how credentialed are
they" — distinct from the deal-momentum narrative above. **Important scope note, corrected after a
first pass got this wrong:** "certifications and awards" here means **credentials WWT itself holds
or has won through that vendor's partner program** (e.g. "WWT is a CrowdStrike Focused Partner" or
"WWT named CrowdStrike Flex Partner of the Year") — not the vendor's own public certifications or
industry accolades as a company. Those are two entirely different things pointing at different
sources; don't conflate them.

- **The authoritative source is WWT's own public partner page**, `wwt.com/partner/<vendor-slug>/
  expertise` (read via `Glean read_document` or a web fetch — it's public). It carries a clean
  "`<Vendor>` Certifications and Awards" list with named credential + region + year, e.g.
  "CrowdStrike Focused Partner — Americas — 2020" and "CrowdStrike Flex Partner of the Year —
  Global — 2024." The companion `wwt.com/partner/<vendor-slug>/overview` page usually has the named
  WWT practice team (useful cross-check against the CRM `Partner_Manager__c`/Account Owner) plus an
  "About `<Vendor>` & WWT" blurb and recent partner-branded articles/blogs — a good source for the
  **other service capabilities** part of this section too, instead of guessing from general public
  knowledge.
- **A secondary, curated cross-check**: `wwt.com/corporate/awards-and-recognitions/overview` lists
  per-vendor certification counts, but only for a handful of WWT's largest infrastructure OEMs
  (Cisco, Dell, HPE, NetApp, F5, Intel, NVIDIA, Microsoft, Palo Alto Networks as of this writing) —
  most partners, including most cybersecurity/AI vendors, won't appear there. Don't treat its
  absence as meaningful; the per-vendor `/expertise` page is the one that matters.
- A `WebSearch` for a public press release (e.g. `"World Wide Technology" <vendor> "partner of the
  year"`) is a good corroborating source and often surfaces the exact announcement date/quote — use
  it to double check or add color to what the `/expertise` page states, not as the primary source.
- **Legacy Salesforce object**: `Certifications_and_Specializations__c` (`Account__c` lookup to the
  partner Account — `SELECT Vendor__c, Certification_or_Specialization_Type__c FROM
  Certifications_and_Specializations__c WHERE Account__c = '<partnerAccountId>'`) is the same
  underlying concept (a WWT-held certification/specialization tied to a vendor relationship) but
  it's an old object whose `Vendor__c` picklist is capped to a small set of legacy hardware vendors
  (Cisco, Citrix, EMC, HP, NetApp, VMware). Expect zero rows for any modern partner and say so as
  "no legacy record on file," not "this partner has no certifications" — the `/expertise` page
  above is the real answer, this is just a secondary data point.
- **`Partner_Skill__c`/`WWT_Skill__c`/`WWT_Parent_Skill__c`** look relevant by name (a partner/skill
  matrix with `Status__c` of Pending/Approved/Rejected, `Location__c`/`US_Locations__c`, an
  `Is_Active_Partner__c` flag) but **`Partner_Skill__c` has no confirmed lookup field to Account** —
  schema inspection turned up no `Account__c`/`Partner__c` reference on it, only a link up to
  `WWT_Skill__c`. Treat any attempt to tie a `Partner_Skill__c` row to a specific partner company as
  unverified (probably a name-text match at best) until you've confirmed the actual linkage against
  live data — don't present it as a solid finding the way the Account-level fields are.
- **Labs**: there is no "lab" object in Salesforce. WWT's Advanced Technology Center content is
  reachable two ways — `Glean search` with `app: "atc platform"` (labeled "Cisco Lab" in the
  connector's own app-filter description, but not limited to Cisco in practice), or directly reading
  `wwt.com/partner/<vendor-slug>/overview`, which usually has a "`<Vendor>` in the ATC" section
  listing specific named labs (with launch counts), integration labs with adjacent vendors, and any
  Cyber Range/capture-the-flag events.
- Fold this into the same per-company subagent as the other enrichment work (workflow step 4) —
  don't spin up a fifth pass just for this.

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
   opportunities, note any other populated partner-status fields, pull whatever ZoomInfo/document/
   meeting/Slack signals are available (see the enrichment section above), pull certifications/
   awards/labs/service-capability information (see that section below), and write a short
   narrative blurb combining public-knowledge company context with what internal sources actually
   show. A forked subagent inherits the parent's context, so it only needs its own company's
   already-known facts plus clear scope boundaries (explicitly rule out any name-collision or
   shared-opportunity risk already spotted).
5. **If a subagent's final report doesn't actually contain the requested findings** (e.g. it
   trails off into a status update instead of synthesizing), message it directly asking it to
   report what it already found — don't discard its work and re-run from scratch.
6. **Always launch a separate QA subagent** once the data is compiled (template in
   `references/fork-prompt-templates.md`) — this step is never skipped, including for a single
   company handled directly rather than via a fork in step 4. Paste the fully compiled dataset,
   have it check the arithmetic, compute the correct cross-company combined totals with any shared
   opportunity de-duplicated (when there's more than one company), spot-re-run at least one live
   query to catch drift, and flag anything internally inconsistent — including a large discrepancy
   between a CRM-derived number and any other reported figure a document search turned up (e.g. a
   vendor's own reported bookings vs. a SOQL Closed-Won total), which is worth an explanation
   attempt rather than presenting both silently. **Use QA's corrected numbers**, not the pre-QA
   draft.
7. **Build the report.** Start from `references/report-template.html` — a self-contained,
   theme-aware (light/dark) HTML template with a kicker/header, a headline stating the
   conclusion (not just the topic), a BLUF callout, a combined summary table across all companies,
   one section per company (status line, a short narrative blending public company context with
   any ZoomInfo/document/meeting/Slack signals found, a certifications/awards/labs/capabilities
   list where anything was found, a callout for anything notable — missing partner manager, zero
   closed-won, duplicate accounts, a whitespace angle, a relationship-health flag surfaced from
   Slack, etc. — and a top-opportunities table with named contacts), and a footer byline. If the
   user has their own preferred document style/template, follow that instead. Keep the same
   summarize-don't-quote judgment from the enrichment step when writing this up — a published
   report is not the place for a verbatim internal Slack message.
8. **Publish the report.** If the user has access to an internal page-hosting service (e.g. WWT's
   my-pages, reachable via its MCP connector), publish there and share the resulting URL — that's
   the actual deliverable, don't wait to be asked to share it. Otherwise, write the finished HTML
   to a file and tell the user where it is.

## Notes

- This is read-only everywhere — CRM, ZoomInfo, documents, meetings, and Slack are all queried for
  information only; nothing is written, posted, or edited in any of them.
- Company count of 1-2 doesn't need the multi-subagent *research* treatment; do the SOQL/SOSL (and
  any enrichment lookups) directly yourself. The parallel-research-subagent approach earns its keep
  once you're covering enough companies (roughly 4+) that splitting the legwork across forks is
  worth the overhead. **The QA subagent is not part of that tradeoff — always run it, even for one
  company.** It has caught real errors on single-digit-company runs before (an arithmetic slip, and
  a large unexplained discrepancy between a CRM total and a vendor-reported figure), and it's cheap
  relative to the cost of publishing a wrong number.
- Not every environment has every connector (ZoomInfo, Glean, Slack, Webex, Google Drive, etc.).
  Missing connectors just mean a thinner narrative for that section, not a failed report — CRM
  data alone (partner status + pipeline) is still a complete, useful report on its own.
- If asked to update an existing partner report rather than create a new one, find the existing
  page (e.g. by name/slug on the hosting service) and upload a new version to it instead of
  creating a new one.
