# Partner Pipeline Report

A Claude Code plugin/skill for building a Salesforce partner-pipeline report — partner status
(tier, program status, partner manager), named sponsors/economic buyers, and pipeline (open +
lifetime closed-won) — for any list of partner/vendor companies. Runs one subagent per company in
parallel to gather and verify the data, then a QA subagent to catch arithmetic errors and
de-duplicate any deal bundled across two partners, before rendering a clean, theme-aware
standalone report.

First built 2026-07-20 for a six-partner cybersecurity/AI/quantum pipeline report at World Wide
Technology (WWT).

## Requirements

- A Salesforce CRM connector exposing SOQL/SOSL query tools (an MCP connector, or equivalent) that
  Claude Code can call. The skill discovers the connector's tool names dynamically via
  `ToolSearch`, so any SOQL/SOSL-capable connector should work — it doesn't need to be named
  exactly like WWT's internal one.
- (Optional) An internal page-hosting service to publish the finished report to a shareable URL
  (e.g. WWT's `my-pages`). Without one, the report is written to a local HTML file instead.

## Install

```
/plugin marketplace add MartinGNystrom/partner-pipeline-report
/plugin install wwt-partner-pipeline-report@wwt-partner-pipeline-report
```

## Usage

Ask Claude something like:

> Find Salesforce opportunities for these companies and build a partner report: Acme Corp,
> Widget Inc, ... I'm interested in partner status, sponsors, and pipeline.

The skill will:

1. Resolve each company to its Salesforce partner `Account` record(s)
2. Pull partner-status fields (tier, program status, partner manager) and pipeline aggregates
3. Launch one subagent per company to independently verify the numbers, pull named sponsors from
   `OpportunityContactRole`, and add brief company context
4. Run a QA subagent to check arithmetic, de-duplicate any opportunity shared across two partners
   (e.g. a bundled deal), and spot-check the live data
5. Render the report from `references/report-template.html` (or the user's own preferred style)
   and publish it, or write it locally

See `skills/partner-pipeline-report/SKILL.md` for the full workflow and the Salesforce query
patterns it relies on, and `skills/partner-pipeline-report/references/fork-prompt-templates.md`
for the exact subagent prompts used.

## Notes

- Entirely read-only against Salesforce — no writes are made at any point.
- The CRM field names referenced in `SKILL.md` (`Partner_Tier__c`, `Partner_Manager__c`, etc.)
  reflect WWT's Salesforce org. If you're pointing this at a different org, check the equivalent
  fields first — the general pattern (partner status lives on the Account; opportunities link to
  partners by name, not a dedicated lookup; sponsors live on `OpportunityContactRole`) is common,
  but exact field names will vary.

## License

MIT — see `LICENSE`.
