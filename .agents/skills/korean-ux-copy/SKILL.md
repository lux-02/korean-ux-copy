---
name: korean-ux-copy
description: Audits Korean user-facing copy in any codebase — detects 15 Korean UX anti-patterns (KA-100–KA-152): AI-like tone, translated SaaS phrasing, cold error messages, weak empty states. 73% detection rate on GPT-generated Korean SaaS fixtures. Produces one Kanana-powered, conversion-safe rewrite per finding. Triggers: 한국어 카피 감사, 번역투, 빈 화면 메시지, 오류 문구, CTA 수정, 다시 실행, 재실행, 보완. For full pipeline (scan+analyze+rewrite+patch) use kcopy-pipeline instead. Do not use for general translation or legal rewriting without explicit review.
---

<!-- Mirror location: .agents/skills/korean-ux-copy/ (Codex)
     Canonical location: .claude/skills/korean-ux-copy/ (Claude Code)
     Sync policy: always apply changes to the canonical location first, then mirror here. -->

# Korean UX Copy

## Goal

Audit user-facing Korean text in a project, diagnose AI-like or awkward UX copy, and propose one Kanana-powered conversion-safe Korean rewrite per finding. Flow: deterministic diagnosis → UX Copy Lift → Benefit Hook Lift → Kanana batch rewrite → agent safety gate.

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
6. Run UX Copy Lift pass per target: record `uxGap`, `userState`, `nextAction`, `liftReason` using `references/ux-copy-lift.md`.
7. Run Benefit Hook Lift pass for conversion-eligible surfaces using `references/benefit-hook-lift.md`. One `hooked_safe` rewrite for hero/CTA/feature/pricing/SEO/FAQ surfaces. `safe_plain` for error/toast/empty-state/aria/legal.
8. Collect selected targets → run `scripts/kanana-ensure.mjs` → send one batch via `scripts/kanana-rewrite-batch.mjs` (max 12 items). Reject `rejected_claim_boundary` candidates. If `parseStatus: raw_text`, extract safe candidates manually and mark `review_recommended`.
9. Produce itemized report: original, issue, UX gap, lift reason, rewrite, copy mode, hook labels, provider status, risk label, file reference. Include at least top 8 findings when 8+ exist.
10. For edits: dry-run diff first. Apply only safe copy-only changes on explicit user request.

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

## Output Shape

One primary `Rewrite` by default. `hooked_safe` for conversion surfaces, `safe_plain` for trust/recovery/legal.

```txt
File: app/page.tsx:18
Role: hero_headline
Original: "당신의 비즈니스 여정은 여기서 시작됩니다"
Issue: Generic translated SaaS tone; vague journey metaphor.
UX Gap: Does not explain what the user can do next.
Benefit Hook: Turns a vague journey claim into a concrete first action.
Lift Reason: Replaces mood-setting copy with a specific benefit and next step.
Rewrite: "필요한 기능만 골라 바로 시작하세요"
Copy Mode: hooked_safe
Hook Labels: benefit|scene|cta
Provider: kanana_parsed|kanana_lines|kanana_lines_partial|provider_http_error
Provider Candidate: accepted|rejected_claim_boundary|rejected_unmatched_id
Risk: review_recommended
```
