# Phase 5 updated_docs — Parent Class Invite Refine

- **task_id**: `2026-04-13-parent-class-invite-refine`
- **seed_id**: `seed_6d7077aac472` (parent_seed_id=`seed_37b35654542f`, refinement supersede)
- **decisions 반영**: 72건 (D-01~D-50 + 사용자 대리 4건 + E-01~E-05 + 유지 13건)

갱신 대상은 **in-place 편집**으로 처리함. 원시드(`seed_37b35654542f`)의 parent-viewer-roadmap.md는 v2 기준으로 재작성되었으나 **역사 보존용 변경 로그 2026-04-12 행은 유지**.

---

## 1. `plans/parent-viewer-roadmap.md` (메이저 갱신 — in-place)

### 1.1 헤더

| 위치 | 변경 |
|---|---|
| 문서 제목 직하 메타 | v1/v2 시드 ID·인터뷰 ID 병기. v1은 "archived", v2(`seed_6d7077aac472`, `interview_20260413_075525`, ambiguity 0.10)는 "active" 표기 |

### 1.2 §0 핵심 명제

- 코드 포맷 "6자리" → "8자리"
- 페어링 흐름 "코드로 페어링한 뒤" → "학급 단위 8자리 코드로 가입한 뒤, 학급 명단에서 자녀를 셀프매칭으로 신청하고, 교사 승인을 통과한 경우에 한해"
- v2 변경 트리거 문단 신규 추가 (학생별 발급 부담 → 학급 1:코드 1 전환 근거, 완전 폐기 즉시 전환)

### 1.3 §1.1 페어링 · 인증 (v2) — 전 행 재작성

| 기존 → 신규 행 | 요지 |
|---|---|
| 페어링 방식 | 학생 카드 + QR → **학급 1:코드 1** (`ClassInviteCode`) |
| 코드 엔트로피 | 32⁶≈10⁹ → **32⁸≈10¹²** |
| 코드 수명 | 48h / maxUses 3 → **학기말 자동 만료 + 교사 수동 회전 + 무제한 maxUses** |
| brute-force 방어 | IP 5회/15분 + 코드 10회 즉시 만료 → IP 5회/15분 + 코드 50회/일 + **학급 100회/일**. 코드 자동 만료 제거 (D-13 DoS 벡터) |
| 교사 회전 알림 (신규 행) | 거부율 임계 초과 시 교사 배지만 v1 제공 |
| ParentSession 발급 시점 (신규 행) | signup 시점 (pending 조회용) |
| BoardMember 발급 시점 (신규 행) | approve 시점 (RLS 오염 방지) |
| pending HTTP 응답 (신규 행) | 200 OK + `{"status":"pending"}` payload flag |
| 동일 이메일 재가입 | 학부모 세션 누적 → **학급 코드 재입력 + 재승인** |

### 1.4 §1.2 상태 전이 · Revoke · 격리 (v2) — 섹션명 확장 + 상단 4행 신규

| 신규 행 | 요지 |
|---|---|
| `ParentChildLink.status` 유니언 | 2-value → 4-value (`pending`·`active`·`rejected`·`revoked`) |
| 상태 전이 머신 | 5개 전이 (signup → pending / approve / reject·auto_expire·rotate → rejected / revoke) |
| `revokedReason` enum 확장 | +3종 (rejected_by_teacher·auto_expired_pending·code_rotated). v1 값 재명명 (teacher_action→teacher_revoked, self_withdraw→parent_self_leave, + year_end) |
| `rejectedReason` enum 신규 | wrong_child / not_parent / other |
| 감사 필드 6종 | requestedAt·approvedAt·approvedById·rejectedAt·rejectedById·rejectedReason |
| Revoke 경로 | + **학급 코드 회전 시 pending 일괄 rejected** |
| Rotate vs active | active 유지, pending만 rejected |
| 자녀 본명 노출 | **active 이후만** |
| 학급 명단 마스킹 (신규 행) | §1.5 상술로 연결 |
| 거부 이메일 격리 (신규 행) | 교사 이름·연락처 미노출 |

### 1.5 §1.3 알림 · Tier (v2 재정의) — 전 행 재작성

| 변경 | 요지 |
|---|---|
| 교사 승인 인박스 알림 (신규 행) | D+0 배지 / D+3 이메일 / D+6 경고 / D+7 요약 |
| 발급 한도 (v2 재정의) | "자녀당 N" 폐지 (학급 코드 체제 의미 붕괴) |
| Free tier | "자녀당 2명" 삭제 |
| Pro tier | "자녀당 5명" 삭제, 주간 이메일 중심 |
| 교사 자유 메시지 (신규 행) | v1 미제공, v2 파킹 |
| 에스컬레이션 (신규 행) | v1 미제공, v2 파킹 |

### 1.6 §1.4 탈퇴 · 감사 (v2) — 부분 갱신

| 변경 | 요지 |
|---|---|
| revokedReason | teacher_action → teacher_revoked 등 신규 enum 재명명 반영 |
| 90일 내 재가입 | "교사 재발급" → "학급 코드 재입력 + 재승인" |
| 감사 필드 (v2 확장) | 승인자·발급자 분리: `ClassInviteCode.issuedById`(발급 교사) ≠ `ParentChildLink.approvedById`(승인 교사). 신규 6종 감사 필드 추가 |
| 거부 이메일 쿨다운 (신규 행) | 3회 초과 24h |

### 1.7 §1.5 승인 게이트 흐름 (v2 신규 — 섹션 신설)

6개 서브섹션 신설:
- §1.5.1 셀프매칭 흐름 시퀀스
- §1.5.2 학생 명단 노출 범위 (마스킹 규칙·rate limit)
- §1.5.3 승인 인박스 · SLA · 자동 만료 (7일 TTL / 24h 권고 / D+0·D+3·D+6·D+7 스케줄 / Cron KST 02:00)
- §1.5.4 거부 사유 · 안내 톤 (wrong_child / not_parent / other 고정 문구)
- §1.5.5 자동 만료 이메일 (고정 문구)
- §1.5.6 code_rotated 처리 (pending 일괄 rejected, active 유지)

### 1.8 §1.6 교사 UI (v2) — 섹션 신설

- 학급 설정 "학부모 액세스" 탭 3-섹션 (초대 코드 / 승인 인박스 / 연결된 학부모)
- 학생 카드 드롭다운 "학부모 초대" **제거** (D-35)
- 배지 색상 체계 (회색/노랑/빨강)

### 1.9 §2 데이터 모델 (Prisma)

| 변경 | 요지 |
|---|---|
| `ParentChildLink` | status 4-value + 감사 6종 + rejectedReason 필드 + revokedReason enum 확장. `@@index([requestedAt])` 추가 (Cron D+7 스캔) |
| `ParentInviteCode` | **제거** |
| `ClassInviteCode` | **신설** (code 8자리, classroomId, issuedById, expiresAt 학기말, maxUses null, rotatedAt) |
| RLS 정책 요약 | `status='active'` 조건 추가. `ClassInviteCode`는 교사만 직접 조회, 학부모는 검증 엔드포인트 경유 |

### 1.10 §3.1 페어링 흐름 (v2 시퀀스) — 전면 재작성

- 교사 학생 카드 드롭다운 → 학급 설정 탭으로 진입점 이동
- `POST /api/parent-invite-codes` → `POST /api/class-invite-codes`
- `POST /api/parent/redeem` → `POST /api/parent/signup` + `POST /api/parent/match/code` + `GET /api/parent/match/students` + `POST /api/parent/match/request`
- pending → approve / reject / auto_expire / code_rotated 4-way 분기 시퀀스

### 1.11 §4 parent 계층 미들웨어 (2종)

- §4.1 `parentAuthOnlyMiddleware` (v2 신규) — 매칭 전 전용
- §4.2 `parentScopeMiddleware` (v1 유지 + `status='active'` 조건 추가)
- ESLint 룰 2종 분기 명시

### 1.12 §5 자녀 범위 매트릭스

- 공통 원칙 #5 신규: "v2 pending 동안 자녀 본인 콘텐츠도 열람 불가"

### 1.13 §6 /parent/* PWA 구조

- `(authed-preActive)` 그룹 신설: match/select, match/pending, rejected
- `(authed-active)` 그룹 (status='active' 필수)로 분리

### 1.14 §7 작업 분할 (v2 — PV-1~PV-16, 총 33~34일)

- PV-1·2·3·5·8·12 **개정**
- PV-4·6·7·9·10·11 **유지**
- PV-13(셀프매칭)·PV-14(승인 인박스)·PV-15(Cron + 알림)·PV-16(코드 회전) **신설**
- 총 공수 27일 → 33~34일
- 1차/2차/v2+ 파킹 재분배

### 1.15 §8 수용 기준

17개 → 28개 항목. v2 고유 항목 11개 추가 (ClassInviteCode·status 전이·승인 인박스·pending HTTP 200·Cron·거부 사유 3종·코드 회전·pending 차단·parentAuthOnly·거부 이메일 격리·쿨다운).

### 1.16 §9 리스크 — 부분 갱신

| 행 | 변경 |
|---|---|
| parentScopeMiddleware 우회 | ESLint 룰 "parentScope or parentAuthOnly" 2종 분기 |
| Crockford brute-force | 6자리 10⁹ → 8자리 10¹². 학급당 100회/일 추가. 자동 만료 제거 |
| 사칭 시도 (신규) | 교사 승인 게이트 + 쿨다운 + 거부율 배지 |
| pending 적체 (신규) | D+0/3/6/7 4단 스케줄 |

### 1.17 §10 파킹 — 3개 항목 추가

- 교사 자유 메시지 (v2)
- 에스컬레이션 경로 (v2)
- 사칭 감지 SOP 고도화 (P2)

### 1.18 §11 변경 로그 — 2026-04-13 행 추가

`seed_6d7077aac472` (parent_seed_id=`seed_37b35654542f`, supersede) — v2 학급 단위 코드 + 셀프매칭 + 교사 승인 게이트 도입 전체 요지 15줄 이상 상세 로그.

---

## 2. `plans/seeds-index.md`

| 변경 | 요지 |
|---|---|
| 전체 Seed 개수 | 10개 → **11개** (refinement 1개 별도 행 추가) |
| Refinement 처리 안내 문단 | 상단 마스터 표 아래 신규 안내 |
| Seed 7 행 | "superseded by `seed_6d7077aac472` (2026-04-13)" 표기 추가 |
| Seed 7-v2 행 신설 | `seed_6d7077aac472` / `interview_20260413_075525` / ambiguity 0.10 / padlet |
| Seed 7 섹션 제목 | "v1: 2026-04-12 — superseded by seed_6d7077aac472 v2: 2026-04-13" 부제. 상단에 보존 알림 박스 |
| 🎯 Seed 7-v2 섹션 신설 | v1 대비 delta 23행 요약 표 + change_trigger + 유지 정책 + 파킹 추가 + 상세 설계 링크 |

---

## 3. `plans/phase0-requests.md`

| 변경 | 요지 |
|---|---|
| PV-v2-BUNDLE 블록 신설 | PV-12 블록 직후에 "학부모 페어링 v2" 섹션 신설 후 통합 진입 JSON 블록 1개 추가 |
| 블록 특징 | `refinement: true`, `parent_seed_id`, `seed_id`, `task_id`, `supersedes`(PV-1·2·3·5·8·12), `retains_from_v1`(PV-4·6·7·9·10·11), `new_cards`(PV-13·14·15·16), 24개 acceptance 항목 + 8개 v2_parked |
| 기존 PV-1~PV-12 블록 | **그대로 보존** (v1 역사). 주석으로 "위는 v1 역사 보존, 아래 v2가 활성 계약" 명시 |

---

## 4. `ideas-parking-lot.md`

| 변경 | 요지 |
|---|---|
| "parent-viewer v2 refinement" 섹션 신설 | seed_6d7077aac472 출처 표기. 6개 파킹 항목 추가 |
| E-01 거부 이메일 쿨다운 | plan 본문 §1.4 행에 반영되어 파킹 대상 아님 |
| E-02 재신청 deep link | plan 본문 §1.5.4에 반영되어 파킹 대상 아님 |
| E-03 자동 만료 Cron | plan 본문 §1.5.3에 반영되어 파킹 대상 아님 |
| E-04 교사 알림 채널 | plan 본문 §1.3·§1.5.3에 반영되어 파킹 대상 아님 |
| E-05 rejectedReason enum 저장 | plan 본문 §1.2·§2에 반영되어 파킹 대상 아님 |
| 신규 파킹 6개 | 교사 자유 메시지(v2) · 에스컬레이션(v2) · 사칭 감지 SOP(P2) · Kakao/Google OAuth·비번(v2+) · 부/모 합산 권한(v2+) · Parent 앱 네이티브(v3+) |

---

## 5. v1 원시드 처리 (수정 금지)

| 파일 | 상태 |
|---|---|
| `tasks/2026-04-12-parent-viewer-access/phase4/seed.yaml` | **수정 없음** — 원시드 보존 원칙 준수. plan 문서에서 supersede 표기만 적용 |

---

## 갱신 요약 카운트

- **plans/parent-viewer-roadmap.md**: 메이저 in-place 갱신 (섹션 §0·§1.1·§1.2·§1.3·§1.4·§2·§3.1·§4·§5·§6·§7·§8·§9·§10·§11 + 신규 §1.5·§1.6). 라인 증가 ~+250.
- **plans/seeds-index.md**: 마스터 표 1행 수정 + 1행 신규 추가 + 카운트 갱신 + Seed 7 섹션 상단 주의 박스 + Seed 7-v2 섹션 신규.
- **plans/phase0-requests.md**: PV-v2-BUNDLE 블록 1개 신규 추가 (PV-1~PV-12 v1 블록 보존).
- **ideas-parking-lot.md**: refinement 파킹 섹션 신규 (6개 항목).

**총 갱신 문서 수**: 4개 (모두 in-place, 신규 파일 없음).
