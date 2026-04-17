# Aura-board × Canva 통합 착수 플랜

> 작성일: 2026-04-12
> 대상: Aura-board (상위 `padlet/` 프로젝트)
> 원칙: **이 문서는 `ideation/` 내 기획 산출물**. 실제 구현은 `padlet/`의 feature 파이프라인(phase0→phase10)에 별도 task로 등록.

---

## 공통 전제

- 스택: Next.js 16 App Router + Prisma + 순수 CSS + @dnd-kit + zod
- 이미 있음: `src/lib/canva.ts`(OAuth/Export/Folder), `/api/canva/*` 라우트, PDF 병합 스킬
- 저장소 결정 필요: 이미지 저장소(Vercel Blob 권장). 다수 작업이 이 결정에 블로킹됨

---

## P0-① oEmbed 라이브 Canva 카드

### 개요
Canva 디자인 URL을 카드에 붙이면 라이브 iframe 임베드. 원본 수정 시 자동 반영(Notion 방식).

### 수용 기준
- [ ] 카드에 `canva.com/design/...` URL 입력 시 3초 내 임베드 렌더
- [ ] 원본 변경이 30초 내 자동 반영
- [ ] viewer 역할도 보기 가능, 편집은 불가
- [ ] 비공개 디자인 URL은 에러 상태 카드로 폴백

### 변경 지점
| 파일 | 변경 |
|---|---|
| `src/lib/canva.ts` | `resolveCanvaEmbedUrl(url)` 함수 추가 — Canva oEmbed 엔드포인트 호출해 embed URL + thumbnail + title 반환 |
| `src/components/DraggableCard.tsx` | `linkUrl`이 Canva 패턴이면 `<iframe src={embedUrl} sandbox=...>` 렌더 분기 |
| `src/app/api/cards/route.ts` (POST) | 링크 입력 시 Canva 판별 → `linkImage/linkTitle` 자동 채우기 |
| `next.config.ts` | CSP headers — `frame-src https://www.canva.com` 추가 |
| (선택) `prisma/schema.prisma` | `Card.kind String @default("link")` 추가로 미래 확장 대비 |

### 의존 / 리스크
- Canva 비공개 디자인은 iframe에서 로그인 요구 → 교사 가이드 "Public view link로 공유" 필요
- 학교망 canva.com 차단 시 → `linkImage` 썸네일 폴백 표시

### 공수: 1~2일

---

## P0-② Content Publisher Intent 앱 (Canva → Aura-board)

> **서버 수신 구현 (padlet 측)은 Seed 8(`canva-publisher-receiver-roadmap.md`)에서 처리**.
> 본 섹션은 상위 기획(Canva 앱 + Aura 수신 엔드포인트 2개 산출물 개요)만 담당하고, `/api/external/cards` 수신 엔드포인트·PAT 시스템 확장·교사 토큰 UI·3-stage 마이그·Tier 게이팅·Upstash rate-limit·Vercel Blob 스트리밍 업로드·1회 노출 모달의 상세 사양과 작업 분할(CR-1~CR-10)은 Seed 8 로드맵을 참조한다. 이 섹션은 Canva 앱 프로젝트(`content-publisher-app/`) 측 내용의 single source of truth로 남는다.

### 개요
별도 Canva 앱 프로젝트. 학생이 Canva 에디터에서 "Aura-board에 게시" → 보드로 카드/제출 자동 생성.

### 수용 기준
- [ ] 교사가 Canva에서 Aura-board 앱 연결 시 본인 교실 목록 표시
- [ ] 학생이 Canva 디자인 완성 → "Aura-board에 게시" → 10초 내 해당 보드에 카드 생성
- [ ] 업로드 실패 시 Canva 에디터 Toast로 오류 알림
- [ ] viewer 역할 학생은 앱 내 게시 버튼 비활성화

### 구성 (2개 산출물)
**A. Canva 앱 프로젝트** (새 디렉토리 `ideation/content-publisher-app/`)
- Apps SDK Starter Kit 포크
- `src/intents/content_publisher/index.tsx`
  - `renderSettingsUi` — Classroom/Board/Section 드롭다운 + 캡션
  - `renderPreviewUi` — Aura-board 카드 미리보기
  - `getPublishConfiguration` — PNG 1장, 비율 4:5~1.91:1
  - `publishContent` — PNG fetch → Aura-board `POST /api/external/cards`
- 스코프: `canva:design:content:read`

**B. Aura-board 수신 엔드포인트** (padlet feature task로 등록)
- `POST /api/external/cards` — Canva 앱에서 호출, 보드 멤버십 검증 후 Card 생성
- 인증: 교사 Personal Access Token (초기) → 추후 OAuth2 provider 전환
- `src/lib/external-auth.ts` — 토큰 발급/검증

### 의존 / 리스크
- Canva 플랜 제한 (Edu 교사는 무료) — 학교 Canva for Education 가입 선행
- 공개 마켓플레이스 배포는 심사 필요 → 초기 프라이빗 공유로 파일럿

### 공수: 1~2주

---

## P1-③ Autofill — 교사 대량 생성

### 개요
교사가 브랜드 템플릿 + 학생 명단 선택 → 수료증/이름표 30장 일괄 생성 → 보드에 자동 배포.

### 수용 기준
- [ ] 교사 UI에서 Canva 템플릿 선택 가능
- [ ] Classroom의 학생 명단 자동 주입 (name/number)
- [ ] 30장 기준 2분 내 완료 + 진행률 표시
- [ ] 생성물이 지정 섹션에 카드로 일괄 생성

### 변경 지점
| 파일 | 변경 |
|---|---|
| `src/lib/canva.ts` | `canvaStartAutofill(token, templateId, data[])` + `canvaPollAutofill(token, jobId)` 추가 |
| `src/app/api/canva/autofill/route.ts` (신규) | 교사 호출 엔드포인트 + 폴링 워커 |
| `src/app/(teacher)/autofill/page.tsx` (신규) | 템플릿 선택 + 학생 필드 매핑 UI |
| `src/components/AutofillProgress.tsx` (신규) | 진행률 표시 |

### 의존 / 리스크
- 긴 폴링 → Vercel Queues 또는 Cron으로 비동기화 권장 (Functions timeout 300s 내 대부분 커버되지만 안전하게)
- 템플릿 필드명 규약 필요 (`{student_name}`, `{student_number}`)

### 공수: 3~5일

---

## P1-④ Quiz 자동 생성 (Canva 슬라이드 → QuizQuestion)

### 개요
교사가 Canva 발표자료 URL을 붙이면 Export → 텍스트/구조 추출 → QuizQuestion 자동 시드.

### 수용 기준
- [ ] 10페이지 이내 Canva 슬라이드를 Quiz로 변환 가능
- [ ] 질문/옵션/정답 추출 실패 항목은 수동 편집 UI로 폴백
- [ ] 교사가 변환 후 저장 전 확인·수정 가능

### 변경 지점
| 파일 | 변경 |
|---|---|
| `src/lib/canva.ts` | Design Editing API 래퍼 — 페이지별 텍스트 엘리먼트 추출 |
| `src/app/api/quiz/import-from-canva/route.ts` (신규) | URL → QuizQuestion[] 파싱 |
| `src/app/(teacher)/quiz/new/page.tsx` | "Canva로 가져오기" 입력 필드 추가 |

### 의존 / 리스크
- Design Editing API는 2026 신규 → SDK 버전 확인 필요
- 자유형식 슬라이드 파싱 품질 변동성 — 명시적 템플릿 권장

### 공수: 1주

---

## P2-⑤ Webhook — 카드 자동 refresh

### 개요
Canva Webhook 구독 → 디자인 변경 이벤트 수신 → 보드 카드 썸네일/메타 갱신.

### 수용 기준
- [ ] Canva에서 디자인 수정 후 1분 내 모든 참조 카드 갱신
- [ ] Webhook 서명 검증
- [ ] 실패 이벤트 재시도 큐

### 변경 지점
| 파일 | 변경 |
|---|---|
| `src/app/api/webhooks/canva/route.ts` (신규) | Webhook 수신 + 서명 검증 + 큐 enqueue |
| `src/lib/canva.ts` | Webhook 구독/해지 관리 함수 |
| `prisma/schema.prisma` | `Card.externalId String?` + `externalUpdatedAt DateTime?` 추가 |

### 공수: 2~3일

---

## P2-⑥ LMS LTI 어댑터

### 개요
Aura-board를 LTI 1.3 provider로 등록 → Google Classroom / Canvas LMS에서 학습 도구로 바로 연결.

### 수용 기준
- [ ] Canvas LMS에서 Aura-board 추가 후 보드가 수업에 임베드됨
- [ ] LTI deep linking으로 특정 보드 지정 가능
- [ ] Grade passback (Submission.grade → LMS) 동작

### 변경 지점
- 새 라이브러리 의존: `ltijs` 또는 직접 구현
- `src/app/lti/` 엔드포인트 세트 (login, launch, deep-link, grades)
- Classroom 모델에 `ltiContextId String?` 추가

### 의존 / 리스크
- LTI 1.3 인증서/JWKS 관리 필요
- 학교별 플랫폼 등록 프로세스 존재 — 파일럿 학교 1곳 선행 필수

### 공수: 2~3주

---

## 의존 그래프

```
[저장소 결정 (Vercel Blob)]
  ├─ P0-① oEmbed (약결합 — 썸네일 캐시용)
  ├─ P0-② Content Publisher ← 강결합 (PNG 업로드 필수)
  ├─ P1-③ Autofill ← 강결합 (PDF 일괄 저장)
  └─ P1-④ Quiz 자동 ← 중결합 (슬라이드 PDF 저장)

[P0-② Content Publisher Intent]
  ↓ (앱 UX 틀 확정)
[P2-⑤ Webhook] — 독립
[P2-⑥ LTI] — 독립, 파일럿 학교 필요
```

## 권장 실행 순서

1. **Week 1**: 저장소 결정 + P0-① 완료
2. **Week 2~3**: P0-② Canva 앱 + Aura 수신 엔드포인트
3. **Week 4**: P1-③ Autofill
4. **Week 5**: P1-④ Quiz 자동 생성
5. **Week 6~7**: P2-⑤ Webhook
6. **Week 8~10**: P2-⑥ LTI (파일럿 학교 확보 후)

---

## Padlet 파이프라인 연결 지침

각 항목을 padlet `feature` 파이프라인의 새 task로 올릴 때 phase0 request에 필요한 필드:

```json
{
  "type": "feature",
  "title": "<항목 제목>",
  "goal": "<수용 기준 기반 1문장>",
  "context_ref": "ideation/plans/implementation-roadmap.md#<섹션 앵커>",
  "research_ref": "ideation/research/canva-developer-aura-research.md"
}
```

이 문서들이 각 task의 phase1(planner) 입력으로 들어가 스코프 결정의 기초가 된다.
