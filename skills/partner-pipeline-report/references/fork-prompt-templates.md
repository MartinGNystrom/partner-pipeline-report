# Fork/subagent prompt templates

These are the actual prompts used for the first run of this skill (2026-07-20, six partners:
Claroty, Filigran, Doppel, Command Zero, Keyfactor, BlackCloak, for a stakeholder-facing partner
report at WWT). Reuse the structure, swap in the per-company facts discovered during the setup
pass.

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

Return a concise structured report (under 350 words). Do not write any files or publish
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
