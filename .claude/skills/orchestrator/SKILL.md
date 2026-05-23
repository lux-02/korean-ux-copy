---
name: kcopy-pipeline
description: Korean UX copy 전체 파이프라인을 실행하는 오케스트레이터 스킬. Scanner → Analyzer → Refiner → Patcher 4단계를 조율하여 대상 코드베이스의 한국어 UX 카피를 진단하고 전환 안전한 패치를 생성합니다. 한국어 카피 전체 리뷰, 파이프라인 실행, 스캔+분석+리라이트+패치 통합 실행, 재실행, 업데이트, 보완, 결과 개선, 이전 결과 기반 작업, 파이프라인 다시 돌려줘, 전체 감사, 이전 스캔 업데이트 요청 시 반드시 사용하세요. 단일 파일 수정이나 빠른 카피 확인은 korean-ux-copy 스킬을 사용하세요.
---

# K-Copy Pipeline Orchestrator

**실행 모드:** 서브 에이전트 순차 파이프라인 (Pipeline)

에이전트: scanner → analyzer → refiner → patcher

## 역할

Scanner → Analyzer → Refiner → Patcher 4개 에이전트를 순차 조율하여 한국어 UX 카피 감사와 패치 생성을 end-to-end로 실행한다. 중간 산출물은 `_workspace/`에 파일 기반으로 공유된다.

---

## Phase 0: 컨텍스트 확인

파이프라인 시작 전 기존 실행 상태를 확인한다.

```bash
ls _workspace/ 2>/dev/null && echo "EXISTS" || echo "NEW"
```

| 상태 | 처리 |
|------|------|
| `_workspace/` 없음 | 초기 실행 — Phase 1부터 전체 실행 |
| `_workspace/` 있음 + 새 입력/경로 | `_workspace/` → `_workspace_prev/` 이동 후 새 실행 |
| `_workspace/` 있음 + 부분 수정 요청 | 해당 Phase부터 재실행 (아래 부분 재실행 참조) |

---

## Phase 1: 스캔 (Scanner Agent)

에이전트 정의: `.claude/agents/scanner.md`

**지시:**
대상 경로를 스캔하여 사용자에게 보이는 한국어 텍스트를 추출한다.
1. `rg --type-add 'web:*.{tsx,jsx,ts,js,vue,svelte,html,md,mdx,json,yaml,yml}' -t web -l '\p{Hangul}'`로 파일 필터링
2. 각 파일에서 CopyTarget 추출 (JSX 텍스트 노드, 속성, i18n 값, Markdown 산문)
3. `{name}`, `{{count}}`, `${value}`, `%s` 등 토큰 보호
4. 결과를 `_workspace/01_scan_result.json`에 저장

**입력:** ScanRequest (경로, 제외 패턴)
**출력:** `_workspace/01_scan_result.json`

완료 조건: JSON 파일 생성, `targets` 배열 포함, `schemaVersion` 필드 존재

---

## Phase 2: 분석 (Analyzer Agent)

에이전트 정의: `.claude/agents/analyzer.md`

**지시:**
`_workspace/01_scan_result.json`을 읽어 각 CopyTarget을 진단한다.
1. 표면 역할 분류 정교화 (hero, button, error, empty_state 등)
2. AI 투 Korean, 번역 톤, 과장 클레임, 약한 UX 가이드 감지
3. 규칙 ID 포함한 Finding 생성 (KQ-001~KQ-007)
4. 스코어카드 계산 (ai_likeness, clarity, cultural_fit 등)
5. 결과를 `_workspace/02_analysis_result.json`에 저장

**입력:** `_workspace/01_scan_result.json`
**출력:** `_workspace/02_analysis_result.json`

완료 조건: JSON 파일 생성, `findings` 배열 및 `scorecard` 객체 포함

---

## Phase 3: 리라이트 (Refiner Agent)

에이전트 정의: `.claude/agents/refiner.md`

**지시:**
`_workspace/02_analysis_result.json`을 읽어 우선순위 타겟에 리라이트를 생성한다.
1. 전환 적격 표면에 Benefit Hook Lift 적용 (references/benefit-hook-lift.md 로드)
2. Kanana 설정 시: `scripts/kanana-ensure.mjs` 실행 후 `scripts/kanana-rewrite-batch.mjs`
3. Kanana 불가 시: 동일 클레임 게이트로 `llm_fallback` 사용
4. 각 후보에 `provider`, `tonePreset`, `confidence` 레이블 추가
5. 결과를 `_workspace/03_rewrite_result.json`에 저장

**입력:** `_workspace/02_analysis_result.json`
**출력:** `_workspace/03_rewrite_result.json`

완료 조건: JSON 파일 생성, `candidates` 배열 포함, 각 항목에 `provider` 필드

---

## Phase 4: 패치 (Patcher Agent)

에이전트 정의: `.claude/agents/patcher.md`

**지시:**
`_workspace/03_rewrite_result.json`과 `_workspace/02_analysis_result.json`을 읽어 dry-run diff와 리포트를 생성한다.
1. contextHash로 소스 범위 검증
2. `safe_auto`, `review_recommended`, `manual_only` 레이블 부여
3. unified diff 프리뷰 생성
4. Markdown 리포트 생성 (요약 → 스코어카드 → Finding → Diff → 스킵 항목)
5. `_workspace/04_patch_plan.json`과 `_workspace/04_report.md`에 저장
6. `--apply` 플래그가 없으면 적용하지 않음

**입력:** `_workspace/03_rewrite_result.json`, `_workspace/02_analysis_result.json`
**출력:** `_workspace/04_patch_plan.json`, `_workspace/04_report.md`

완료 조건: 두 파일 모두 생성, 리포트에 요약 + 스코어카드 + diff 프리뷰 포함

---

## 데이터 흐름

```
사용자 요청
    │
    ▼
Phase 0: _workspace/ 상태 확인
    │
    ▼
Phase 1: scanner
    └→ _workspace/01_scan_result.json
         │
         ▼
    Phase 2: analyzer
         └→ _workspace/02_analysis_result.json
              │
              ▼
         Phase 3: refiner
              └→ _workspace/03_rewrite_result.json
                   │
                   ▼
              Phase 4: patcher
                   └→ _workspace/04_patch_plan.json
                   └→ _workspace/04_report.md
                        │
                        ▼
                   사용자에게 리포트 요약 출력
```

---

## 에러 핸들링

| 상황 | 처리 |
|------|------|
| Phase 1: 한국어 파일 없음 | "Korean copy not found" 리포트, 정상 종료 |
| Phase 1: 파일 파싱 실패 | 파일 스킵, 이유 기록, 계속 |
| Phase 2: Finding 없음 | 이상 없음 리포트, 계속 |
| Phase 3: Kanana 실패 | llm_fallback 자동 전환, 리포트에 명시 |
| Phase 4: contextHash 불일치 | 해당 타겟 스킵, 이유 기록 |
| 중간 파일 손상 | 해당 Phase 재실행 제안 후 중단 |

Phase 실패 시 1회 재시도 후 재실패하면 해당 결과 없이 계속 진행, 리포트에 누락 명시.

---

## 부분 재실행

특정 Phase부터 재실행 가능:

```bash
# Phase 2부터 (기존 스캔 결과 재사용)
# 전제: _workspace/01_scan_result.json 존재

# Phase 3부터 (기존 분석 결과 재사용)
# 전제: _workspace/02_analysis_result.json 존재

# Phase 4만 (기존 리라이트 결과 재사용)
# 전제: _workspace/03_rewrite_result.json 존재
```

---

## 최종 출력 형식

```
스캔 완료: {파일 수}개 파일, {타겟 수}개 한국어 카피 대상

스코어카드:
- AI 투 지수: {0.0~1.0}
- 번역 톤: {0.0~1.0}
- 명확성: {0.0~1.0}
- 문화적 적합성: {0.0~1.0}

상위 Finding ({n}개):
[Finding 목록]

리포트: _workspace/04_report.md
패치 계획: _workspace/04_patch_plan.json

적용하려면: 패치 계획 검토 후 apply 모드로 재실행하세요.
```

---

## 테스트 시나리오

**정상 흐름 (Next.js 프로젝트):**
1. 입력: "이 프로젝트의 한국어 카피 전체 감사해줘" + 경로
2. Phase 0: `_workspace/` 없음 → 초기 실행
3. Phase 1: 23개 CopyTarget 발견
4. Phase 2: 15개 Finding, ai_likeness 0.72
5. Phase 3: 12개 리라이트 후보 (llm_fallback, Kanana 미설정)
6. Phase 4: dry-run diff, `_workspace/04_report.md` 생성
7. 출력: 요약 + 스코어카드 + 상위 Finding + 리포트 경로

**에러 흐름 (Kanana 실패):**
1. Phase 3: `kanana-ensure.mjs` 실패
2. 자동 llm_fallback 전환, 클레임 게이트 동일 적용
3. 리포트에 `provider_status: llm_fallback` 명시
4. 정상 완료

**부분 재실행 (리라이트만 다시):**
1. 입력: "분석은 됐고 리라이트만 다시 해줘"
2. Phase 0: `_workspace/02_analysis_result.json` 존재 확인
3. Phase 3부터 재실행, 기존 Phase 1~2 결과 재사용
4. 완료
