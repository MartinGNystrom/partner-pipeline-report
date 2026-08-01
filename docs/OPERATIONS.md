# Partner Pipeline Report skill — operational notes

Relocated from Claude Code memory (`reference_partner_pipeline_report_skill`) on 2026-08-01 during a context
cleanup. Memory now keeps a short pointer plus the non-obvious facts; the
operational detail lives here, in the repo it describes.

---

The `wwt-partner-pipeline-report` skill (invoked as `/partner-pipeline-report`) is a real git repo
Martin owns at github.com/MartinGNystrom/partner-pipeline-report, installed locally as a Claude Code
plugin/marketplace at `~/.claude/plugins/marketplaces/wwt-partner-pipeline-report` — **edit that repo
directly**, not the empty stub at `personal-context/skills/partner-pipeline-report/` (no content,
not a submodule — just a leftover placeholder). Version is tracked in `.claude-plugin/plugin.json`
and `.claude-plugin/marketplace.json` (kept in sync) plus a git tag per release,
`wwt-partner-pipeline-report--vX.Y.Z`. There's no CHANGELOG.md — release history lives in commit
messages/tags only.

**2026-07-27, v1.5.0: separated the QA subagent's process narrative from the report's actual
content**, per Martin's instruction. The report body (BLUF, combined summary, per-company sections,
callouts, competitive-landscape) must state only business/technical facts about the partners
themselves — never narrate *how* a number was checked or corrected ("QA confirmed...",
"spot-checked against live data..."). That process narrative — what QA verified, what it corrected
and why, what it couldn't reconcile, which connectors were unavailable — now goes exclusively into
a new "Report processing notes" appendix at the very end of `report-template.html` (muted styling,
dashed rule, visually subordinate), never mixed into the sections above it. The QA subagent prompt
in `references/fork-prompt-templates.md` was updated to narrate freely in its final message rather
than compress for the report's sake; SKILL.md's workflow steps 6-7 spell out the test: if a sentence
is about the partner, it's body content; if it's about the report-building process, it's appendix
content.

**2026-07-27, v1.6.0: fixed the CRM field-semantics bugs from the Rubrik report.** SKILL.md's "Key
CRM facts" section and both subagent prompt templates now say plainly: Opportunity has partner
lookup fields `OEM__c`/`Services_Primary_OEM__c` in the WWT org (an earlier version wrongly claimed
no such field existed, forcing a `Name`-only search that undercounts), and `Amount` is gross profit,
not revenue — `Total_Revenue__c` is revenue, roughly 10x apart, and the two must never be combined
or relabeled. Also documented: the WWT `StageName` picklist (5 open + `Closed Won`/`Closed
Lost`/`Closed:  No Bid` with a double space), and that `OpportunityLineItem` is empty org-wide there.
Also linkified the fixed (non-templated) source URLs already cited in the docs — the
[wwt.com/corporate/awards-and-recognitions/overview](https://wwt.com/corporate/awards-and-recognitions/overview)
cross-check page and the CrowdStrike `/expertise` page example — plus the
[Rubrik briefing](https://my-pages.apps.wwt.com/nystromm/rubrik-partner-pipeline-briefing) now cited
as the source of the fix.

**Gotcha discovered rebuilding the Rubrik report (2026-07-27): pushing to the marketplace repo does
NOT update the installed plugin Skill invocations actually use.** Editing/committing/pushing files in
`~/.claude/plugins/marketplaces/wwt-partner-pipeline-report` only updates that git checkout — the
`Skill` tool loads from a separate versioned cache at
`~/.claude/plugins/cache/wwt-partner-pipeline-report/wwt-partner-pipeline-report/<version>/`, pinned
by `~/.claude/plugins/installed_plugins.json`'s `installPath`/`version`/`gitCommitSha` fields (and
mirrored, less critically, in `known_marketplaces.json`'s `lastUpdated`). After pushing v1.6.0, a
`Skill` invocation still loaded the stale v1.3.0 cache from a week earlier — confirmed by the tool
result's own "Base directory for this skill" line, which names the exact cache path/version in use.
No plugin-update tool was found via `ToolSearch`; there may be a `/plugin` CLI command that handles
this properly, but the manual fix (safe, low-risk, all local config/cache files) is: `rsync -a
--exclude='.git'` the marketplace checkout into a new
`.../cache/wwt-partner-pipeline-report/wwt-partner-pipeline-report/<new-version>/` directory, then
edit `installed_plugins.json`'s entry for `wwt-partner-pipeline-report@wwt-partner-pipeline-report`
to point `installPath`/`version`/`gitCommitSha`/`lastUpdated` at it. **Check this every time after
pushing a fix to this skill (or any other installed plugin) before trusting a `Skill` invocation** —
the tool gives no warning that it's serving a stale cached copy. **Refined 2026-07-30: this fix
does not take effect within the same already-running session** — re-invoking `Skill` right after
editing `installed_plugins.json` (same session that had the plugin loaded before the edit) still
reported the old `installPath`/version in its "Base directory for this skill" line, even though the
JSON on disk was already correct. The plugin/skill resolution is evidently cached in-memory at
session start. The fix is real and will be picked up cleanly by a **new** Claude Code session/job —
don't conclude the rsync+JSON fix failed just because the current session doesn't see it; verify by
reading the cache directory and `installed_plugins.json` directly instead of trusting an in-session
`Skill` call as the check.

**2026-07-30, v1.9.0: added ARMOR framework alignment and joint-strategic-themes enrichment**,
prompted by reviewing the real Rubrik EBC/partner-summit meeting notes from 2026-07-28 (see
[[rubrik-partnership]]). Two gaps found: (1) the meeting-transcripts enrichment bullet only
captured retrospective relationship-health signals, not the forward-looking joint GTM motion,
coalition/program status, and named next-steps-with-owners a summit/EBC/QBR-style meeting actually
produces — now called out explicitly as legitimate narrative content. (2) No discipline existed for
cross-checking a security/AI partner's own claims (or a meeting's in-room framing of them) against
WWT's own ARMOR framework rather than transcribing them at face value. New optional "AI/
cybersecurity framework alignment (ARMOR)" section added — only applies to cybersecurity/AI
partners, cites/reconciles against an existing ARMOR vendor matrix if one exists rather than
re-deriving scores, and demonstrates the discipline with the real triggering example: a vendor's
"immutable air-gapped architecture" was credited in the room to Infrastructure Security when it's
actually a Data Protection control (backup immutability), with the vendor's own agent-runtime
sandboxing explicitly deferred to a third party — a category conflation the skill now explicitly
tells writers to catch rather than repeat. Workflow steps 4 and 7 updated to reference both
additions. Version bumped in both `plugin.json` and `marketplace.json`, tagged
`wwt-partner-pipeline-report--v1.9.0`, pushed to `main` directly (Martin's own repo, same pattern as
v1.5.0-v1.8.0).

**2026-07-27, v1.7.0: made the report body a pure business document, with a second mandatory QA
pass to enforce it.** Triggered by Martin catching that the v1.6.0-fixed Rubrik rebuild still leaked
CRM/field language into its business-facing body (`Partner_Manager__c field itself is blank`,
`found only via OEM__c, not Name`, `CRM-derived number`, etc.) despite the v1.5.0 appendix-separation
rule already existing. New hard rule in SKILL.md: the body must never name CRM/Salesforce/SOQL, an
object/field name, or any report-building-process language ("verified," "spot-checked," "CRM-derived")
— only plain business facts, with one exception (citing an external research source like ZoomInfo/
Slack/a named meeting for a business signal is normal sourcing, not "processing"). **Two QA passes now
run as explicitly separate steps, per Martin's follow-up instruction** — a data-QA pass (step 6,
unchanged from v1.5.0: arithmetic, live-data re-confirmation, discrepancy investigation) and a new
business-language-compliance pass (step 8, added in v1.7.0: scans the drafted body for banned-term
violations and forces a rewrite) — never combined into one review. `fork-prompt-templates.md` has a
full prompt template for the new pass with a concrete banned-term list.

**Real-world first run (Rubrik briefing, 2026-07-27, published report gone through v1→v2→v3):**
the compliance pass found 11 violations in the v2 body (Salesforce/CRM named directly, six leaked
field names, "blank field"/"found via"/"CRM-derived" process phrasing) spanning the pipeline-summary
caption, the partner-manager status line, two flags, one competitor-table cell, one competitive
flag, two named-contact cells, and the flag below the top-opportunities table — all rewritten to
plain business language in v3 (e.g. "Partner_Manager__c field itself is blank" → "no partner manager
formally assigned"). One borderline case (deal-stage labels like "Pursuit"/"Discover" as table
values) was judged acceptable as ordinary sales vocabulary, not exposed schema, and left alone.

**2026-07-27, v1.8.0: adopted "Partner Pulse" as the standing report title/series name.** Martin
asked for a better recurring title than the ad hoc "Partner Pipeline Briefing"; offered "Partner
Pulse" / "Alliance Radar" / "Alliance Compass" as options, he picked **Partner Pulse**. Every report
is now titled "**[Company] Partner Pulse**" for a single company (e.g. "Rubrik Partner Pulse") or
"**Partner Pulse: [Company A], [Company B], ...**" for multiple — both the `<title>` tag and the
`doc-kicker` line in `report-template.html`. This is a naming-only change, no content/workflow
impact.

**2026-07-30, v1.10.0: three more additions, folded back in from the same-day Gigamon rebuild**
(see [[gigamon-partnership]]). (1) New "WWT's own partner-tier mechanics" section — check the
partner's actual trailing-12-month revenue against WWT's own tier-promotion thresholds
(Approved/Select/Advantage/Strategic, each gated by a minimum-revenue figure) rather than reporting
current tier as static fact; the real finding that justified this was Gigamon's revenue already
clearing the Advantage threshold while its CRM tier still said Select. (2) The ARMOR
framework-alignment section got an explicit 4-level rating rubric (Strong / Moderate / Weak — named
not demonstrated / Not addressed), a rule to check whether another named WWT partner already covers
a "not addressed" domain (a real complementarity found: Gigamon's Infrastructure Security strength is
the exact inverse of the backup-vendor matrix's Infrastructure Security weakness), and an explicit
note that mapping against a non-WWT-authored framework (MITRE ATLAS was the example — Martin first
asked for that, then corrected to ARMOR) must be labeled as the report's own synthesis, not a joint
claim. (3) New "Visualizing the data" section — build a handful of charts via the `dataviz` skill
when publishing to an HTML destination; when the report already has an established single-accent
style (like Martin's own "Wire" personal style), use accent-opacity/intensity as the encoding instead
of introducing a new multi-hue categorical palette, with every entity's name as a direct text label
so identity still isn't color-alone. Also documents a real self-caught mistake worth remembering: an
early draft shaded a pipeline-by-stage chart light-to-dark implying deal-maturity progression, which
wasn't actually a confirmed ordering — fixed by re-encoding on dollar magnitude (which was supported)
before publishing, not by asserting the unconfirmed claim anyway.

Related: [[rubrik-partnership]], [[salesforce-pipeline-via-glean]], [[gigamon-partnership]]
