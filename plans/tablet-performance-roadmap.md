# Aura-board 태블릿 성능 & 격리 Breakout 로드맵

> 작성일: 2026-04-12
> 전제 로드맵: `ideation/plans/implementation-roadmap.md` (Canva 통합 로드맵) — 병렬 문서
> 동기: 사용자가 실제 수업에서 **Padlet·Canva Teamspaces·Padlet Breakout rooms** 셋 다 태블릿 다수 접속 시 렉 심각하다고 보고. Aura-board의 비기능 요구사항 중 최우선.

---

## 0. 핵심 명제

> **Aura-board의 차별화 축 = "저가 태블릿 30대에서 렉 없이 돌아가는 교실 보드 + 성능 좋은 모둠(Breakout) 격리"**

Canva 통합 기능들이 아무리 풍부해도 태블릿에서 끊기면 교실 도구로 실패. 모든 기능 제안은 **태블릿 영향 평가**를 1차 필터로 통과해야 함.

---

## 1. 기존 제품 렉 원인 분석 (벤치마크)

| 관찰된 현상 | 추정 원인 |
|---|---|
| Padlet 보드 로딩 느림 | 전체 카드 일괄 로드, 이미지 풀 해상도 즉시 다운로드 |
| Padlet Breakout도 렉 | 전체 보드 상태 후 클라 필터, WebSocket 방송이 모든 섹션에 브로드캐스트 |
| Canva Teamspaces 편집 렉 | 실시간 커서 + 전체 렌더링 엔진 + iframe 중첩 |
| iframe 카드 많은 보드 멈춤 | Canva 프론트엔드 N번 로드, iPad Safari 메모리 압박 |

---

## 2. 성능 예산 (수용 기준 전역 적용)

기준 단말: **갤럭시 탭 S6 Lite, Chrome Android, 학교 Wi-Fi 50 Mbps 공유**

| 지표 | 목표 |
|---|---|
| 초기 TTI (30카드 보드) | < 3s |
| 드래그 중 프레임 | 60fps 유지 |
| 동시 접속 30명, 카드 변경 전파 지연 | < 500ms |
| 섹션 전환 체감 | < 300ms |
| iframe 동시 마운트 수 상한 | 3개 |
| 메모리 점유 (1시간 사용 후) | < 500MB |
| WebSocket 평균 메시지 크기 | < 2KB |

**미달 시 기능 배포 금지** — feature 파이프라인의 phase9(qa_tester)에 성능 매트릭스 추가.

---

## 2a. 30-카드 5×6 정형 격자 성능 예산 (assignment-board Seed 11)

> 추가일: 2026-04-14 · 시드: `seed_38c34e91bf28` · 관련: `plans/assignment-board-roadmap.md §7`
> 대상: `Board.layout="assignment"` 보드 뷰 (학급 N≤30, 30 slot × 5×6 정형 격자)

§2 일반 예산 위에 얹는 **결정적 격자 전용 추가 게이트**. `DraggableCard` 대신 정적 `StaticSlotCard`를 쓰므로 드래그·실시간 재배치 비용이 0인 대신, 30장 썸네일 동시 디코딩과 S-Pen 오인식이 주 리스크.

| 지표 | 목표 | 근거 |
|---|---|---|
| DOM 카드 노드 수 | ≤ 30 (N≤30 하드 제한, Q3 결정) | 북극성 primitive 5 "자리표" 은유. virtualization 불필요 |
| 카드당 DOM 자식 수 | ≤ 6 (이름·번호·상태 배지·썸네일·아이콘·hitbox) | 총 180 노드 수준으로 탭 S6 Lite 4GB RAM 여유 |
| 썸네일 단일 사이즈 | 160×120 WebP 서버 리사이즈 | T0-④ 파이프라인 재사용. 원본 응답 금지 |
| 썸네일 디코딩 전략 | `loading="lazy"` + IntersectionObserver | 뷰포트 밖(주로 4~6행) 디코딩 지연 |
| 카드 상태 토글 | 순수 CSS `[data-submission-status]` 속성 선택자 | React 리렌더 회피. 30장 상태 변경이 전체 트리 리렌더 트리거하지 않아야 함 |
| 카드 표면 S-Pen 캔버스 | **금지** (모달 내부에서만 마운트) | 카드 `onTap`=모달 오픈 단일 핸들러. S-Pen 오인식 방지 |
| 드래그 바인딩 | **제거** (StaticSlotCard 컴포넌트 신규) | 격자 고정, `order: {slotNumber}` CSS Grid만 사용 |
| iframe | **v1 금지** | Canva 임베드는 썸네일만. v2 "라이브 보기" 단일 iframe 검토 |
| 모달 마운트 정책 | 지연 마운트 + 닫기 시 즉시 언마운트 | 필기 캔버스·첨부 업로드 UI는 모달 내부 전용 |
| WebSocket 채널 | `board:${id}:assignment` 단일 (slot별 분리 금지) | 메시지 < 200B (slotId + status 델타), 100ms 디바운싱 |
| 모달 오픈 TTI | < 500ms | Submission·Card 상세 단일 fetch, 서버 측 조인 쿼리 1회 |
| 30장 동시 제출 시 채널 부하 | 평균 메시지 < 200B | 동시 쓰기 충돌 없음(slot 1:1). §2 < 2KB 예산 하회 |

**phase9 QA 게이트 추가 항목 (assignment-board 한정)**:

```
- [ ] 갤럭시 탭 S6 Lite Chrome 실측 30장 격자 TTI < 3s
- [ ] 30장 썸네일 전부 로드 후 메모리 < 500MB (§2 예산 승계)
- [ ] 카드 상태 전환 시 React Profiler로 리렌더 범위 ≤ 해당 카드 (전체 트리 X)
- [ ] 카드 표면 S-Pen 이벤트 리스너 0개 (CodeGrep 검증)
- [ ] `StaticSlotCard`가 `react-dnd`·`dnd-kit` import 없음 (드래그 의존성 0)
- [ ] 모달 닫힘 후 DOM query로 `.submission-modal` 노드 부재 검증 (언마운트 확인)
- [ ] WebSocket 메시지 스냅샷 평균 < 200B, p95 < 500B
```

§2 공통 예산과 중복되는 항목은 승계하며, 위 표는 **격자 전용 제약의 구체화**다.

---

## 3. 아키텍처 원칙 (비기능 요구사항)

| 원칙 | 구현 지침 |
|---|---|
| **서버 측 스코프 쿼리** | 클라는 절대 전체 보드를 받지 않음. 섹션/뷰포트 단위 페치 |
| **WebSocket 채널 분리** | room key = `board:${id}:section:${id}` — 관심 없는 이벤트 수신 금지 |
| **델타 기반 페이로드** | 전체 카드 객체 금지, 변경 필드만 broadcast, 100ms 디바운싱 |
| **뷰포트 가상화** | IntersectionObserver로 뷰포트 안 카드만 활성 렌더 |
| **iframe 지연 마운트** | 기본은 썸네일, "라이브 모드" 토글 또는 탭 시에만 iframe 마운트 |
| **이미지 파이프라인** | Vercel Image Optimization + `srcset` + `loading="lazy"`, 원본 노출 금지 |
| **섹션 라우트 분리** | `/b/:slug/s/:sectionId` — 섹션 전환은 라우트 교체, 컴포넌트 트리 통째로 교체 |
| **네트워크 후퇴** | 오프라인 감지 시 읽기 전용 캐시 모드 (SWR + idb-keyval) |

---

## 4. 기존 Canva 로드맵 항목별 태블릿 영향 평가

| 항목 | 태블릿 영향 | 대응 |
|---|---|---|
| P0-① oEmbed 라이브 카드 | 🔴 고위험 (iframe 폭발) | **가상화 + 라이브 모드 토글 + 보드당 iframe ≤3** 요건 추가 |
| P0-② Content Publisher Intent | 🟢 영향 낮음 | 학생이 Canva 앱에서 게시, Aura-board 측 부담 無. **태블릿 친화**. 우선 배포 권장 |
| P1-③ Autofill | 🟡 교사 전용 | 학생 태블릿 영향 無. 결과 배포 시 썸네일 우선 |
| P1-④ Quiz 자동 생성 | 🟢 플레이는 가벼운 Kahoot 스타일 | 유지 |
| P2-⑤ Webhook refresh | 🟡 서버 측 처리 | 카드 stale 방지로 오히려 이득 |
| P2-⑥ LMS LTI | 🟢 영향 낮음 | 유지 |
| ~~P1-⑦ Teamspace 동기화~~ | 🔴🔴 **매우 고위험** | **폐기 권장** (또는 교사 기기 전용 지연 동기화로 격하) |

---

## 5. 별도 작업: T0-① 섹션 격리 Breakout 뷰 (성능 우선)

> **크로스 참조**: 이 작업은 `plans/breakout-room-roadmap.md` (Seed 6 `seed_bb1d4eb1c442`, 2026-04-12)의 **구현 전제**다. Breakout Room 보드의 BR-5(배포모드 3종)·BR-6(열람모드+RBAC)는 아래 `Section.accessToken` / `src/lib/realtime.ts` 채널 키(`board:${bid}:section:${sid}`) / `viewSection` RBAC에 직접 의존한다. **`Section.accessToken` 필드 마이그레이션은 본 T0-①에서 선행**되며 BR-1은 이를 재사용한다(중복 마이그레이션 금지).

### 개요
학생은 모둠 링크로 진입 → 본인 모둠 섹션만 보고 기여. Padlet Breakout의 기능을 **성능 좋게** 재구현.

### 수용 기준
- [ ] 섹션 링크 진입 시 **해당 섹션 카드만** 서버에서 수신 (네트워크 패널로 검증)
- [ ] 갤럭시 탭 S6 Lite 30명 동시 접속, 섹션 내 카드 50장 기준 TTI < 3s
- [ ] 섹션 전환 시 이전 섹션 DOM/iframe 완전 언마운트 (메모리 프로파일 검증)
- [ ] 다른 섹션의 카드 변경 이벤트가 WebSocket으로 도달하지 않음
- [ ] 교사 뷰에서는 모든 섹션 통합 뷰 제공 (권한 기반 분기)

### 변경 지점
| 파일 | 변경 |
|---|---|
| `src/app/board/[id]/s/[sectionId]/page.tsx` (신규) | 섹션 전용 서버 컴포넌트 |
| `src/app/api/sections/[id]/cards/route.ts` (신규) | 섹션 스코프 카드 페치 |
| `src/lib/realtime.ts` (신규 또는 확장) | 채널 키 = `board:${bid}:section:${sid}` |
| `src/lib/rbac.ts` | `viewSection` 권한 — 섹션 토큰/멤버십 검증 |
| `prisma/schema.prisma` | `Section.accessToken String?` 추가 (링크 격리용) |
| `src/app/(teacher)/sections/[id]/share/page.tsx` (신규) | 교사용 모둠 링크 발급 UI |

### 리스크
- 교사 통합 뷰가 무거워질 수 있음 → 교사 뷰는 기본 "섹션 목록 + 각 섹션 요약"으로, 상세는 섹션별 진입
- 기존 `/board/[id]` 뷰와의 라우팅 충돌 — 하위 경로로 처리

### 공수: 1주

---

## 6. 별도 작업: T0-② iframe 가상화 + 라이브 모드 토글

### 개요
Canva oEmbed 카드가 여러 개일 때 iframe 동시 마운트로 인한 메모리/네트워크 폭발 방지.

### 수용 기준
- [ ] 기본 렌더는 썸네일 + 재생 아이콘 오버레이
- [ ] 카드 탭/클릭 시에만 iframe 활성화
- [ ] 보드 전체에서 동시 활성 iframe ≤ 3 (LRU로 오래된 것 언마운트)
- [ ] 뷰포트 밖으로 나간 카드는 iframe 즉시 언마운트
- [ ] 카드 상단에 "라이브" 토글 상태 표시

### 변경 지점
| 파일 | 변경 |
|---|---|
| `src/components/DraggableCard.tsx` | `CanvaEmbedSlot` 분리 — 기본 Image, active 시 iframe |
| `src/hooks/useIframeBudget.ts` (신규) | 전역 LRU로 활성 iframe 수 제한 |
| `src/hooks/useInViewport.ts` (신규) | IntersectionObserver 래퍼 |

### 공수: 2~3일

---

## 7. 별도 작업: T0-③ WebSocket 페이로드 최적화

### 개요
현재 구현 추정(실시간 엔진 미확정)과 무관하게, 실시간 도입 시 최초부터 경량 프로토콜 강제.

### 수용 기준
- [ ] 카드 이동 이벤트 메시지 < 200 바이트 (id + x/y + version)
- [ ] 100ms 디바운싱 — 드래그 중 초당 10회 이하 전파
- [ ] 섹션 채널 분리로 불필요 이벤트 0%
- [ ] 서버가 클라로 보내는 메시지에 불필요한 전체 객체 포함 금지

### 변경 지점
- 실시간 엔진 선택 (Liveblocks vs Yjs vs 자체 WS) 의존 — research task 선행 필요
- `src/lib/realtime-protocol.ts` — 메시지 타입 정의 + 직렬화 전략

### 공수: 3~5일 (엔진 결정 이후)

---

## 8. 별도 작업: T0-④ 이미지/썸네일 파이프라인

### 개요
카드 이미지·Canva 썸네일 모두 적응형 해상도 + lazy load.

### 수용 기준
- [ ] 원본 해상도 응답 금지 (서버가 거부)
- [ ] 뷰포트 진입 전 이미지는 로드 안 함
- [ ] `srcset`으로 디바이스 픽셀 비율 대응
- [ ] 3G 네트워크 에뮬레이션에서 카드 썸네일 < 500KB/장

### 변경 지점
| 파일 | 변경 |
|---|---|
| `next.config.ts` | `images.remotePatterns`에 Canva 도메인 추가 |
| `src/components/CardImage.tsx` (신규) | `<Image>` 래퍼 + `loading="lazy"` + placeholder |
| `src/app/api/canva/thumbnail/route.ts` (신규) | Canva 썸네일 프록시 + 리사이즈 |

### 공수: 2일

---

## 9. 선행 research task (실시간 엔진 결정)

성능 목표를 맞추려면 실시간 동기화 방식 결정이 필수. `_handoff.md`에 이미 "실시간 동기화 방식 결정 (Liveblocks vs Yjs)"가 미결로 남아 있음.

**평가 차원**:
| 차원 | Liveblocks | Yjs | 자체 WS |
|---|---|---|---|
| 태블릿 성능 | 중간 (클라 경량) | 높음 (CRDT로 delta 최소) | 구현 따라 |
| 채널 격리 구현 | 표준 Room 모델 | 수동 | 수동 |
| 교실 30명 확장성 | SaaS 비용 증가 | 자체 호스팅 가능 | 서버 설계 |
| 외부 의존성 | 강함 | 약함 | 없음 |

**권장**: research task로 분리하되 기준 단말(iPad 9th)에서 프로토타입 벤치 필수.

---

## 10. 우선순위 실행 순서

| 주차 | 작업 |
|---|---|
| Week 1 | T0-④ 이미지 파이프라인 (기존 카드에도 즉시 이득) |
| Week 2 | T0-① 섹션 격리 Breakout 뷰 |
| Week 3 | T0-② iframe 가상화 (P0-① oEmbed 카드 선행 조건) |
| Week 4 | research: 실시간 엔진 벤치 |
| Week 5 | T0-③ WebSocket 프로토콜 — 엔진 결정 후 |

Canva 통합 로드맵(`implementation-roadmap.md`)의 P0-①은 T0-② 완료 후 착수, 기타 Canva 항목은 독립 진행 가능.

---

## 11. 전역 수용 게이트 (padlet feature 파이프라인 phase9 추가)

기존 QA 매트릭스에 다음을 **강제 추가**:

```
- [ ] 갤럭시 탭 S6 Lite Safari 실측 TTI < 3s (30카드 보드)
- [ ] 드래그 60fps 유지 (DevTools Performance 패널 증빙)
- [ ] 섹션 채널 분리 검증 (다른 섹션 이벤트 0건)
- [ ] 메모리 프로파일 1시간 사용 후 < 500MB
- [ ] iframe 동시 마운트 ≤ 3 (코드 + 런타임 검증)
```

미달 시 배포 금지.

---

## 12. 결론

Canva 통합 풍부함과 태블릿 성능은 **상충**한다. 이 문서는 상충 관계를 명시화하고, 성능을 상위 제약으로 강제하는 설계 규범을 정의한다. Canva 로드맵의 각 항목은 이 문서의 원칙·예산·게이트를 통과해야만 배포된다.

---

### 변경 로그
| 날짜 | 시드 ID | 변경 |
|---|---|---|
| 2026-04-14 | `seed_38c34e91bf28` | §2a 신규: 30-카드 5×6 정형 격자 성능 예산(assignment-board 전용). StaticSlotCard·CSS 상태 토글·iframe v1 금지·모달 전용 S-Pen 게이트 추가. phase9 QA 체크리스트 확장. |
| 2026-04-16 | `seed_0badf1e571bc` | §2b 최신 제약(수행평가 응시 화면): **iframe 0, OCR 클라우드 오프로드 전용, S-Pen 캔버스 800×400 고정 px**. assessment-autograde 로드맵(`plans/assessment-autograde-roadmap.md` §3) 참조. |

---

## 2b. 최신 제약 — 수행평가 응시 화면 (assessment-autograde Seed 12)

> 추가일: 2026-04-16 · 시드: `seed_0badf1e571bc` · 관련: `plans/assessment-autograde-roadmap.md` §3
> 대상: `AssessmentTemplate` 응시 화면 (학생 `/assessment/[templateId]/take`)

- **iframe 0** — 응시 화면 가상화·LRU 포함 일체 금지 (§2의 ≤3 예산을 0으로 강화)
- **OCR 클라우드 오프로드 전용** — Tesseract.js·온디바이스 OCR 금지. SHORT 문항 `inkImageUrl` PNG → Gemini Vision 1-round-trip
- **S-Pen 캔버스 800×400 고정 px** — tldraw/perfect-freehand, 60fps, 입력 이벤트 직결(throttle 없음)
- 현재 문항 ±1개 lazy 마운트 (30문항 × 4보기 일괄 렌더 금지, Snapdragon 720G TTI < 3s)
- 드래프트 IndexedDB + Supabase autosave 300ms debounce, 문항별 background flush
- Realtime 구독 스코프: 학생=자기 submission만 (보드 전체 ChangeFeed 금지)
