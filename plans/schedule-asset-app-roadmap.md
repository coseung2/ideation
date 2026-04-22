# schedule-asset-app 로드맵 (일정 + 강제성 자산관리 Android 앱 · 외부 신규 프로젝트)

> 작성일: 2026-04-22
> Seed: `seed_20260422_schedule_asset_app_a7c3` (task `2026-04-22-schedule-asset-app`)
> Interview: `self-driven-interview-2026-04-22-schedule-asset-app` (Ouroboros MCP 미가용 → 에이전트 self-driven, ambiguity 0.00)
> **Destination**: `internal` 잠정 (phase7에서 외부 신규 destination 등록 후보 결정 — 예: `schedule-asset-app/` 또는 `coercive-budget/`)
> 전제 문서:
> - `ideation/tasks/2026-04-22-schedule-asset-app/phase1/exploration.md`
> - `ideation/tasks/2026-04-22-schedule-asset-app/phase2/sketch.md`
> - `ideation/tasks/2026-04-22-schedule-asset-app/phase3/decisions.md`
> - `ideation/tasks/2026-04-22-schedule-asset-app/phase4/seed.yaml`

---

## 0. 핵심 명제

> **Banksalad·Toss류 기록형 가계부가 끝낸 지점("보여주고 경고만 한다")에서 본 앱은 시작한다. 20~30대 한국 소비자가 월 저축 목표에서 이탈하지 않도록 3층 강제성 스택 — (1) 오픈뱅킹 주간 사전 격리, (3) Android Accessibility 기반 쇼핑·배달 앱 차단 행동 마찰, (2) 기부처 화이트리스트 수취 페널티 이체(v1.1) — 을 적용한다. 일정 탭은 월급일·카드결제일·목표 마감일을 자동 주입해 "일정+자산"이 단일 표면을 이루게 한다. MVP는 2인 풀스택 팀이 마이데이터 사업자 등록 없이도 합법적으로 출시 가능한 범위(1층+3층)로 제한한다.**

본 프로젝트는 Aura-board(`padlet/`)와 **완전 독립**이며, ideation 산출물은 설계 문서만이다. 실제 코드는 외부 저장소 (예: `schedule-asset-app/` 또는 `coercive-budget/`)에 위치하며, phase7 dispatcher에서 `internal` fallback 또는 새 destination 등록으로 분기 결정한다.

---

## 1. MVP 범위 (layers 1+3)

### 1층 — 사전 격리 (Open Banking 주간 자동이체)

| 축 | 결정 |
|---|---|
| 주체 | 금융결제원 오픈뱅킹 REST API 이용 (핀테크 이용기관) |
| 기본 빈도 | **금요일 저녁 주 1회** (D6 결정) |
| 일일 모드 | opt-in (설정에서 활성화). 건당 수수료·일일 한도 리스크 때문에 비기본 |
| 수취 계좌 | 사용자 본인 명의 **잠금 저축 계좌** (`AssetFreeze`) |
| 베타 검증 | 개발자 개인 계좌로 이체 로직 검증 (클로즈드 베타) |
| 엔티티 | `SavingRule` (trigger + action + target_account) + `AssetFreeze` (lock_level, release_condition, release_date) |

### 3층 — 행동 마찰 (Android Accessibility 앱 차단)

| 축 | 결정 |
|---|---|
| 구현 | Android Accessibility Service — 지정 쇼핑·배달 앱 실행 감지 시 delay/block overlay |
| 발동 조건 | 예산 초과 상태 (`DailyBudget.overflow=true`) 또는 `always` 설정 |
| 권한 헬스체크 | 주 1회 자동 확인. 해제 감지 시 즉시 인앱 고지 |
| 14일 무권한 | 3층 기능 자동 비활성화 + 인앱 공지 (사일런트 실패 방지, 신뢰 유지) |
| 엔티티 | `BlockedAppRule` (app_package, action=delay\|block, trigger=budget_exceeded\|always, delay_seconds) |

### 2층은 v1.1 이월 — 아래 §4 참조

---

## 2. 데이터 수집 전략

- **Android MVP only**. iOS는 v2 축소판 (수동 입력 + 1층만, SMS·Accessibility 없음).
- **이중 경로 병렬 구현** (Play Store `READ_SMS` 승인 리스크 대응):
  - 주 경로: `READ_SMS` 기반 SMS 파서
  - 대체 경로: `NotificationListener` 알림창 접근 권한 기반 동일 파서
  - 두 경로는 파서 로직 공유로 1인주 수준 추가 비용
- **지원 카드사 5개사** (국내 점유율 상위 ≈ 70%):
  - 신한 / 삼성 / 현대 / KB / BC
  - 나머지는 수동 입력 fallback
- 파서 유지비는 MVP 범위 내 제한. 포맷 변경 대응은 핫픽스 정기 릴리스.
- 거래 원문(SMS body) 및 계좌·카드 번호는 **로컬 SQLCipher DB only**. 서버 영속화 금지.

---

## 3. 일정 탭 (financial-event-aware schedule)

"일정+자산" 결합이 본 앱 고유 가치. 캘린더가 빈 껍데기면 경쟁 가계부앱 대비 차별화 실종 (D5 결정).

- **재무 이벤트 자동 주입**:
  - 월급일 (`payday`)
  - 카드결제일 (`card_billing`)
  - 목표 마감일 (`goal_deadline`)
  - 규칙 실행 로그 (`rule_execution` — 1층 격리/3층 차단 발동 결과)
- 사용자 일반 일정 (`user_event`)은 보조 — 빈 칸을 채우는 역할이지 메인이 아님
- `ScheduleEvent.type` enum으로 구분. Sketch 모델에 이미 정의되어 추가 구현 비용 낮음

---

## 4. v1.1 — 2층 (페널티 이체)

| 축 | 결정 |
|---|---|
| 엔티티 | `PenaltyPledge` 활성화 — 사전 승인 금액 + 수취 기관 + 트리거 조건 |
| **수취 대상** | **지정 기부처 화이트리스트 only** (공익법인). 제3자 개인 계좌 금지 (전자금융거래법 리스크) |
| 트리거 | 월 저축 목표 미달 시 익월 초 자동 이체 |
| 파트너 리서치 | MVP 베타 기간 중 수행 — 굿네이버스·유니세프·월드비전 등 API 또는 정기후원 등록 경로 조사 |
| 법인화 조건 | v1.1 진입 전 법인 등록 여부 판단 (핀테크 이용기관 정식 등록은 법인 권장) |
| 계약 필요 | 수취 기관과 정기후원·공익법인 계약 선행 |

---

## 5. 보안·규제 경계

| 축 | 결정 |
|---|---|
| 데이터 저장 | **로컬 우선** (SQLite + SQLCipher). 서버는 익명 메타·사용자 id·동기화 토큰만 |
| 원문 서버 저장 | **금지** — 계좌번호·카드번호·SMS 본문 모두 로컬 암호화 DB에만 |
| 암호화 키 | Android Keystore 관리. 디바이스 외부 유출 없음 |
| 마이데이터 사업자 | **미등록** (자본금 5억 초과 — 2인 팀 범위 외) |
| 오픈뱅킹 등록 | 핀테크 이용기관 정식 등록은 **정식 런칭 전** 완료. 클로즈드 베타는 개발자 개인 계좌 |
| 전자금융거래법 | 위임결제·자금세탁 리스크 회피 — 페널티 수취는 공익법인 only |
| 법인화 시점 | v1.1 진입 전 판단 분기. 개인 개발자도 등록 가능하나 법인 권장 |
| 개인정보보호법 | 서버 sync 시 해시·토큰 only. Raw identifier DTO 금지 |

---

## 6. 수익 모델

| 단계 | 모델 | 근거 |
|---|---|---|
| **MVP ~ v1.1** | **완전 무료, 광고·추천 없음** ✅ (사용자 확정 2026-04-22) | 초기 사용자 확보 최우선. 강제성 앱이 과금까지 얹으면 사용자 부담. 철학("사용자 저축 성공") 정합 |
| v2 재검토 후보 | 페널티 수수료 1~3% 또는 월 구독 3,900원 | 실사용 데이터·리텐션 검증 후 재논의 |

시드 기록: `revenue_model.mvp = "none"` / `revenue_model.v2_candidates = ["penalty_fee_1to3pct", "subscription_3900_month"]`.

---

## 7. 팀 운영

| 축 | 결정 |
|---|---|
| 구성 | 풀스택 2인 — **안선민, 심보승** |
| 역할 분담 | **영역 분할 없음** ✅ (사용자 확정 2026-04-22). 둘 다 풀스택 |
| 태스크 오너십 | 태스크 보드·이슈 단위 교대 소유. 고정 경계 미설정 |
| 동기화 | **주간 동기 미팅 필수** |
| 코드 리뷰 | **PR 리뷰 필수** — 브랜치 충돌 방지 규칙 |
| bus factor | 2 유지 목표 (역량 균등 성장) |
| 리스크 | 스케줄링·브랜치 충돌 비용 ↑ → 초반 주간 동기화 + 코드리뷰 규칙으로 상쇄 |

시드 기록: `team.mode = "fullstack_shared"` / `team.convention = "weekly_sync + PR review required"`.

---

## 8. 파킹된 항목

| 항목 | 이유 | 재검토 시점 |
|---|---|---|
| OS 레벨 실시간 카드 결제 차단 | 카드사·PG 제휴 필수, 사실상 불가능 | 무기한 |
| iOS 완전판 | SMS 불가 + Screen Time 제약 → 3층 중 2개 반쪽 | v2 (축소판 manual+1층만) |
| OCR 영수증 인식 | MVP 스코프 외. 수동 입력 피로가 베타 데이터에서 드러나면 우선순위 상향 | v1.1 재검토 |
| 가족·커플 예산 공유 | 사회적 감시 결합 리텐션 후보 | v2+ |
| 제3자 계좌 페널티 송금 | 전자금융거래법 위임결제 리스크 | 법률 해석 변경 시 (사실상 봉인) |
| 마이데이터 사업자 등록 | 자본금 5억 범위 초과 | 법인화·투자 유치 시 |

---

## 9. 다음 단계

- **phase6 handoff-writer**: receiving-project phase0 request + handoff_note 작성. padlet feature 파이프라인 아니므로 `plans/phase0-requests.md`는 미갱신.
- **phase7 dispatcher**: destination 결정 분기
  - (a) `internal` fallback — 신규 외부 destination 등록 전 잠정 보관
  - (b) 새 destination "schedule-asset-app" 또는 "personal-finance" 등록 후 INBOX 생성 → 라우팅
- **phase0 준비 사항** (외부 저장소 초기화 시):
  - Android Kotlin + Jetpack Compose
  - SQLite + SQLCipher
  - Open Banking REST 샌드박스 환경 (금융결제원)
  - Gradle·Detekt·Ktlint 초기화
  - 주간 sync + PR 규칙 문서화

---

## 변경 로그

- 2026-04-22: Initial — seed `seed_20260422_schedule_asset_app_a7c3` 반영. MVP layers 1+3 / v1.1 layer 2 확정, 5개 카드사 파서, Android 단독 MVP, 완전 무료 MVP~v1.1, 풀스택 공유 2인 팀.
