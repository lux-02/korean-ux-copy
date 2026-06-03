---
name: korean-ux-copy
description: Audits Korean user-facing copy in any codebase — detects 25 Korean UX anti-patterns (KA-100–KA-205): AI-like tone, translated SaaS phrasing (가지고 있다 / 되어진다 / ~를 통해 / ~에 대하여 translation-ese), solution-label claims, FOMO coercion, cold empty-result states, over-polite passive tone. Detection rate measured against a frozen oracle (100% on high+medium findings; ≥80% gate) on GPT-generated Korean SaaS fixtures. Produces one Kanana-powered, conversion-safe rewrite per finding; advisory info-tier findings reported separately. Provider boundary includes PII masking and a guaranteed no-network fallback. Triggers: 한국어 카피 감사, 번역투, 빈 화면 메시지, 오류 문구, CTA 수정, 솔루션 문구, 과잉 경어, 다시 실행, 재실행, 보완. For full pipeline (scan+analyze+rewrite+patch) use kcopy-pipeline instead. Do not use for general translation or legal rewriting without explicit review.
---

<!-- Canonical location: .claude/skills/korean-ux-copy/ (Claude Code)
     Mirror location:    .agents/skills/korean-ux-copy/ (Codex)
     Sync policy: always apply changes to both locations. -->

# Korean UX Copy

## Goal

Audit user-facing Korean text in a project, diagnose AI-like or awkward UX copy, and propose one Kanana-powered conversion-safe Korean rewrite per finding. Flow: deterministic diagnosis (tri-state scoped rules) → UX Copy Lift → Benefit Hook Lift → provider rewrite → edit-distance safety gate → report.

High and medium findings drive the risk score. Info-tier (stylistic) findings are reported in a separate section and excluded from the score.

## Workflow

1. Determine scope from the user request. Default: user-facing copy in source/docs/i18n files.
2. Inspect files with `rg`, excluding generated, dependency, secret, and build output.
3. Load references as needed:
   - Korean copy rules: `references/korean-copy-rules.md`
   - UX Copy Lift: `references/ux-copy-lift.md`
   - Benefit Hook Lift: `references/benefit-hook-lift.md`
   - Kanana setup: `references/kanana-provider.md`
   - Patch safety: `references/safety-and-patching.md`
4. Identify Korean copy targets: JSX/HTML text, labels, placeholders, aria/title/alt, toast/modal/error strings, i18n values, Markdown prose, metadata.
5. Diagnose by surface role: hero/CTA, button, error, empty state, form helper, toast, docs, SEO, legal/policy.
6. Apply tri-state rule scoping per target:
   - `hard_scoped` rules fire only on their declared surface role (e.g. KA-142 fires on `empty_state` only).
   - `soft_info` rules fire at `info` severity when the surface role does not match the rule's primary role (e.g. KA-114 on a button).
   - `unscoped` rules fire at their declared tier regardless of role (e.g. KA-122).
   - Route `info`-severity findings to `infoTargets`; do not include them in `flaggedTargets` or the risk score.
7. Run UX Copy Lift pass per target: record `uxGap`, `userState`, `nextAction`, `liftReason` using `references/ux-copy-lift.md`.
8. Run Benefit Hook Lift pass for conversion-eligible surfaces using `references/benefit-hook-lift.md`. One `hooked_safe` rewrite for hero/CTA/feature/pricing/SEO/FAQ surfaces. `safe_plain` for error/toast/empty-state/aria/legal.
9. Collect selected targets → mask PII (phone, RRN, email, address) before any provider call → run `scripts/kanana-ensure.mjs` → send one batch via `scripts/kanana-rewrite-batch.mjs` (max 12 items). Reject `rejected_claim_boundary` candidates. If `parseStatus: raw_text`, extract safe candidates manually and mark `review_recommended`. If no API key is configured, the provider falls through to `llm_fallback` with zero network egress.
10. Apply edit-distance safety gate: NFC-normalized similarity between original and rewrite. Change ratio <0.05 → reject (no-op rewrite). Change ratio >0.7 → downgrade `safe_auto` to `review_recommended`. Gate runs after the `manual_only` block; it never upgrades a label.
11. Produce itemized report in two sections:
    - **Risk findings** (high / medium / low): original, issue, UX gap, lift reason, rewrite, copy mode, hook labels, provider status, risk label, edit distance grade, file reference. Include at least top 8 when 8+ exist.
    - **Info (stylistic)** section: list info-tier findings with a note that they are advisory and excluded from the risk score.
    - Include provider disclosure: provider name, model alias if available, timestamp, whether content was masked.
12. For edits: dry-run diff first. Apply only safe copy-only changes on explicit user request.

## Detection Rules (v0.2)

| Rule | Name | Scope | Tier |
|---|---|---|---|
| KA-114 | `solution_label` — bare "...솔루션" category claim | soft_info | medium |
| KA-122 | `dont_miss_out` — 놓치지 마세요 / 단 하나의 기회 FOMO coercion | unscoped | medium |
| KA-142 | `no_result_exists` — cold empty-result with no next step | hard_scoped (empty_state) | medium |
| KA-160 | `passive_polite_overload` — stacked honorifics (~하실 수 있으십니다) | soft_info | info |
| KA-200 | `has_possession` — 가지고 있다 → 있다 (translation-ese) | soft_info | medium |
| KA-201 | `double_passive` — 되어진다 / 되어집니다 → 된다 | soft_info | medium |
| KA-202 | `via_through_overuse` — ~를 통해 → ~해서 | soft_info | low |
| KA-203 | `regarding_formal` — ~에 대하여 over-formal | soft_info | low |
| KA-204 | `in_respect_of` — ~에 있어(서) → ~에서 | soft_info | medium |
| KA-205 | `modal_can_headline` — ~할 수 있습니다 in headlines only | hard_scoped (hero) | low |

The KA-114/122/142/160 set extends the original KA-100–KA-152 rules; the KA-200 family adds im-not-ai Category A translation-ese detection scoped to short UX copy (KA-205 fires on headlines only, never on error/empty/form where "할 수 있어요" is good UX). KA-160 findings appear only in the info section; KA-200-family findings downgrade to `info` when their surface role does not match (soft_info).

## Commands

Resolve script paths relative to this skill directory.

```bash
node scripts/kanana-ensure.mjs
node scripts/kanana-setup.mjs
node scripts/kanana-rewrite-batch.mjs --input copy-targets.json
node scripts/kanana-rewrite-batch.mjs --input copy-targets.json --max-items 12
```

`kanana-rewrite-batch.mjs` is the only script that calls Kanana. Collect all targets first, then spend one request.

## Safety Defaults

- Do not read or modify `.env`, tokens, certificates, lockfiles, build output, or dependency folders.
- Do not send secrets, private user data, or full source files to Kanana.
- Do not rename variables, props, translation keys, routes, analytics events, or API contracts.
- Preserve placeholders: `{name}`, `{{count}}`, `${value}`, `%s`, ICU tokens.
- Do not invent features, integrations, discounts, guarantees, or support promises.
- Do not widen product capability or change legal meaning to make copy more clickable.
- Legal, privacy, refund, consent, and policy copy is report-only unless explicitly requested.
- Mask PII (phone numbers, resident registration numbers, email addresses, addresses) before any provider call.
- When no Kanana API key is configured, the provider falls back to `llm_fallback`; report includes `provider_status=llm_fallback` and no external network call is made.

## Output Shape

One primary `Rewrite` by default. `hooked_safe` for conversion surfaces, `safe_plain` for trust/recovery/legal.

```txt
File: app/page.tsx:18
Role: hero_headline
Severity: medium
Original: "당신의 비즈니스 여정은 여기서 시작됩니다"
Issue: Generic translated SaaS tone; vague journey metaphor.
UX Gap: Does not explain what the user can do next.
Benefit Hook: Turns a vague journey claim into a concrete first action.
Lift Reason: Replaces mood-setting copy with a specific benefit and next step.
Rewrite: "필요한 기능만 골라 바로 시작하세요"
Copy Mode: hooked_safe
Hook Labels: benefit|scene|cta
Provider: kanana_parsed|kanana_lines|kanana_lines_partial|provider_http_error|llm_fallback
Provider Candidate: accepted|rejected_claim_boundary|rejected_unmatched_id
Edit Distance Grade: safe_auto|review_recommended|rejected
Risk: review_recommended
Disclosure: { provider: "kanana" | "llm_fallback", timestamp: "...", masked: true|false }
```

Info-tier findings appear in a separate section at the end of the report:

```txt
--- Info (stylistic) — excluded from risk score ---
File: app/settings.tsx:44
Role: form_helper
Rule: KA-160 (passive_polite_overload)
Original: "해당 기능을 사용하실 수 있으십니다"
Note: Stacked honorific suffix may feel distant in product UI. Review recommended.
Suggestion: "이 기능은 설정에서 켤 수 있어요."
```
