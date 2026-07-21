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
10. Query `Certifications_and_Specializations__c` where `Account__c = '<accountId>'` for any
    on-file certifications (Vendor__c, Certification_or_Specialization_Type__c). This object is a
    legacy one capped to a small set of old hardware vendors (Cisco/Citrix/EMC/HP/NetApp/VMware) —
    a zero-row result just means no legacy record exists, not that <Company> has no
    certifications; say it that way rather than asserting an absence.
11. If a ZoomInfo connector is available, pull award/recognition signals via
    `search_scoops`/`enrich_company_signals` with `scoopTypes: ["Award"]` (and optionally
    `"Partnership"`/`"Product Launch"`).
12. If a Glean connector is available, search with `app: "atc platform"` for "<Company>" to check
    for WWT Advanced Technology Center lab/demo content.
13. Write 1-2 sentences on other WWT service capabilities around <Company>'s product (implementation,
    managed services, professional services) if you know of any from public knowledge or the
    documents/meetings pulled above — otherwise state plainly that nothing specific was found,
    don't invent a service catalog.

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

## QA subagent prompt (launched once, after all per-company subagents report back)

```
QA pass on the compiled Salesforce partner-pipeline data for the <audience> report covering
<N> partners: <list>. <N> parallel subagents each gathered per-company data (pipeline aggregates
by stage, top open opportunities, named contacts, partner-status fields). Here is the compiled
dataset to verify:

<paste the full compiled per-company dataset: tier/status/manager, stage-by-stage pipeline
numbers, claimed open-pipeline totals, and any cross-company shared/bundled opportunity Ids>

YOUR TASKS:
1. Arithmetic check: verify each company's "open total" (sum of non-Closed stages) matches the
   stage-by-stage numbers given above. Show your math and flag any mismatch.
2. Compute the combined open pipeline across all companies, correctly de-duplicating any shared
   opportunity (state the Id and amount) so it's counted only ONCE in the combined total.
3. Compute combined lifetime Closed Won across all companies.
4. Using the CRM MCP tools, independently re-run one spot-check aggregate query per company
   (SELECT StageName, COUNT(Id), SUM(Amount) FROM Opportunity WHERE Name LIKE '%<company>%'
   GROUP BY StageName) and confirm the numbers above are still accurate right now (data may have
   changed). Flag any company where live data differs from what's stated above.
5. Sanity-check for other issues: any indication of a missed additional Account for any company
   (differently-named or regional entity)? Note anything internally inconsistent or suspicious.

Report back concisely (under 400 words): pass/fail per company's arithmetic, the corrected
combined totals, and any live-data discrepancies found. This is a verification pass only — do not
rewrite the report itself.
```

This QA pass caught a genuine $200 arithmetic error in the first compiled draft (one company's
open-pipeline total) — it earns its keep. Always apply the corrected numbers from QA before
writing the report, not the pre-QA draft numbers.
