# Fork/subagent prompt templates

These are the actual prompts used for the first run of this skill (2026-07-20, six partners:
Claroty, Filigran, Doppel, Command Zero, Keyfactor, BlackCloak, for a stakeholder-facing partner
report at WWT), plus the enrichment tasks added the same day to pull in ZoomInfo, documents,
meetings, and Slack. Reuse the structure, swap in the per-company facts discovered during the
setup pass.

## Per-company subagent prompt (one per partner, launched together in a single message)

```
Focus ONLY on <Company Name> (Account Id <accountId>[, and duplicate Account Id <id2> if one exists])
for the <report audience> partner report. Use the CRM MCP tools
(mcp__<connector>__soqlQuery / __find / __getRelatedRecords) which you already have access to.

I already have: Account partner status (Type=<Type>, Partner_Type__c=<...>, Partner_Tier__c=<...>,
Status__c=<...>, Partner_Manager=<Name, Title>[, WWT_Executive_Partner_Sponsors__c=<...>,
Sponsor_Name__c=<...>]), and aggregate pipeline by stage (<Stage: count/$amount, ...>), all found
via Name LIKE '%<Company>%'.

Your job:
1. Verify these aggregate numbers with your own SOQL query (SELECT StageName, COUNT(Id),
   SUM(Amount) FROM Opportunity WHERE Name LIKE '%<Company>%' GROUP BY StageName) — flag any
   discrepancy.
2. Pull the top 5 largest OPEN (non-Closed) opportunities by Amount, with Id, Name, Account.Name,
   StageName, Amount, CloseDate.
3. For those top opportunities, query OpportunityContactRole (child relationship on Opportunity,
   fields Contact.Name, Contact.Title, Role, IsPrimary) to find any named sponsors/economic
   buyers/champions. Report what you find, or state clearly if no contact roles are populated.
4. Note any other partner-status fields worth surfacing on the Account if populated (e.g.
   Registration_Number__c, Partner_Playbook_Link__c, Engaged_Partner_Manager__c,
   Partner_Priority__c).
5. Write 2-3 sentences describing <Company>'s relevance to <the report's theme, e.g.
   cybersecurity/AI/quantum> — this is public-knowledge company context, not from CRM.
6. If a ZoomInfo connector is available (search with ToolSearch for "ZoomInfo company" if
   unsure), pull 2-4 bullet points of recent firmographic/buying-intent signals for <Company>
   (funding, leadership changes, expansion, layoffs, product launches) via
   enrich_company_signals/enrich_news/enrich_scoops. Skip this step entirely if no ZoomInfo tools
   are found — don't treat it as a failure.
7. If a documents connector is available (e.g. Glean — search with ToolSearch for "Glean search"
   if unsure), search internal docs for "<Company>" and note anything materially relevant (sales
   collateral, competitive notes, QBR/EBR decks) by title, with a one-line summary of the relevant
   point — don't quote large blocks of text.
8. If a meetings connector is available (Glean meeting_lookup, a Webex Meetings connector, or
   similar), look for recent meetings mentioning <Company> and summarize key takeaways/action
   items at a high level.
9. If a Slack connector is available, search for recent internal mentions of "<Company>" and
   summarize any relationship-health signals (blockers, escalations, wins) found. Paraphrase,
   don't quote messages verbatim, and only include something if it's clearly fine to surface in a
   shared report — when in doubt, leave it out.
10. Read `wwt.com/partner/<company-slug>/expertise` (via Glean `read_document` or a web fetch — it's
    public) for WWT's own named certifications/awards WITH <Company> (e.g. "CrowdStrike Focused
    Partner — Americas — 2020"). This is about credentials WWT holds through <Company>'s partner
    program, NOT <Company>'s own public certifications or industry accolades as a company — don't
    conflate the two. If the page doesn't exist or has nothing listed, say so plainly.
11. Read `wwt.com/partner/<company-slug>/overview` (same access method) for the named WWT practice
    team (cross-check against CRM Partner_Manager__c/Account Owner), an "About <Company> & WWT"
    blurb, recent partner-branded articles, and a "<Company> in the ATC" section listing named labs
    with launch counts — this covers both the labs and other-service-capabilities parts of task 13
    below in one read.
12. Query `Certifications_and_Specializations__c` where `Account__c = '<accountId>'` as a secondary
    check (Vendor__c, Certification_or_Specialization_Type__c). This is a legacy object capped to a
    small set of old hardware vendors (Cisco/Citrix/EMC/HP/NetApp/VMware) — a zero-row result just
    means no legacy record exists, not that <Company> has no WWT-held certifications; the
    `/expertise` page in task 10 is the real answer here.
13. If task 11 didn't turn up ATC lab content, also try a Glean search with `app: "atc platform"`
    for "<Company>" directly. Write 1-2 sentences on other WWT service capabilities around
    <Company>'s product (implementation, managed services, professional services) from whatever
    task 11 turned up — otherwise state plainly that nothing specific was found, don't invent a
    service catalog.
14. Answer explicitly: does <Company> have a demo in the ATC? Run a general Glean search (no `app`
    filter) for "<Company> ATC demo" across email/SharePoint/meeting notes/Slack in addition to
    tasks 11/13's public-page and `atc platform` checks — the internal search often reveals the
    real, current state (e.g. a demo unit already staged but not yet run) that the public partner
    page won't show. State clearly which applies: a working ATC demo/lab exists, hardware/plans
    exist but no demo has been run yet, or no ATC presence at all.
15. Determine if <Company> is publicly traded or private (check for a ticker, or use ZoomInfo
    `search_companies`'s `companyType` field). If public, a one-line note (ticker + rough scale) is
    enough. If private, pull a funding/investor picture: ZoomInfo `search_companies` for employee
    count/revenue, `enrich_company_signals` (`signalTypes: ["SCOOP","NEWS"]`) and `account_research`
    for funding/investor signals, and a `WebSearch` for "<Company>" crunchbase funding investors as
    a public cross-check (watch for name collisions with similarly-named but unrelated companies —
    confirm city/website match). Write 2-4 sentences: what's been raised and by whom (if anything),
    and what that implies about relative financial strength given the deal size/engagement WWT is
    considering with <Company> specifically — name the risk plainly if a small or unfunded partner
    is being asked to carry outsized deal or delivery weight, and name the offsetting strength
    (e.g. real enterprise logos already using the product) if that's also true. If every source
    comes back empty, conclude "likely bootstrapped/self-funded" rather than reporting a data gap.
16. Identify 3-6 real competitors to <Company> via `WebSearch` (e.g. "<Company> competitors
    alternatives" plus a query naming <Company>'s specific product category) — do NOT rely on
    ZoomInfo `find_similar_companies` as the primary source, it has returned firmographically
    similar but functionally irrelevant results before. For each competitor, run the same CRM checks
    as <Company> itself: SOSL `FIND {<Competitor>} IN NAME FIELDS RETURNING Account(Id, Name, Type,
    Industry, Website, Partner_Type__c, Partner_Tier__c, Status__c)` for partner status, and
    `SELECT StageName, COUNT(Id), SUM(Amount) FROM Opportunity WHERE Name LIKE '%<Competitor>%'
    GROUP BY StageName` for their own WWT pipeline/closed-won history. A competitor with no Account
    record at all is a real, reportable finding ("no formal WWT relationship") — don't skip it. For
    any non-public competitor, a quick `WebSearch` funding/investor check adds scale context. State
    plainly which competitor (if any) is ahead of <Company> on WWT program maturity (tier, pipeline,
    deal history) versus which have no WWT relationship at all, and what that means for <Company>'s
    competitive position specifically.

Return a concise structured report (under 500 words). Do not write any files or publish
anything — just report findings back in your final message.
```

Notes from the first run:
- If a company has a same-name collision in CRM (e.g. "Doppel" vs. an unrelated "Dopple"), call
  out the wrong one explicitly in the prompt so the subagent doesn't conflate them.
- If a company has duplicate Account records (happened with Command Zero), give the subagent both
  IDs and ask it to check opportunities against both, plus flag the duplication itself as a
  finding — don't silently pick one.
- If a partner's product shows up bundled with another partner's in the same Opportunity Name
  (happened with "Newmont_Doppel & Blackcloak"), flag it in both companies' prompts so each
  subagent knows not to treat it as an exclusive deal, and so the compiling step knows to
  de-duplicate it.
- One subagent returned a garbled non-answer ("Only X's deep-dive is still outstanding") despite
  having done the tool calls. If a subagent's final message doesn't contain the actual requested
  data, message it directly asking it to synthesize and report the findings it already gathered —
  don't re-run its tool calls from scratch in a new agent.
- Tasks 6-9 (ZoomInfo/documents/meetings/Slack) are additive, not required — a subagent should say
  "no ZoomInfo connector found" or similar and move on rather than treating a missing connector as
  a blocker. Don't pad the report with a boilerplate "not available" line for every company if it
  just adds noise; a brief note is only worth including where the gap actually matters.
- Tasks 10-13 (certs/awards/labs/capabilities) are also additive. Don't let a subagent present
  `Partner_Skill__c` as a confirmed per-partner record — schema inspection found no Account/Partner
  lookup field on it, only a link up to `WWT_Skill__c`. If a subagent tries to match one to
  `<Company>` by name, it should say so explicitly as an unverified guess, not a finding.
- **Corrected after the first CrowdStrike run got it wrong**: "certifications and awards" means
  what WWT itself holds/won through the partner's program (e.g. "WWT is a CrowdStrike Focused
  Partner," "WWT named CrowdStrike Flex Partner of the Year"), not the vendor's own public
  certifications or industry accolades as a company. The first pass used a ZoomInfo Award-scoop
  search (which surfaces the vendor's own recognitions) and concluded "nothing found" — wrong
  direction entirely. The real answer was sitting on WWT's own public
  `wwt.com/partner/crowdstrike/expertise` page the whole time: two named credentials with region and
  year. Always check the `/expertise` and `/overview` pages first for this section; don't reach for
  ZoomInfo here at all.
- **Added after a single-company Halocon run (2026-07-21)**: the public `wwt.com/partner/.../
  overview` page and `atc platform` Glean search only tell you about a *published* ATC lab — they
  said nothing about Halocon at all. A plain Glean search for "Halocon ATC demo" (no app filter)
  surfaced the real, internal-only state: an executive-briefing recap describing a demo hardware
  unit already staged at a specific ATC building, with running the actual demo still listed as an
  unstarted next step. Always run both the published-page check and the general internal search —
  relying on only one would have reported either a false "no ATC presence" or missed the nuance that
  the demo hasn't actually happened yet.
- **Same Halocon run**: ZoomInfo `search_companies` and `account_research` both came back with zero
  funding/investor signals for the company. That absence was itself the finding — combined with a
  small headcount/revenue (23 employees, ~$2.9M revenue) and an internal note about the company's
  founder-funded origin, it supported a plain "likely bootstrapped, no institutional capital cushion"
  conclusion, which mattered because the company was being discussed internally as a potential
  anchor for a much larger, multi-quarter engagement. Report the absence-of-funding-signal finding
  itself, framed against what the partner is being asked to carry — don't just say "no data found"
  and stop.
- **Added after the same Halocon run, competitive-landscape pass**: ZoomInfo `find_similar_companies`
  on Halocon returned 15 "lookalikes" (Inflection AI, Lanthorn.ai, Pryon, etc.) all matched on a
  misclassified "Business Intelligence (BI) Software" industry code — none were actual physical-
  security competitors. A plain `WebSearch` for "Halocon competitors AI physical security camera
  threat detection alternatives" immediately surfaced the real set (Verkada, Coram AI, IntelliSee,
  ZeroEyes, Evolv Technology, Avigilon). Of those, only Verkada and Coram AI had any WWT Account
  record at all; the CRM check found Verkada as an Approved (Tier 4) Active partner with real
  (if mixed — $117K closed-won against $2.44M closed-lost) deal history and Coram AI as a nominal
  Active partner with zero recorded Opportunities. A follow-up `WebSearch` confirmed Verkada is
  extremely well-capitalized ($757M raised, $5.8B valuation, backed by Sequoia/CapitalG/Nvidia) —
  the write-up named this plainly as Halocon's most credible competitive threat inside WWT, not a
  neutral fact to log next to the others.

## QA subagent prompt (always launched, even for a single company)

This is a standing rule, not a judgment call based on company count: run this pass every time,
whether the report covers one company or a dozen. Skip nothing about it just because there's no
cross-company total to compute.

```
QA pass on the compiled Salesforce partner-pipeline data for the <audience> report covering
<N> partner(s): <list>. Here is the compiled dataset to verify:

<paste the full compiled dataset: tier/status/manager, stage-by-stage pipeline numbers, claimed
open-pipeline totals, top opportunities with contacts, any enrichment findings (certs/awards/
labs/capabilities), and any cross-company shared/bundled opportunity Ids if N > 1>

YOUR TASKS:
1. Arithmetic check: verify each company's "open total" (sum of non-Closed stages) matches the
   stage-by-stage numbers given above. Show your math and flag any mismatch.
2. If N > 1: compute the combined open pipeline across all companies, correctly de-duplicating any
   shared opportunity (state the Id and amount) so it's counted only ONCE in the combined total,
   and compute combined lifetime Closed Won across all companies.
3. Using the CRM MCP tools, independently re-run at least one spot-check aggregate query per
   company (SELECT StageName, COUNT(Id), SUM(Amount) FROM Opportunity WHERE Name LIKE '%<company>%'
   GROUP BY StageName) and confirm the numbers above are still accurate right now (data may have
   changed). Flag any company where live data differs from what's stated above.
4. Spot-check 2-3 top opportunities and their named contacts against live data to confirm nothing
   was misquoted, and re-verify any certifications/enrichment finding that came from a direct query
   (e.g. Certifications_and_Specializations__c).
5. If any enrichment source reported a figure that doesn't reconcile with the CRM-derived number
   (e.g. a vendor's own reported "bookings" total vs. a SOQL Closed-Won sum), don't let both numbers
   sit side by side unexamined — attempt an explanation (different measurement definitions? opps
   that reference the product without the company name in Opportunity.Name, missed by a Name LIKE
   sweep?) and report your confidence in it. It's fine to conclude "no clean explanation found" if
   that's the honest answer.
6. Sanity-check for other issues: any indication of a missed additional Account for any company
   (differently-named or regional entity)? Any close date or figure that looks like a data-entry
   artifact rather than a real forecast? Note anything internally inconsistent or suspicious.

Report back concisely (under 400 words): pass/fail on the arithmetic, live-data confirmation
results, the contact/enrichment spot-check outcome, any discrepancy explanation attempted, and any
other flags. This is a verification pass only — do not rewrite the report itself.
```

This QA pass has caught real issues even on small runs: a genuine $200 arithmetic error in a
six-company draft's open-pipeline total, and — on a single-company CrowdStrike run — a large,
real discrepancy between WWT's own internal "CrowdStrike OEM Bookings" figures and the SOQL
Closed-Won total that was worth investigating rather than silently reporting both. Always apply
the corrected numbers and any discrepancy finding from QA before writing the report, not the
pre-QA draft.
