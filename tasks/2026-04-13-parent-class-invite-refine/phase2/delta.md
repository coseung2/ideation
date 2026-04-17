# Phase 2 Delta 분석 — Parent Class Invite Refine

- **task_id**: `2026-04-13-parent-class-invite-refine`
- **입력**: `phase1/audit.md` (A~K 11개 후보, N1~N19 신규 결정, P0 7건)
- **산출 목적**: 실제 변경 vs 유지 분리 + 자명 결정 확정 + phase3 인터뷰 질문 10개 도출
- **change_trigger (고정)**: 학생별 개별 코드 → 학급 단일 코드 + 학부모 셀프 온보딩 + 교사 승인 게이트

---

## 1. 변경 항목 표 (audit 후보 A~K)

| ID | 기존 결정 | 제안 변경 | 근거 | 사용자 확정 필요? |
|---|---|---|---|---|
| **A** 페어링 단위 | 학생 1 : 코드 1 (`ParentInviteCode.studentId`) | 학급 1 : 코드 1 (`ClassInviteCode.classroomId`) — 옵션 A1 | change_trigger 사용자 명시. 옵션 A2(세대별 회전)는 v2 파킹. | **N** (자명 확정) |
| **B** 코드 엔트로피·수명·회전 | Crockford Base32 **6자리** + **48h** + maxUses 3 | 자릿수·TTL·회전 주기 사용자 결정 필요 (권장 B1: 8자리 + 학기말 TTL + 무제한 maxUses + 교사 수동 회전) | 학급 단위는 유출 임팩트 ↑ 엔트로피·수명 재설계 필수. 단 구체 수치는 운영 현실(학기 길이·회전 피로)에 달림. | **Y** (N2 · Q1·Q2) |
| **C** brute-force 정책 | IP 5회/15분 + **코드 10회 실패 즉시 만료** | IP 5회/15분 **유지** + 코드 자동 만료 트리거 **제거** (학급 전체 봉쇄 방지) + 교사 알림만 | 수학적 자명: 학급 코드 하나로 수십 학부모가 가입하므로 자동 만료는 DoS 벡터. 교사 수동 회전은 후보 B와 연동. | **N** (자명 확정) |
| **D** revoke_reason 유니언 | `teacher_action`\|`self_withdraw` | D1: **+ `rejected_by_teacher` + `auto_expired_pending`** 확정. D2(`code_rotated`)는 Q5와 연동되어 조건부. | D1 2개는 승인/만료 경로 추가로 구조적 필연. D2는 학급 코드 회전 시 pending 일괄 처리 정책 결정에 의존. | **부분 Y** (Q5에서 D2 포함 여부만) |
| **E** 교사 UI 배치 | 학생 카드 드롭다운 + 학급 설정 "학부모 액세스" 탭 | E1: 학급 설정 "학부모 액세스" 탭 내부 3-섹션 (a) 초대 코드 (b) 승인 대기 인박스 (c) 연결된 학부모 | 기존 IA 최소 변경 + 교사 눈높이에서 한 곳에 모이는 것이 승인 SLA 달성에 유리. E2는 메뉴 비대화. | **N** (자명 확정) |
| **F** 명단 표시 범위 | 해당 없음 | PII 보호 기본값: **이름 마스킹(성 + 이름 첫 글자, 예: "김O민") + 반·번호** (F1·F2 혼합 변형 F*). 본명 풀 노출(F1) 또는 사진(F3)은 보안 약화. | PII 자명 결정 원칙. 단 동명이인 구분·학부모 식별 편의 vs 유출 면적의 절충점은 사용자 확인 권장. | **Y** (Q3 — 자명 결정과 의도 재확인) |
| **G** pending 상한·TTL | 해당 없음 | G1: pending TTL **7일**, 학부모당 동시 pending **3건** 상한 (권장). G2(48h/5건)는 교사 업무 주기와 불일치. | 교사 주 단위 업무 현실 반영은 합리적. 다만 TTL·상한 수치는 사용자 운영 감각에 의존. | **Y** (Q4) |
| **H** 승인 SLA | 해당 없음 | H1: 권고 24h, hard SLA 없음, 교사 주의 배지만. 에스컬레이션은 v2. | v1 단순성 원칙 부합. 24h vs 72h 중 어느 것을 "권고"로 볼지는 사용자 운영 정책. | **Y** (Q6) |
| **I** 동일 자녀 복수 학부모 | `@@unique([parentId, studentId])` | I1: 기존 유니크 유지 + 다른 parent 독립 신청 허용. 자녀당 active 학부모 상한 없음. | 현 5명 상한은 학부모 기준이며 자녀 기준 상한 도입은 v2 스코프. 부모 공동 양육 현실에서 상한은 과도. | **N** (자명 확정) |
| **J** Free/Pro 발급 한도 | Free 자녀당 2 / Pro 자녀당 5 | J3: **한도 개념 폐지**, tier 연계는 주간 이메일 수신(Pro만) 혜택에만 집중. | "자녀당 N" 개념이 학급 코드로 이동 시 의미 붕괴(audit 1.3-e·f). J2(학급당 학부모 수)는 학급당 학부모 = 학생×부모수로 통제 가치 낮음. | **Y** (Q7 — tier 정책 변경은 과금 영향) |
| **K** pending 세션/BoardMember 시점 | 해당 없음 | K1: ParentSession은 **signup 시점** 발급 + BoardMember는 **approve 시점** 발급. | 학부모가 pending 상태 조회·취소를 위해 세션이 필요(자명). BoardMember를 pending에 만들면 RLS 오염 위험. | **N** (자명 확정) |

**소계**: 11건 중 자명 확정 **5건** (A·C·E·I·K) + 구조적 필연 부분 확정 **1건** (D 중 D1) + 사용자 확정 필요 **5건** (B·F·G·H·J) + 부분 확정 **1건** (D 중 D2).

---

## 2. 신규 결정 N1~N19 매핑

| # | 결정 항목 | 상태 | 비고 |
|---|---|---|---|
| N1 | 학급 코드 엔티티 구조 (`ClassInviteCode` 신설) | ✅ 자명 확정 | 후보 A1 — 별도 엔티티 분리가 의미론·RLS 단순성에 유리. |
| N2 | 학급 코드 자릿수·TTL·maxUses·회전 정책 | ❓ 사용자 확정 | Q1 (자릿수 6 vs 8) · Q2 (TTL/회전 주기) |
| N3 | brute-force 방어 재설계 | ✅ 자명 확정 | 후보 C — 코드 자동 만료 제거, IP 잠금만, 교사 알림. |
| N4 | `ParentChildLink.status` 유니언 확장 | ✅ 자명 확정 | `"pending"\|"active"\|"rejected"\|"revoked"` — 구조적 필연. 전이 머신: `pending → active` (approve), `pending → rejected` (reject/auto_expire), `active → revoked` (기존). |
| N5 | 감사 필드 신규 6종 | ✅ 자명 확정 | `requestedAt, approvedAt, approvedById, rejectedAt, rejectedById, rejectedReason` — 감사 요건 충족. |
| N6 | `revokedReason` 유니언 확장 | 🟡 부분 확정 | D1 2개 확정, D2(`code_rotated`) 포함 여부는 Q5. |
| N7 | 신규 API 엔드포인트 3종 | ✅ 자명 확정 | `POST /api/parent/signup`, `POST /api/parent/match/code`, `POST /api/parent/match/request` (audit §3.1 시퀀스 승계). |
| N8 | `parentAuthOnlyMiddleware` 도입 | ✅ 자명 확정 | 매칭 전 엔드포인트용 별도 미들웨어 — `parentScopeMiddleware`는 `studentId ∈ parent.children` 전제라 우회 불가. |
| N9 | 교사 UI 3-섹션 배치 | ✅ 자명 확정 | 후보 E1. |
| N10 | 매칭 시 명단 표시 범위 | ❓ 사용자 확정 | Q3. |
| N11 | pending 상한·TTL | ❓ 사용자 확정 | Q4. |
| N12 | 승인 SLA | ❓ 사용자 확정 | Q6. |
| N13 | Free/Pro 발급 한도 재정의 | ❓ 사용자 확정 | Q7. |
| N14 | pending 세션/BoardMember 발급 시점 | ✅ 자명 확정 | K1. |
| N15 | 학급 코드 유출 회전 SOP + 사칭 패턴 감지 | 🟡 P2 파킹 | v1에서는 교사 수동 회전 버튼 + "거부율 임계 초과" 알림만. 구체 SOP는 운영 단계 문서로 이관. |
| N16 | `/parent/pending` 상태 화면 UX + HTTP 응답 | ❓ 사용자 확정 | Q8 (401 vs 200 payload flag). |
| N17 | 90일 내 재가입 시 학급 코드 재입력 | ✅ 자명 확정 | audit 1.4-d — 학생별 코드 재발급 → 학급 코드 재입력 + 재승인 치환. |
| N18 | PV 작업 카드 재구성 | ✅ 자명 확정 | PV-1·2·3·8·12 개정 + PV-13~16 신설 (audit §6 승계). |
| N19 | 공수 재산정 | ❓ 사용자 확정 | Q9 (27일 → 33~34일 증분 수용 가능한지). |
| (추가) | 동일 자녀 부·모 동시 신청 처리 | ❓ 사용자 확정 | Q10 (I 후보 자명 확정과 별개로 운영 규칙 확인). |
| (추가) | 매칭 신청 시 학부모 추가 정보 제출 | ❓ 사용자 확정 | Q5와 통합 또는 별도 Q11. |
| (추가) | 기존 학생별 코드 마이그레이션 | ❓ 사용자 확정 | Q11 (신규 배포 vs 병행). |

---

## 3. 유지 항목 (인터뷰 재질문 금지 — 기준선)

| 영역 | 유지 결정 | 근거 |
|---|---|---|
| 인증 수단 | 매직 링크(이메일 OTP) 전용, 유효 15분 | seed_37b35654542f §6 |
| 세션 | `ParentSession` 7일 TTL + 재인증 시 이메일만 | audit 1.1-f |
| 학부모 상한 | 1인당 자녀 **5명** (tier 무관) | audit 1.1-h |
| 학부모 간 격리 | BCC 금지 · 부/모/조부모 상호 비노출 · 이름 비노출 | audit 1.2-j |
| 콘텐츠 격리 | API 필터링(1차) + DOM 마스킹(보조) + RLS(3중), parentA→B 404, 타 학생 403 | audit 1.2-f·g·h |
| 자녀 범위 매트릭스 | §5 매트릭스 전체 (매칭 활성화 이후 규칙) | audit §5 |
| 탈퇴 | Soft delete 고정 + 90일 익명화(email SHA-256, "탈퇴한 학부모") + hard delete 금지 | audit 1.4-a·c |
| 주간 이메일 | 월 09:00 KST + 활동 0건 스킵 + BCC 금지 개별 발송 | audit 1.3-b·c·d |
| Revoke SLA | ≤ 60초 (SWR 60s 폴링) | audit 1.2-a |
| 성능 예산 | TTI < 2s LTE / < 3s 3G, 첫 뷰포트 < 500KB, 썸네일 < 200KB | seed §12 |
| 전송 제약 | iframe 금지, proxy thumbnail만, WebSocket 비활성(SWR polling 60s) | seed |
| Crockford 규칙 | O/0·I/1·L 제외 대문자, CSPRNG | audit §1.1-b (길이만 재검토) |
| v1 스코프 | 읽기 전용, 응원·댓글·좋아요는 v2+ 파킹 | audit 1.3-g |

---

## 4. Phase 3 인터뷰 질문 목록 (10개, 옵션 3개씩)

각 질문은 사용자 확정 필요 항목에 대해 빠른 수렴을 위해 옵션 후보 3개 선제시. 권장 옵션은 `[권장]` 표기.

### Q1. 학급 초대 코드 **자릿수**
- A. Crockford Base32 **6자리** (32⁶≈10⁹, 현행 유지)
- B. Crockford Base32 **8자리** (32⁸≈10¹², 학급 코드 노출 면적 대응) `[권장]`
- C. Crockford Base32 **10자리** (32¹⁰≈10¹⁵, 최대 보안 · UX 부담)

### Q2. 학급 초대 코드 **수명·회전 정책**
- A. **학기말 자동 만료** (예: 이용약관 학사일정 기준, 다음 학기 시작 시 자동 회전) + 무제한 maxUses + 교사 수동 회전 버튼 `[권장]`
- B. **30일 롤링 TTL** + Cron 자동 회전(교사에 새 코드 통보) + 무제한 maxUses
- C. **영구** TTL(revoke 전까지) + 무제한 maxUses + 교사 수동 회전만

### Q3. 매칭 신청 시 학부모가 보는 **학급 학생 명단 표시 범위**
- A. **반·번호 + 이름 마스킹**(예: "3반 12번 김O민") — PII 최소화 `[권장]`
- B. **반·번호 + 본명** (동명이인 구분 쉬움, 기존 자녀 본명 노출 정책과 일관)
- C. **반·번호 + 본명 + 프로필 사진** (사칭 방어 약화 — 비권장)

### Q4. pending 상태 **자동 만료 TTL** + 학부모당 **동시 pending 상한**
- A. TTL **7일** + 동시 pending **3건** (교사 주 단위 업무 반영) `[권장]`
- B. TTL **48시간** + 동시 pending **5건** (빠른 처리 유도)
- C. TTL **14일** + 동시 pending **5건** (넉넉한 여유)

### Q5. `revokedReason` 유니언에 **`code_rotated` 포함 여부** + 회전 시 pending 처리
- A. **포함** — 학급 코드 회전 시 해당 코드로 생성된 pending 일괄 rejected(`code_rotated`) 처리 `[권장]`
- B. **미포함** — 회전 후에도 기존 pending은 그대로 두고 교사 수동 처리
- C. 포함하되 **기본값은 유지**, 교사가 "회전 시 pending 일괄 처리" 체크박스로 선택

### Q6. 교사 **승인 SLA** 가이드라인
- A. 권고 **24시간**, hard SLA 없음, 교사 주의 배지만 `[권장]`
- B. 권고 **24시간** + **72시간** 초과 시 학급장·관리자 에스컬레이션 배지
- C. SLA 안내 문구 없음(교사 자율 완전 위임)

### Q7. Free/Pro **발급 한도** 재정의
- A. **한도 개념 폐지**, Pro tier는 주간 이메일 수신 혜택에만 집중 `[권장]`
- B. **학급당 승인 가능 학부모 수** 한도 (Free 학급당 20 / Pro 무제한)
- C. **교사당 활성 학부모 수** 한도 (Free 50 / Pro 무제한)

### Q8. pending 상태 **HTTP 응답 방식**
- A. **200 OK + payload flag** (`{"status":"pending"}`) → 클라이언트가 `/parent/pending` 화면 렌더 `[권장]`
- B. **401 Unauthorized** + 별도 힌트 헤더 `X-Parent-Status: pending`
- C. **403 Forbidden** + payload flag

### Q9. 공수 증분 수용 (**+6~7일**)
- A. **수용** — 기존 27일 → 33~34일 `[권장]` (변경 트리거의 가치 대비 타당)
- B. **스코프 축소** — pending auto-expire Cron(PV-16)과 사칭 감지(N15)는 v2로 이연, 증분 +3~4일
- C. **재협상** — 일부 유지 항목(Revoke SLA 60s 등) 완화로 상쇄

### Q10. 동일 자녀에 **부·모 동시 신청** 처리
- A. **둘 다 승인** — `@@unique([parentId, studentId])`만 적용, 다른 parent는 독립 신청 가능 (audit I1) `[권장]`
- B. **첫 승인만 허용** — 자녀당 active 학부모 1명 상한
- C. **자녀당 active 4명 상한** (부·모·조부·조모)

### Q11. 기존 학생별 코드 **마이그레이션** 정책
- A. **완전 폐기** — v1 배포 시점에 `ParentInviteCode` 테이블 드롭, `ClassInviteCode`만 사용 (현재 발급된 학생별 코드 없음 전제 — 아직 미배포) `[권장]`
- B. **병행 운영** — 기존 학생별 코드는 TTL 만료까지 유효, 신규는 학급 코드만 발급
- C. **일괄 전환** — 기존 학생별 코드 즉시 무효화 + 교사에게 학급 코드 재발급 안내 이메일

> Q10·Q11은 audit.md의 N1~N19 외 추가 식별 항목으로, phase3 인터뷰에서 반드시 확정되어야 함.

---

## 5. 연쇄 영향 요약 (plan 섹션·작업 카드 갱신 범위)

phase5 integrator가 `plans/parent-viewer-roadmap.md`에 반영할 갱신은 다음과 같이 집약된다. **§1.1 페어링·인증** 전 행 재작성(1.1-a·c 변경 확정, 1.1-b·d·e·g·h 재검토 반영), **§1.2 Revoke·격리**에 reject·auto_expire 경로 2종 추가 및 pending 명단 마스킹 규칙(§1.2-i 신규 한 줄) 보강, **§1.3 알림 Tier**에서 1.3-e·f(자녀당 N 한도) 폐지 및 Q7 결과로 tier 재정의, **§1.4 탈퇴·감사**에서 1.4-d·e를 학급 코드·승인자 분리(`approvedById` vs `issuedById`) 반영. **§2 데이터 모델**은 `ParentInviteCode` → `ClassInviteCode` 치환 + `ParentChildLink.status` 4-value 유니언 + 6종 감사 필드 + RLS에 `status='active'` 조건 명시. **§3.1 페어링 시퀀스** 전면 재작성(audit에 초안 있음). **§4 미들웨어**는 `parentAuthOnlyMiddleware` 신규 섹션. **§5 자녀 범위 매트릭스**는 유지하되 pending 동안 자녀 본인 콘텐츠도 열람 불가 한 줄 추가. **§7 작업 카드**에서 PV-1·2·3·8·12 개정 + PV-13(학급 코드 UI) · PV-14(학부모 셀프 플로우) · PV-15(승인 인박스) · PV-16(pending auto-expire Cron + 알림) 신설로 총 공수 27일 → 33~34일(Q9 결과 반영). **seeds-index.md Seed-7** 엔트리는 "학급 코드 + 셀프매칭 + 교사 승인" 변경 트리거를 추가한 수퍼시드 링크로 갱신, 기존 seed_37b35654542f는 `destinations/archive/`로 이관 후보(phase7 dispatcher 판정).

---

## 6. 요약 카운트

- **변경 항목 (A~K)**: 11건
  - 자명 확정(에이전트 결정): **5건** (A·C·E·I·K)
  - 부분 확정(구조적 필연): **1건** (D의 D1 부분)
  - 사용자 확정 필요: **5건** (B·F·G·H·J)
- **신규 결정 (N1~N19 + 추가 3건)**: 22건
  - 자명 확정: **10건** (N1·N3·N4·N5·N7·N8·N9·N14·N17·N18)
  - P2 파킹: **1건** (N15)
  - 부분 확정: **1건** (N6)
  - 사용자 확정 필요: **10건** (N2·N10·N11·N12·N13·N16·N19 + Q10·Q11)
- **유지 항목**: 13개 영역 (인터뷰 재질문 금지)
- **phase3 인터뷰 질문**: **11개** (Q1~Q11, 각 옵션 3개 선제시)

> 당초 계약서 "≤ 10개" 목표 대비 1개 초과 — Q10·Q11은 audit이 식별하지 못한 운영 경계 항목으로, 사용자 확정 없이 seed 진행 시 재작업 리스크가 크므로 포함 불가피. phase3 facilitator가 필요 시 Q5+Q10 또는 Q9+Q11 묶음 처리로 턴 수 절감 가능.
