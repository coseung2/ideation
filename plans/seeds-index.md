# Aura-board — 생성된 Seed 인덱스

> 작성일: 2026-04-12
> 생성 방식: `/ouroboros:interview` → `/ouroboros:seed` (4개 세트)
> 인터뷰 답변은 에이전트 자율 판단으로 진행, 사용자 승인은 사후 이 문서로 확인

---

## 📋 전체 Seed 13개 (Aura-board 10개 + refinement 1개 + 외부 프로젝트 2개)

> **Refinement 처리**: Seed 7(v1)은 2026-04-13 refinement 파이프라인 통과 후 `seed_6d7077aac472` (v2)로 **supersede**되었다. 원시드(`seed_37b35654542f`)는 감사 이력으로 읽기 전용 보존, v2가 활성 스펙이다. parent-viewer-roadmap.md는 v2 기준으로 in-place 갱신됨.

| # | 주제 | Seed ID | Interview ID | Ambiguity | 연관 plan | Destination |
|---|---|---|---|---|---|---|
| 1 | 그림보드 × 학생 라이브러리 | `seed_91bcd4e99efb` | `interview_20260412_064317` | 0.177 | `drawing-board-library-roadmap.md` | padlet |
| 2 | Tier/요금제 수익 모델 | `seed_8967b77d7759` | `interview_20260412_071332` | 0.113 | — (정책 승계원) | padlet |
| 3 | 행사 신청 보드 v3 미결 | `seed_43fdf181262f` | `interview_20260412_071856` | 0.141 | `event-signup-roadmap.md` | padlet |
| 4 | 식물관찰일지 운영 규칙 | `seed_a79a953f188c` | `interview_20260412_072354` | 0.159 | `plant-journal-roadmap.md` | padlet |
| 5 | Canva 통합 6항목 재검토 | `seed_408901a564e8` | `interview_20260412_072729` | 0.129 | `implementation-roadmap.md` | padlet + aura-canva-app |
| 6 | Breakout Room 보드 | `seed_bb1d4eb1c442` | `interview_20260412_102210` | 0.147 | `breakout-room-roadmap.md` | padlet |
| 7 | 학부모 읽기 전용 뷰어 액세스 (v1) | `seed_37b35654542f` | `interview_20260412_111153` | 0.074 | `parent-viewer-roadmap.md` | padlet → **superseded by `seed_6d7077aac472`** (2026-04-13) |
| 7-v2 | 학부모 페어링 v2 — 학급 코드 + 셀프매칭 + 교사 승인 (refinement) | `seed_6d7077aac472` | `interview_20260413_075525` | 0.10 | `parent-viewer-roadmap.md` (in-place v2 갱신) | padlet |
| 8 | Canva Publisher 수신 엔드포인트 & PAT | `seed_26af361e92b7` | `interview_20260412_124111` | 0.121 | `canva-publisher-receiver-roadmap.md` | padlet |
| 9 | 공문 붙임파일 자동 생성 CLI (gongmun-assistant v1) | `seed_3f953e443fa1` | `interview_20260412_131559` | 0.177 | `gongmun-assistant-v1-roadmap.md` (신규) | **gongmun-assistant** (외부 신규 프로젝트) |
| 10 | 코지 P2E 목장 게임 (mallang-ranch-p2e) | `seed_f85d1cb8a245` | `interview_20260413_070940` | 0.132 | `mallang-ranch-p2e-roadmap.md` (신규) | **mallang-ranch** (외부 신규 P2E) |
| 11 | 과제 배부 보드 (assignment-board) | `seed_38c34e91bf28` | `interview_20260414_131412` | 0.083 | `assignment-board-roadmap.md` (신규) | padlet |
| 12 | 수행평가 자동채점 파이프라인 (assessment-autograde) | `seed_0badf1e571bc` | `interview_20260415_224854` | 0.10 | `assessment-autograde-roadmap.md` (신규) | padlet |

모든 seed ambiguity ≤ 0.2 (임계치) — seed-ready 상태. `ooo run`으로 실행 가능.

> **Seed 1·3·4·6의 개별 "학부모 열람" 단편 언급은 Seed 7 매트릭스(`parent-viewer-roadmap.md#5`)로 통합·재기재**. 각 feature plan은 Seed 7을 single source of truth로 참조한다.

---

## 🎯 Seed 2: Tier/요금제 수익 모델 (이번 세션 핵심 결정)

**가격 구조** — 교사 1인 구독 단일 유료 티어

| Tier | 가격 | 반 | 스토리지 | 영상 | Canva |
|---|---|---|---|---|---|
| **Free** | ₩0 | 1반 | 500MB | ❌ | oEmbed 링크만 |
| **Pro** | 월 ₩9,900 또는 연 ₩99,000 (2개월 무료) | 5반 | 20GB | 총 10시간 (Cloudflare Stream) | Autofill·Export + 월 100회 → **재설계됨**: 고비용 월 10 jobs × 50 records + 저비용 무제한 (분당 60 rate limit) |

**결제·운영**
- 14일 무료 체험
- Stripe 1차, 토스페이먼츠 v2 고려
- 학교 일괄 결제(세금계산서) Enterprise v2+ 파킹
- 쿼터 초과 시 **안내 모달 + 업그레이드 CTA**, 자동 결제 X
- 월 결제 언제든 해지 가능 (방학 중 해지 패턴 대응)

---

## 🎯 Seed 3: 행사 신청 보드 v3 미결

| 미결 | 결정 |
|---|---|
| 학번·학년 검증 | **완전 자기 기입(honor system)**. 교사가 심사 시 육안 검증 |
| 동명이인 | 학년+반+학번 조합으로 식별 |
| 스팸 방어 | 3중: **쿠키 토큰**(1h/3건) + **ipHash**(5min/10건 → hCaptcha) + **requireApproval 기본 ON** + **선발 N×5 상한 자동 마감** |
| 링크 재발급 | 기존 Submission 불변 유지. 재발급은 "새 유입 차단"용 |
| 공동 담당 | **단일 owner + BoardCollaborator** (co-editor·reviewer 역할). 결제·삭제·양도는 owner만 |
| 마감 후 제출 | 차단. `waitlisted`는 심사 결과로 정원 초과자 대기용 (교사 수동 승급) |
| 장기 roster 연동 | v1.5 — 교사가 반 학생 명단 등록했다면 자동 매칭·하이라이트 |

---

## 🎯 Seed 4: 식물관찰일지 운영 규칙

| 미결 | 결정 |
|---|---|
| 관찰 기간 | **교사가 학급마다 시작/종료일 직접 지정**. 시스템이 식물 종 기반 기본값 제안(예: 강낭콩 선택 → 90일 후) |
| 식물 사망 | **재시작 모델**. `isActive=false`로 아카이브(기록 보존) + 새 StudentPlant 생성 |
| UNIQUE | `(studentId, classroomId, isActive=true)` — 활성 식물 1개 제약 |
| 식물 교체 | 교사 승인 시 허용. 이전 기록 아카이브 |
| 타임라인 | "첫 번째 강낭콩(4/1~4/20 떡잎 단계 고사) → 두 번째 강낭콩(4/22~)" 표시 |
| 상호 열람 | **반 공개 기본** + 학생이 개별 관찰 `isPrivate` 토글 |
| 교사 정책 | 학급 단위 기본값 "비공개로 시작" 설정 가능 (민감 학급) |
| 학부모 | 자녀 것만 항상 열람(privacy 무관) |
| 팀 관찰 | v1 미지원 |
| "응원" 같은 상호작용 | v2 파킹 |

---

## 🎯 Seed 5: Canva 통합 6항목 재검토

**핵심 재설계**

| 항목 | 기존 → 변경 |
|---|---|
| 기준 단말 | iPad → **갤럭시 탭 S6 Lite** |
| iframe 정책 | ≤3 → **LRU 3개 + 가상화**. 카드 30~50개 배치 가능, 뷰포트+최근 탭한 것만 활성, 나머지는 Vercel Image 프록시 썸네일 |
| Free/Pro 구분 | 없음 → **Free는 oEmbed만**, Pro는 Connect API |
| Canva API 한도 | 월 100회 (모든 콜) → **고비용/저비용 분류**: 고비용 월 10 jobs × 50 records(Autofill·Resize·Editing), 저비용 무제한 (rate limit 분당 60) |
| P0-② Publisher | 학생·교사 혼재 → **v1은 교사 프라이빗 앱만** (학교 인증, Canva for Education). 학생 발행·공개 Marketplace 앱은 v2 |
| P1-④ Quiz 파싱 | 단일 경로 → **3단 하이브리드**: (1) canvaGetDesignContent 규칙 파싱 → (2) GPT-4o-mini LLM fallback → (3) 교사 편집 UI (필수). 10페이지 30초 SLA |
| **P1-⑦ Teamspace** | 권고 폐기 → **공식 폐기 확정** |

---

## 🎯 Seed 6: Breakout Room 보드 (2026-04-12 추가)

**핵심 결정 요약** (decisions.md Q1~Q7)

| 축 | 결정 |
|---|---|
| 신규 엔티티 | `BreakoutTemplate` + `BreakoutAssignment` 2종만 (BreakoutGroup 신설 금지) |
| Section | **재활용** — T0-① 섹션 격리 인프라·`Section.accessToken` 재사용. `layout=="breakout"` 시 i18n "모둠 N" 라벨 |
| 시스템 템플릿 | **7+1종** (KWL·브레인스토밍·아이스브레이커·찬반 토론·Jigsaw·모둠 발표 준비·갤러리 워크 + Pro예비 6색 모자). 월드카페는 **v2 파킹** |
| Tier | **Free 3종**(KWL·브레인스토밍·아이스브레이커) / **Pro 5종** + 커스텀 Free 3개·Pro 무제한·학교 공용은 Pro 전용 (Seed 2 정책 승계) |
| 배포모드 | 3종: `link-fixed` · `self-select`(초기 1회) · `teacher-assign` |
| 열람모드 | **기본 own-only** · `peek-others`는 갤러리 워크/발표 준비만. 교사는 항상 전체 접근(상위 RBAC) |
| 모둠 기본값 | 4모둠 · 정원 6 · 상한 10모둠 |
| 복제 방식 | **복사(독립)**. 모둠 간 완전 격리. 교사 "모든 모둠에 복제" **단발 액션** 버튼. 템플릿 원본 수정은 **역전파 X** |
| teacher-pool | **보드 레벨 단일 섹션** (모둠 수와 독립) |
| 학생 이동 (v1) | **교사 재배정만 허용**. 학생 셀프 이동은 v2 파킹 |
| 기본 가시성 | 반 공개(isPublic=true) |

상세 설계: `plans/breakout-room-roadmap.md` (작업 BR-1~BR-9).

---

## 🎯 Seed 7: 학부모 읽기 전용 뷰어 액세스 (v1: 2026-04-12 — **superseded by seed_6d7077aac472 v2**: 2026-04-13)

> **⚠️ 본 섹션은 v1 시드(`seed_37b35654542f`) 원본을 감사 이력 목적으로 보존.**
> **2026-04-13 refinement 결과 `seed_6d7077aac472`가 v2로 발행되어 본 시드를 대체한다. 활성 스펙은 v2이며, 아래 Seed 7-v2 섹션을 참조.**

**핵심 결정 요약 (v1, 역사 보존)** (`phase3/decisions.md` 17건 통합)

| 축 | 결정 (v1) |
|---|---|
| 페어링 | **Crockford Base32 6자리 코드 + QR** (`crypto.randomBytes` CSPRNG, 32⁶≈10⁹). 48h / maxUses 3 |
| 인증 | **매직 링크(이메일 OTP)만 v1** (비번·OAuth 파킹). 유효 15분. `ParentSession` 7일 |
| 신규 엔티티 | **4종**: `Parent` / `ParentChildLink` / `ParentInviteCode` / `ParentSession` + `BoardMember.role += "parent"` |
| 학부모 1인당 자녀 | **상한 5명** (tier 무관) |
| 발급 한도 | Free **자녀당 2** / Pro **자녀당 5** |
| 알림 | 인앱 배지(Free/Pro) + **주간 이메일 요약(Pro 전용, 월 09:00 KST, 활동 0건 스킵)**. push·알림톡 v2 |
| v1 범위 | **순수 read-only**. 댓글·좋아요·응원 이모지 모두 v2 파킹 |
| 클라이언트 | 스마트폰 포트레이트 PWA `/parent/*` (320~430px). iframe 금지, SWR 60s 폴링, WebSocket 비활성 |
| 성능 예산 | TTI < 2s LTE / < 3s 3G, 첫 뷰포트 < 500KB, 썸네일 < 200KB |
| Revoke SLA | **≤ 60s** (SWR 폴링). `revokedAt` SET → 401 → 자동 로그아웃. 즉시 revoke(<1s, Redis 블랙리스트) v2 |
| brute-force 방어 | **이중 rate limit**: IP 5회/15분 잠금 + 코드 10회 실패 즉시 만료 (`failedAttempts`). Vercel Edge + Upstash Redis |
| 격리 — 타 학생 | parent 토큰으로 타 학생 studentId 직접 호출 → **403** (E2E 필수) |
| 격리 — 타 학부모 | parentA로 parentB 링크 조회 → **404** (RLS `parent_id=auth.parent_id()` 단방향) |
| 방어선 | API 필터링(1차) + DOM 마스킹(보조) + RLS(3중) |
| 썸네일 | presigned 또는 RLS-scoped query (URL 추측 차단). T0-④ 승계 |
| 탈퇴 | **soft delete 고정** + 90일 후 Cron 익명화(email SHA-256, displayName "탈퇴한 학부모"). ParentChildLink 감사 보존 |
| 감사 | `ParentChildLink.issuedById`·`revokedAt`·`revokedReason`(teacher_action/self_withdraw) |
| 자녀 이름 | **본명 기본 노출** (가족 맥락) |
| 학부모 간 이메일 | **개별 발송 (BCC 금지)**, 상호 이름·이메일 비노출 |

**자녀 범위 매트릭스(cross-cutting — Seed 1·3·4·6 통합)**:

| Seed | 콘텐츠 | 학부모 열람 |
|---|---|---|
| 1 그림보드/라이브러리 | `StudentAsset` | 자녀 본인 자산 전체 (isSharedToClass·isPrivate 무관) |
| 3 행사 보드 | `Board(event-signup)` · `Submission` | 자녀 학급 이벤트 메타 + 자녀 본인 Submission·피드백만 (참가자 명단·득점 제외) |
| 4 식물관찰일지 | `PlantObservation` · `StudentPlant` | 자녀 본인 식물 전 단계·관찰 (isPrivate 무관) + 교사 코멘트 |
| 6 Breakout | `BreakoutMembership` · Section/Card | 자녀 본인 세션 결과·제출물만, teacher-pool 제외, peek-others여도 타 모둠 노출 X |
| 공통 숙제 | `Submission` | 자녀 본인 제출 + 교사 코멘트 |
| Quiz 점수 | — | **v1 비노출** |

상세 설계 (v1, 역사 보존): `plans/parent-viewer-roadmap.md` 변경 로그 2026-04-12 행. 작업 카드는 PV-1~PV-12 v1 형태였음.

---

## 🎯 Seed 7-v2: 학부모 페어링 v2 — 학급 코드 + 셀프매칭 + 교사 승인 (2026-04-13 refinement, `seed_6d7077aac472`)

> **Refinement origin**: `seed_37b35654542f` (v1) → `seed_6d7077aac472` (v2). `task_id=2026-04-13-parent-class-invite-refine`, ambiguity 0.10 (≤0.2 게이트 통과). 인터뷰 72건 결정(D-01~D-50 + 사용자 대리 4건 + E-01~E-05 + 유지 13건).

**change_trigger**: 학생별 개별 코드 발급(v1)은 학생 수에 비례한 교사 운영 부담을 유발. v2는 학급 단위 단일 코드 + 학부모 셀프 온보딩 + 교사 승인 게이트로 전환해 발급 부담을 학생 수 → 1로 감소, 사칭 차단은 승인 게이트로 유지.

**핵심 결정 요약** (v1 대비 delta)

| 축 | v1 → v2 결정 |
|---|---|
| 페어링 단위 | 학생 1:코드 1 → **학급 1:코드 1** (`ClassInviteCode` 신설) |
| 코드 포맷 | Crockford Base32 **6자리** → **8자리** (32⁸ ≈ 10¹²) |
| 코드 수명 | 48h / maxUses 3 → **학기말 자동 만료 + 교사 수동 회전 + 무제한 maxUses** |
| brute-force 방어 | IP 5회/15분 + 코드 10회 즉시 만료 → IP 5회/15분 + 코드 50회/일 + **학급 100회/일**. 코드 자동 만료 제거 (DoS 벡터) |
| 온보딩 플로우 | 교사 발급 → 학부모 코드 입력 → 활성화 → **학부모 signup → 학급 명단에서 자녀 셀프매칭 → 교사 승인 게이트 → active** |
| `ParentChildLink.status` | `"active"\|"revoked"` 2값 → **`"pending"\|"active"\|"rejected"\|"revoked"` 4값** |
| revokedReason enum | `teacher_action`/`self_withdraw` → **+ `rejected_by_teacher`·`auto_expired_pending`·`code_rotated` (v1 값은 `teacher_revoked`·`year_end`·`parent_self_leave`로 재명명)** |
| rejectedReason enum (신규) | — → **`wrong_child`\|`not_parent`\|`other`** (자유 텍스트 v2 파킹) |
| 감사 필드 | `issuedById`·`revokedAt`·`revokedReason` → **+ `requestedAt`·`approvedAt`·`approvedById`·`rejectedAt`·`rejectedById`·`rejectedReason` 6종 추가** |
| pending 상한/TTL | — → **학부모당 3건 / TTL 7일** (D+7 Cron 자동 만료 KST 02:00) |
| 승인 SLA | — → **권고 24h, hard SLA 없음**. D+0 배지 / D+3 이메일 / D+6 경고 / D+7 요약 |
| 학생 명단 마스킹 | — → **반 + 번호 + 성+O+끝글자** ("3반 12번 김O민"). 프로필 사진 비노출 |
| 거부 이메일 | — → **사유 3종 드롭다운 고정 문구 + 교사 정보 비노출 + 재신청 deep link + 3회 초과 24h 쿨다운** |
| ParentSession 발급 | redeem 성공 시 → **signup 시점** (pending 조회용) |
| BoardMember 발급 | redeem 성공 시 → **approve 시점** (RLS 오염 방지) |
| pending HTTP 응답 | — → **200 OK + `{"status":"pending"}`** payload flag |
| 교사 UI | 학생 카드 드롭다운 + 학급 설정 탭 → **학급 설정 "학부모 액세스" 탭 3-섹션** (초대 코드 / 승인 인박스 / 연결된 학부모). 학생 카드 드롭다운 제거 |
| Free/Pro 발급 한도 | 자녀당 2/5 → **한도 개념 폐지** (학급 코드 체제에서 의미 붕괴). Pro 혜택은 주간 이메일 전용 |
| 미들웨어 | `parentScopeMiddleware` 단일 → **`parentAuthOnlyMiddleware`(매칭 전) + `parentScopeMiddleware`(매칭 후, `status='active'`) 2종 분기** |
| RLS | `studentId ∈ parent.children` → **`studentId ∈ parent.children WHERE status='active'`** |
| 마이그레이션 | — → **완전 폐기 즉시 전환** (v1 미배포, D-46) |
| 작업 카드 | PV-1~PV-12 (27일) → **PV-1·2·3·5·8·12 개정 + PV-13~16 신설 (33~34일)** |
| 유지 정책 | 매직링크 15분 · ParentSession 7일 · 자녀 5명 상한 · BCC 금지 · 3중 격리 · Revoke 60s · Pro 월 09:00 KST 주간 이메일 · 90일 익명화 |

**파킹 추가**: 교사 자유 메시지(v2), 에스컬레이션 경로(v2), 사칭 감지 SOP 고도화(P2).

상세 설계: `plans/parent-viewer-roadmap.md` (in-place v2 갱신, §1.5 승인 게이트 흐름 + §1.6 교사 UI + §3.1 v2 시퀀스 + §4 미들웨어 2종 + §7 PV-1~PV-16 + §11 변경 로그 2026-04-13 행).

---

## 🎯 Seed 8: Canva Publisher 수신 엔드포인트 & PAT (2026-04-12 추가)

implementation-roadmap의 **P0-② Content Publisher Intent** 중 **서버(padlet) 측** 수신 인프라 구현 전담. Canva 앱 클라이언트(`aura-canva-app/src/intents/content_publisher/index.tsx`)는 이미 `{boardId,title,imageDataUrl,sectionId?}` → `200 {id,url}` 계약을 고정한 상태.

**핵심 결정 요약** (`phase3/decisions.md` D1~D16 16건)

| 축 | 결정 |
|---|---|
| 신규 엔티티 | **없음** — 기존 `ExternalAccessToken` 엔티티에 필드 추가만 (tokenPrefix·label·lastUsedAt·scopeBoardIds) |
| PAT 포맷 | `aurapat_{8-char base62 id}_{40-char base62 secret}` — secret scanner 호환. prefix DB 저장, secret은 `SHA-256(secret‖PEPPER)` 후 폐기 |
| 1회 노출 UX | 발급 모달에서 **Copy + Download(.txt)** 버튼 둘 다. 닫으면 재표시 불가 |
| 유효기간 | 드롭다운 1일/30일/**90일(기본)**/365일/무기한 + "권장: 90일 회전" 주석 |
| Scope (v1) | `cards:write` 단일. `scopes String[]` 유지로 v2 `submissions:write`·`webhooks:receive` 여지 |
| Tier 게이팅 | **cards:write는 Pro 전용**. Free 토큰 수신 시 **402 Payment Required** + upgrade link. 발급 시점 + 수신 시점 이중 tier 재검증 |
| Rate Limit | **3축** Upstash sliding window: per-token 60/min · per-teacher 300/hour · per-IP 300/min. 429 + Retry-After. Upstash 장애 fail-open |
| Request body | Zod **strict** 4필드 — `{boardId: cuid, title: 1–200, imageDataUrl: data:image/png;base64, sectionId?: cuid \| null}`. 알 수 없는 필드 → 422 invalid_data_url |
| boardId vs scopeBoardIds | body.boardId **필수**. 서버가 `scopeBoardIds` allowlist 재검증 (빈 배열 = 교사 전체 허용). 위반 → 403 forbidden_board |
| Response | 200 OK **`{id, url}`만** — imageUrl·내부 필드·전체 card 객체 응답 비포함. 에러는 `{error:{code,message}}` 통일 |
| 업로드 한도 | **4.0MB 하드 가드** (Content-Length) → body parse 전 413. Vercel 4.5MB 천장 대비 여유 |
| 저장소 | **Vercel Blob** (Seed 2 인프라 일관). S3는 Enterprise v2+ |
| 스트리밍 | base64 decode stream → `@vercel/blob` multipart put. 메모리 버퍼 X. p95 < 2000ms |
| 레거시 마이그 | **3-stage**: (1) `tokenPrefix String? @unique` nullable 추가 → (2) 레거시 row `revokedAt=now()` + 교사 이메일 재발급 공지(7일 유예) → (3) NOT NULL 전환 |
| sectionId | v1 API body에 optional 수용. 누락 시 보드 기본 섹션(freeform null). Canva 앱 section 드롭다운은 **v2 파킹** |
| CRC32 checksum | **v1 미포함**. v1.1 검토 (신포맷 `aurapatc_`) |
| Canva 딥링크 | v1은 `https://www.canva.com/apps` 일반 URL 폴백. 교사용 deeplink 조사는 **별도 task** 파킹 |
| PAT 분실 | **복구 불가**. 재발급만. GitHub/GitLab 표준 |
| 교사 UI | `/(teacher)/settings/external-tokens` — 갤탭 S6 Lite 최적화, 목록 + FAB + 발급 모달 + 1회 공개 모달. Free는 `cards:write` 잠금 배지 |
| Card 기본값 | width=240, height=160, content="", authorId=tokenOwner.id, sectionId=body.sectionId ?? null, imageUrl=blobUrl(내부) |

상세 설계: `plans/canva-publisher-receiver-roadmap.md` (작업 CR-1~CR-10).

---

## 🎯 Seed 9: 공문 붙임파일 자동 생성 CLI — gongmun-assistant v1 (2026-04-12 추가)

**프로젝트 경계**: Aura-board와 **완전 독립**한 외부 신규 프로젝트. `ideation/` 산출물은 설계 문서만이며, 실제 코드는 `gongmun-assistant/` 저장소에 위치. DB·웹 서버·계정 없음. 교사 개인 `pipx install`로 동작하는 **단일 런타임 CLI**.

**핵심 결정 요약** (`phase3/decisions.md` D1~D8)

| 축 | 결정 |
|---|---|
| 실행 모드 | **v1 = CLI 고정** (typer). `gongmun.bat` Windows 더블클릭 래퍼. 인자 없이 실행 시 대화형(inquirer). Desktop GUI v2·Web v3+ |
| 입력 경로 | **이중 경로 → Stage 1 단일 수렴**. 주 경로 `--main {hwpx}` · 보조 `--text-file`/stdin · `--prompt`는 **가정통신문 한정** |
| 템플릿 관리 | **하이브리드** — 시스템 번들(6개 = 3 type × 2 school_level) 필수 + 사용자 `~/.gongmun/templates/user/{type}/{name}.hwpx` + YAML 사이드카 디렉토리 규약. CRUD 명령 v2 |
| 슬롯 규약 | `{{snake_case}}` 토큰 + `*` 접두 필수 슬롯. 사이드카 YAML에 `{type, school_level, slots, schema}` |
| LLM provider | **Claude API 기본 (Sonnet, Haiku 폴백) + Ollama 옵션** (qwen2.5:14b-instruct-q4). `~/.gongmun/config.yaml` `llm.provider: anthropic\|ollama` |
| 개인정보 경계 | **3중 방어**: (1) `privacy.py` 패턴 스캐너(주민번호·전화·이메일·학번) → exit 5 (2) 명단 CSV 로컬 결정론, LLM 호출 0회(`test_privacy_boundary.py` mock 단언) (3) LLM 로그 기본 off, `--debug` 시 동일 마스킹 |
| 명단 CSV 자동 삭제 | **하지 않음** — 교사 재실행 업무 패턴. `.tmp/` 중간 산출물만 성공 시 정리 |
| config 저장소 | `~/.gongmun/config.yaml` — school·officer·approval_chain·llm. 결재자 실명은 **로컬 치환 기본**, opt-in `--share-officer-to-llm` |
| 배포 | **개인 설치 (pipx install)**. PyPI. Python >= 3.11. 학교 공용 서버 v3+ (NC 라이선스 블로커 선행 해결) |
| v1 범위 | P0 3종 (명단표·동의서·가정통신문) × 2 학교급(elementary/middle-high) = 시스템 템플릿 6개 |
| v2+ 파킹 | P1 3종(평가표·품의서 내역서·참석자 명부) · P2 2종(개별 결과통지서·보고서/회의록) · Desktop GUI · 학교 서버 · 템플릿 CRUD · 슬롯 자동 탐지 · 한컴어시스턴트 연동 |
| 기반 스킬 | **hwpx-master** (`gongmun-assistant/skills/hwpx-master.SKILL.md`) — `<linesegarray>` 제거 · mimetype ZIP_STORED · section0.xml 한정 · lxml API 전용 |
| 종료 코드 | 0=성공 / 2=hwpx 검증 실패 / 3=LLM 실패 / 4=템플릿 없음 / 5=개인정보 패턴 감지 |
| 리스크 대응 | R1 python-hwpx NC → `hwpx_io/` 격리로 교체 지점 1개 · R2 linesegarray → Stage 7 자동 제거 + Stage 8 검증 |

**8단계 파이프라인**: Stage 1 입력 수렴 → Stage 2 맥락 분석(LLM) → Stage 3 유형 결정(auto 시 LLM) → Stage 4 템플릿 resolve → Stage 5 표 채움(**결정론·LLM 금지**) → Stage 6 서술 슬롯(LLM) → Stage 7 linesegarray 제거·원자적 저장 → Stage 8 검증.

상세 설계: `plans/gongmun-assistant-v1-roadmap.md` (작업 GM-1~GM-10).

---

## 🎯 Seed 10: 코지 P2E 목장 게임 — mallang-ranch-p2e (2026-04-13 추가)

**프로젝트 경계**: Aura-board와 **완전 독립**한 외부 신규 프로젝트. ideation 산출물은 설계 문서만이며, 실제 코드는 외부 저장소 `mallang-ranch/`에 위치. 1인 개발자가 9개월 안에 Base L2 위에서 Season 0 MVP 출시. 코드네임 `mallang-ranch-p2e`, 외부 마케팅 브랜드는 `{TBD_BRAND_NAME}` (사용자 추후 결정).

**핵심 결정 요약** (`phase3/decisions.md` D1~D6 + U1~U3 + N1~N7, supplemental_overlay 병합)

| 축 | 결정 |
|---|---|
| 체인·런타임 | **Base L2** (Coinbase) + Coinbase Smart Wallet. Immutable Passport는 심사 통과 시 2순위 조건부 |
| 클라이언트·아트 | **Godot** (PWA HTML5) + **Blender** (1인 직접 모델링, 외주 $0). iOS 네이티브는 v2 파킹 |
| 베이스 동물 | **5종**: cow / chicken / sheep / rabbit / alpaca |
| 희귀도 | **2단계** (common / rare). epic+ 는 **교배 emergent로만** (overlay U1 피벗) |
| 랜치 크기 | **3종**: trial 6×6 / small 10×10 / large 16×16 |
| **교배 시스템 (MVP)** | 2 부모 → 자손. trait DNA = **off-chain server**, 결과 NFT = **on-chain** (Option B Axie-style) |
| Breeding cooldown | 72h base + 24h/breed. **lifetime 7 cap / 동물** (CryptoKitties siring cap) |
| Breeding cost | 100 $FEED base + 부모 희귀도 합산 가중 |
| Hybrid 패턴 | **사전 정의 30~50종**. 첫 발견자 NFT trophy(commemorative, non-tradeable) + $FEED 보너스 |
| 토큰 구조 | NFT (Animal·Land·Item) + **$FEED 비전송 인게임 포인트** + **$RANCH ERC-20** (TGE 트리거 후) |
| TGE 트리거 | DAU 50K **AND** 월 거래량 $200K **AND 둘 다 2개월 연속** 충족 → Season 2 시작점 (overlay) |
| Land 1차 분양 | 캡 5,000 (Small 4,000 + Large 1,000). WL 48h 우선 → 3주 갭 → 퍼블릭 고정가 (Dutch 옥션 배제) |
| Land 가격 | Small 0.03 ETH public / 0.02 ETH WL · Large 0.15 ETH public / 0.10 ETH WL (ETH $3,000 가정 → 매출 천장 **$675K**) |
| 컨트랙트 | **6개** (overlay): AnimalNFT / LandNFT / ItemNFT / RanchToken / FeedClaim / **Marketplace 자체 deploy** (EIP-712 + Seaport 위임형, 로열티 enforce) |
| 감사 예산 | $40K 상한 ($30K 1차 + $10K 재감사 예비) |
| 1차 세일 분배 | **팀 70% / 트레저리 30%**. 팀 $472K = audit 40 + art_outsource 0 + initial_ops 130 + sustained_ops 192 + reserve 50 + breeding_dev 60 |
| 2차 로열티 | **5%, 50/50** (지속 운영 재원 — 트레저리 지분 확대) |
| 거버넌스 | Season 0: Gnosis Safe **2-of-3** 멀티시그. Season 2 TGE 후: Snapshot + Timelock 7일 |
| **No team reserve** | Sunflower 철학 — 팀 토큰 적립금 0% |
| 지역 | 글로벌 170개국, 한국 IP 차단 + 한국 KYC 페이아웃 차단 |
| 언어 (런치) | **EN / JA / VI / TH** (Korean **미제공**, 위메이드 이미르 전례 — overlay U2) |
| 법인 | Season 0: 개인사업자/싱가포르 BVI Ltd. **Cayman Foundation은 TGE 6개월 전 설립** (N1) |
| Proof of Humanity | **Gitcoin Passport** (Web3 native, 무료, Base 연동 — N4) |
| 오픈소스 | 코드 전체 공개 |
| MVP 기간 | **9개월** (M0–3 컨트랙트+감사, M3–6 게임 루프+아트, M6–9 closed alpha+버그픽스+세일 준비) |

상세 설계: `plans/mallang-ranch-p2e-roadmap.md` (10개 섹션). 시드 데이터 skeleton: `data/mallang-ranch-seed.json`.

---

## 🎯 Seed 11: 과제 배부 보드 — assignment-board (2026-04-14 추가)

**핵심 결정 요약** (`phase3/decisions.md` Q1~Q7, ambiguity 0.083)

| 축 | 결정 |
|---|---|
| Board 확장 | `Board.layout="assignment"` 확장 + 3 필드 (`assignmentDueAt` · `assignmentAllowLate` · `assignmentGuideText`). 신규 BoardType 없음 |
| 신규 엔티티 | **1종** `AssignmentSlot` — `boardId·studentId·slotNumber·cardId?·submissionId?·submissionStatus·gradingStatus·returnReason(≤200자)` + `@@unique([boardId,studentId])` + `@@unique([boardId,slotNumber])` |
| 재사용 | `Classroom`·`Student`·`Card`·`Submission`·`BoardMember`·`Section`(무수정) |
| 교사 가이드 (Q1) | **`Board.assignmentGuideText` 단일 텍스트**. Section(role="guide") + Card 재사용은 v2 파킹 |
| 재제출 정책 (Q2) | 상태 조건부 매트릭스. `not_graded`+마감 전 = in-place 덮어쓰기, 마감 후 = `allowLate` 플래그, `graded/released` = 차단, `returned` = 허용(`gradingStatus` 리셋). `SubmissionHistory`는 v2 파킹 |
| 학급 N 상한 (Q3) | **v1 N ≤ 30 하드 제한**. N>30은 생성 차단 + 분반 안내. 5×8 확장 v2 파킹 |
| 미제출 독려 (Q4) | **v1 인앱 배지 전용**. 학부모 이메일 경유 v1 제외 (parent-viewer §1.3 주간 이메일과 분리). 쿨다운은 v2 |
| 학생 간 열람 (Q5) | **v1 비공개 고정**. `Board.galleryMode` · `galleryReleasedAt`은 v2 스키마 예약, v1 미포함 |
| slotNumber 동기화 (Q6) | **생성 시점 스냅샷**. Student.number 변경해도 slot 좌표 불변. 삭제된 학생은 `submissionStatus="orphaned"` soft 마킹 |
| 반려 UI (Q7) | **전체화면 모달 전용** (격자 롱탭·컨텍스트 메뉴 없음) + **반려 사유 1줄(≤200자) 필수**. 학생 모달 재진입 시 상단 고정 배너. 반려 카드에 격자 `"!"` 배지 |
| 썸네일 | 서버 리사이즈 160×120 WebP + `loading="lazy"` + IntersectionObserver (T0-④ 승계) |
| 카드 모달 | 전체화면 모달 전용 (사이드패널 없음 — 탭 S6 Lite 세로폭 제약) |
| matrix 뷰 | v1 제외. v2에서 owner + 데스크톱 전용 별도 라우트 |
| Canva 시너지 | v1은 선택. 제목 규칙 `완료-{Board.title}-{Student.number}-{Student.name}` + 기존 `canva-assignment-pdf-merge` 스킬로 병합 |

상세 설계: `plans/assignment-board-roadmap.md` (작업 AB-1~AB-10).

---

## 🎯 Seed 12: 수행평가 자동채점 파이프라인 — assessment-autograde (2026-04-16 추가)

**핵심 결정 요약** (`phase3/decisions.md` §1·§6·§7 + `phase4/seed.yaml`, ambiguity 0.10)

| 축 | 결정 |
|---|---|
| Board 확장 | `Board.layout="assessment"` + AssessmentTemplate이 셸에 연결. 별도 BoardType 신설 없음 |
| 신규 엔티티 | **7종**: AssessmentTemplate · AssessmentQuestion · AssessmentSubmission · AssessmentAnswer · GradebookEntry · ProctorEvent · FeatureFlag. Classroom은 `gradebookReleasePolicy`·`schoolManagedDevices` 필드 확장 |
| v1 문항 스코프 (U5) | **MCQ + SHORT 2종만**. OX·NUMERIC은 v1.5, ESSAY는 v2. Question create API Zod gate로 강제 |
| MCQ 채점 (U2) | `correctChoiceIds` ↔ `selectedChoiceIds` 서버 **결정론 매칭**. LLM 호출 금지 가드 |
| SHORT 채점 (U2) | **Gemini 2.5 Flash** 전용 (프로바이더 추상화 유지). 4필드 폼(모범답안·키워드·부분점수·루브릭). KR-SBERT 코사인 < 0.25 또는 10자 미만 자동 0점(LLM 스킵). `partialCredit=false` 시 이진 처리 |
| 인프라 (D1) | **공통 Supabase 단일 프로젝트 공유** (Aura-board ↔ Aura 웹앱) + RLS 3분화 + Realtime p95 < 500ms + PGMQ `grading_retry` (exponential backoff 2·4·8·16·32s, 5회 실패 → `retry_exhausted` 배지) |
| 부정행위 (U3) | **알림 + 자동 화면이탈 잠금 하이브리드**. Page Visibility/fullscreenchange/focus → 답안 영역 disable + `isLocked=true, lockedReason` Supabase 영속 + ProctorEvent + Realtime 교사 배지. **자동 제출·자동 처벌·자동 상태전이 없음**. 해제 2경로(학생 "돌아왔습니다" / 교사 "재개 승인"). **시간 계속 흐름** (endAt = startedAt + durationMin 고정) |
| isLocked 저장 | **Supabase 영속화** (클라이언트 state만이면 새로고침 우회 → 잠금 실효 0) |
| 성적 릴리스 (U4) | `Classroom.gradebookReleasePolicy="teacher_manual"` 고정. `GradebookEntry.releasedAt IS NULL`이면 학생·학부모 비노출 |
| Tier (U1) | Pro 전용 + `FeatureFlag.assessmentTierGate` 런칭 플래그. 개발·베타는 `false`(전 사용자 공개), 런칭 시 `true`로 Pro 게이트 enforce. 발급+수신 이중 tier 재검증 |
| UI 위치 (Q5) | **2곳 운영**: Aura 웹앱 `/aura-web/gradebook` (학생·학부모) + Aura-board `/teacher/assessments/[id]/gradebook` 교사 매트릭스 (**owner + 데스크톱 전용**, 학생 × 문항 > 30×30 시 react-virtual) |
| 매트릭스 스코프 | editor(학생)·viewer(학부모)·태블릿 전부 제외 |
| 태블릿 응시 예산 | iframe **0**, 문항 ±1 lazy, OCR **클라우드 오프로드 전용** (Tesseract.js 금지), S-Pen 캔버스 **800×400 고정 px** 60fps, IndexedDB + Supabase autosave 300ms debounce, 문항별 background flush |
| AI 제안 노출 | 교사 확정 전 autoScore는 학생·학부모 **완전 비공개**. `autoRawResponse` 감사용 보관 |
| 동의서 | 학기 초 학부모 일괄 동의(LLM 국외 이전·손글씨·성적 열람 3건). 미동의 학생 **손글씨 비활성(키보드만)** |
| 감사 로그 | Free=1학기, Pro=학년 + CSV/PDF export. 이후 Supabase cron 자동 파기 |
| 재응시 락 | `status="submitted"` 이후 PATCH 거부. 재시험은 별도 템플릿·별도 레코드 |
| Service Worker | 응시 중 외부 도메인 fetch 화이트리스트(same-origin + Supabase + Storage). 이탈 시 `window_open_blocked` 유사 로그 |
| Knox Kiosk L4 | v1은 UI 플래그 + 가이드 문서만. 실 MDM 연동 0 (BYOD 기본) |
| 타임라인 | **8~10주** (스코프 축소·잠금 API 포함) |

**새로 드러난 분기** (현재 세션 편입 금지):
- **3.1** 학부모 열람 경로 — parent-viewer v2 "성적 탭 편입" vs 별도 route 신설. 결정권: **v1.5 integrate 시점**. 현 v1은 "Aura 웹앱 내부·동일 RLS·동일 PWA 셸"로만 합의
- **3.2** 쌤기부-style 생기부 문장 AI 생성 — 별 task
- **3.3** 학생 탭 카메라 프리뷰 grid — 프라이버시 검토 필요, 기록만
- **3.4** Pro tier 학급 인원 과금 모델 — Team/School tier 신설 별 task 후보
- **3.5** `/aura-web/gradebook` UX 패턴 (하이클래스 타임라인 vs Classroom 표) — phase6 handoff 시 aura INBOX로 동반 송출

상세 설계: `plans/assessment-autograde-roadmap.md` (작업 AA-1~AA-10).

---

## 🔗 인접 시드 간 의존성

```
Seed 1 (그림보드/라이브러리) ─── 개별 애셋 비공개 토글 ─┐
Seed 4 (식물관찰일지)      ─── 관찰 isPrivate 토글 ──┤ → 공유 정책 일관 (반 공개 기본)
Seed 6 (Breakout)         ─── isPublic=true 기본 ───┘
                                                  │
Seed 2 (Tier) ─── 고비용/저비용 분류 ───────────────┼─→ Seed 5 (Canva API) 한도 구조 공유
              ├── 반 수·스토리지 쿼터 ──────────────┤
              └── Free 3종/Pro 5종·커스텀 한도 ──→ Seed 6 (Breakout Template gating)
                                                  │
Seed 3 (행사 보드) ─── accessToken·owner·collaborator ─→ Seed 5 (Publisher Intent) 프라이빗 앱 사용 주체
                                                  │
Seed 6 (Breakout) ─── Section 재활용 + accessToken ──→ T0-① 섹션 격리 뷰 (구현 전제)
Seed 6 (Breakout) ─── 템플릿 복사/역전파 없음 ──────→ Seed 1 (AssetAttachment 복사 패턴 준거)
                                                  │
Seed 7 (Parent Viewer) ─ 자녀 범위 매트릭스 ─────────→ Seed 1·3·4·6 (학부모 열람 single source of truth)
Seed 7 (Parent Viewer) ─ 스마트폰 PWA 성능 ─────────→ T0-④ 이미지 파이프라인 (presigned 썸네일)
Seed 7 (Parent Viewer) ─ BoardMember.role+="parent" ─→ 기존 RBAC 승계
Seed 7 (Parent Viewer) ─ tier 게이팅 (Free 2/Pro 5) ─→ Seed 2 Tier 매트릭스 승계
                                                  │
Seed 8 (Canva Publisher 수신) ─ cards:write Pro 전용 ─→ Seed 2 Tier 매트릭스 (Free 402)
Seed 8 (Canva Publisher 수신) ─ 서버 수신 계약 ──────→ Seed 5 P0-② (프라이빗 교사 앱)·implementation-roadmap
Seed 8 (Canva Publisher 수신) ─ scopes String[] 확장점 ─→ (v2) Webhook receive · Slack/Miro metadata JSON
Seed 8 (Canva Publisher 수신) ─ 교사 UI 갤탭 S6 Lite ─→ tablet-performance-roadmap §2 성능 예산
                                                  │
Seed 11 (assignment-board) ─ Board.layout="assignment" ──→ Seed 3·6과 동일한 layout 확장 패턴 (BoardType 신설 금지 관례)
Seed 11 (assignment-board) ─ Submission 재사용 (status 네임스페이스 분리) ─→ Seed 3 (event-signup Submission.status와 충돌 없음, slot 레벨 필드로 분리)
Seed 11 (assignment-board) ─ 자녀 범위 매트릭스 행 추가 ─────→ Seed 7-v2 (parent-viewer-roadmap PV-7 서버 필터에서 처리)
Seed 11 (assignment-board) ─ 30-카드 5×6 정형 격자 성능 예산 ─→ tablet-performance-roadmap §2a (신규)
Seed 11 (assignment-board) ─ Canva 제목 규칙 + PDF 병합 ─────→ implementation-roadmap P0-② + canva-assignment-pdf-merge 스킬 (v1은 선택)
                                                  │
Seed 12 (assessment-autograde) ─ Board.layout="assessment" ─→ Seed 3·6·11 동일 layout 확장 패턴
Seed 12 (assessment-autograde) ─ 채점·성적 송신 레이어 분리 ─→ Seed 11 (assignment-board) 상위 확장 (assignment는 제출·반려·Roster, assessment는 채점·GradebookEntry·릴리스)
Seed 12 (assessment-autograde) ─ 공통 Supabase + RLS 3분화 + Realtime ─→ Seed 7-v2 (parent-viewer v2 전제 승계), canva-publisher-receiver는 비교 레퍼런스만(외부 PAT 경로 사용 안 함)
Seed 12 (assessment-autograde) ─ 성적 릴리스 `releasedAt IS NOT NULL` ─→ Seed 7-v2 §5 매트릭스 이중 조건(parent active + releasedAt) 추가, 탭 편입 v1.5 미결
Seed 12 (assessment-autograde) ─ Pro 전용 + FeatureFlag.assessmentTierGate ─→ Seed 2 Tier 매트릭스 런칭 플래그 패턴
Seed 12 (assessment-autograde) ─ 응시 화면 iframe 0 + OCR 클라우드 + S-Pen 800×400 ─→ tablet-performance-roadmap §2b (신규)
Seed 12 (assessment-autograde) ─ 매트릭스 owner+데스크톱 전용 ─→ MEMORY matrix_desktop_only 방침 (Seed 11과 동일)
                                                  │
모든 Seed ─── 갤럭시 탭 S6 Lite + iframe LRU 3개 ─── 공통 성능 예산 (단, Seed 7은 스마트폰 예산 별도: TTI<2s LTE / 첫 뷰포트<500KB / iframe 금지; Seed 12 응시 화면은 iframe 0)

────────────────────────────────────────────────────────────────
Seed 9 (gongmun-assistant v1)  ★ 완전 독립 — Aura-board 외부 프로젝트 ★
  · ideation/ 와 코드·DB·인증·배포 채널 전부 분리
  · Seed 1~8 어느 것과도 의존성 화살표 없음
  · destination: gongmun-assistant/ (ideation 외부 신규 저장소)
  · 기반 스킬: hwpx-master (gongmun-assistant/skills/hwpx-master.SKILL.md)
  · 공통 성능 예산·Aura-board Tier·RBAC·RLS 정책 모두 미적용
────────────────────────────────────────────────────────────────

────────────────────────────────────────────────────────────────
Seed 10 (mallang-ranch-p2e)  ★ 완전 독립 — 외부 신규 P2E 프로젝트 ★
  · ideation/ 와 코드·체인·게임 클라이언트·세일 인프라 전부 분리
  · Seed 1~9 어느 것과도 의존성 화살표 없음
  · destination: mallang-ranch/ (ideation 외부 신규 저장소, INBOX 미생성 — phase6 handoff에서 등록)
  · 기반 체인: Base L2 + Coinbase Smart Wallet (Immutable Passport 조건부 2순위)
  · 게임 엔진: Godot 클라이언트 (PWA HTML5) + Blender 모델링 (1인 직접)
  · 공통 성능 예산·Aura-board Tier·RBAC·RLS 정책 모두 미적용
  · 코드네임 사용, 외부 마케팅 브랜드 {TBD_BRAND_NAME} (사용자 추후 결정)
────────────────────────────────────────────────────────────────
```

---

## 📍 다음 단계 옵션

- **`ooo run <seed_id>`** — 특정 seed를 TUI로 실행 (실제 padlet 코드에 구현 착수)
- **`ooo status`** — 세션 상태·drift 추적
- **`ooo evaluate`** — 구현 후 3단계 검증
- 또는 추가 정교화 필요한 소소한 미결을 더 굴리기

## 🔍 확인 부탁

이번 4개 seed 모두 **자율 판단**으로 답변했습니다. 특히 가격(₩9,900/월) 같은 핵심 숫자는 추정치이니 조정 의사 있으면 알려주세요:
- Pro 가격 조정 필요?
- 고비용/저비용 API 분류 재정의?
- Content Publisher v1 교사 주체 결정 OK?
- 반 공개 기본값 OK? (그림보드·식물 둘 다 적용)

문제 있으면 해당 주제 재인터뷰 돌리겠습니다.
