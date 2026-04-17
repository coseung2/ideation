# Canva Developer × Aura-board 통합 리서치 보고서

> 작성일: 2026-04-12
> 대상 프로젝트: Aura-board (상위 폴더 `padlet/`) — 교실 학습 플랫폼 (Next.js 16 + Prisma + SQLite/Postgres)
> 리서치 범위: Canva Developer 제품군 전체 + 실제 통합 사례 + Aura-board 맞춤 매핑

---

## 1. Canva Developer 플랫폼 개요

Canva는 2개의 독립된 개발자 경로를 제공한다.

| 축 | 방향 | 인증 | 주요 용도 | Aura-board 적합성 |
|---|---|---|---|---|
| **Connect API** | 외부 앱 → Canva | OAuth 2.0 + PKCE | 디자인 생성, Export, Autofill, Import, 폴더 관리, Webhook | ★★★ 직접 매칭 |
| **Apps SDK** | Canva 에디터 내부 iframe | Canva 세션 + 앱 스코프 | 에디터 확장, 사이드패널 앱, 외부 배포(Intent) | ★★ 역방향 파트너십 |

### 1.1 2026년 대규모 업데이트 (신규 12 APIs)

| API | 기능 |
|---|---|
| Design Editing API | 레이아웃/엘리먼트 프로그램적 읽기·수정 |
| Data Connectors | CRM·스프레드시트 → Canva 차트·콘텐츠 |
| Design Import by URL | URL만으로 외부 디자인 Canva 로드 |
| Resize API | 여러 포맷 자동 변환 |
| Assets API (video) | 비디오 임포트 포함 |
| Design Metadata | 제목/페이지 치수/duration 조회 |
| Toast Notifications | 에디터 내부 피드백 메시지 |
| Get User Capabilities | 플랜/역할별 기능 분기 |
| Dev Toolkit (in-editor) | 실시간 디버깅·로컬라이제이션 |
| **Canva Dev MCP Server** | Cursor·Claude Code에서 Canva 앱 개발 가속 |

### 1.2 주요 공식 레퍼런스

- Connect API Docs: https://www.canva.dev/docs/connect/
- Apps SDK Docs: https://www.canva.dev/docs/apps/
- Connect Starter Kit: https://github.com/canva-sdks/canva-connect-api-starter-kit
- Apps SDK Starter Kit: https://github.com/canva-sdks/canva-apps-sdk-starter-kit
- OpenAPI 스펙: https://www.canva.dev/sources/connect/api/latest/api.yml

---

## 2. 실제 통합 사례 벤치마크

| 플랫폼 | 통합 방식 | 학습 포인트 |
|---|---|---|
| **HubSpot Data Connector** | Connect API + Data Connector | HubSpot 데이터 → Canva 차트 자동 갱신, Canva 디자인을 HubSpot에서 리뷰 |
| **Slack** | Connect API | 디자인 공유·승인을 Slack 채널화 |
| **Notion (native)** | Smart embed link (oEmbed) | **코드 없는 통합** — Share → Embed 링크 → Notion paste, 라이브 동기화 |
| **ChatGPT** | Connect API | 대화에서 Canva 호출해 디자인 생성 |
| **Canva for Education (LTI)** | LMS 표준 | Google Classroom / Canvas / Moodle / Teams / D2L / Blackboard / Schoology에 LTI로 직결 |

### 2.1 Aura-board가 참조해야 할 모델

- **저마찰 기본**: Notion 방식 (Smart embed link paste)
- **플랫폼화 심화**: Content Publisher Intent (Canva → Aura-board 역방향)
- **엔터프라이즈 참고**: HubSpot Data Connector — 교사 대시보드에 Canva 데이터 가져오기 구상 시

---

## 3. Aura-board 현재 상태 (감사)

### 3.1 이미 구현된 Canva 연동

| 영역 | 위치 |
|---|---|
| OAuth PKCE | `src/lib/canva.ts` — `buildAuthorizationUrl`, `exchangeCode`, `getAccessToken`, `isCanvaConnected` |
| Design 조회 | `canvaGetDesign`, `resolveCanvaDesignId` |
| PDF/PNG Export | `canvaExportDesign` |
| Folder 관리 | `canvaCreateFolder`, `canvaListFolderItems`, `canvaMoveItem` |
| API 라우트 | `/api/canva/design/[id]`, `/api/canva/folders/...`, `/api/canva/organize` |
| 보조 스킬 | `ideation/canva-assignment-pdf-merge/SKILL.md` — 과제 완료본 PDF 자동 병합 |

### 3.2 교실 맥락 모델(스키마)과 Canva 기능 매칭 포인트

| Aura 모델 | 연결 가능한 Canva 기능 | 시나리오 |
|---|---|---|
| `Classroom.code` | Apps SDK Settings UI | 학생이 Canva 앱에서 교실 6자리 코드로 1회 페어링 |
| `Student.qrToken` | Content Publisher 인증 | QR 1회 스캔으로 Canva 앱에 학생 신원 고정 |
| `Submission` | Export API + PDF 병합 스킬 | 기존 스킬 확장 — 제출물 일괄 PDF |
| `Section` | Canva Folder | 이미 연결(organize) — 양방향 동기화 추가 여지 |
| `Quiz.sourceFile` | Export + Design Editing | Canva 슬라이드 → QuizQuestion 자동 파싱 |
| `Card.imageUrl` | Autofill | 보드 메타로 커버 이미지 자동 생성 |

---

## 4. 갭 분석 — 6개 작업 묶음

### P0 (즉시 가치 — 기존 코드 재사용 가능)

**① oEmbed 라이브 카드**
- 현 상태: Card 모델에 `linkUrl/linkImage` 존재, `canvaGetDesign` 있음. 그러나 라이브 iframe 렌더러 없음.
- 추가: DraggableCard에 Canva 서브타입 분기, `<iframe>` + Canva oEmbed 엔드포인트, CSP `frame-src canva.com`
- 가치: Notion 수준의 "링크 붙이면 라이브 임베드" UX

**② Content Publisher Intent 앱**
- 현 상태: 0
- 추가: 별도 Canva 앱 프로젝트 (Apps SDK) + Aura-board에 수신 엔드포인트 `POST /api/external/cards`
- 가치: Canva에서 "Aura-board로 보내기" — 학생 이탈 0

### P1 (교사 고부가 기능)

**③ Autofill**
- 현 상태: 0 (OAuth는 있음)
- 추가: 교사 UI ("브랜드 템플릿 + 학생 명단 → 수료증 30장"), `POST /autofills` 호출 + 폴링 + 결과 PDF 일괄 배포
- 가치: 교사 수작업 시간 단축 (이름표/수료증/개별 피드백 카드)

**④ Quiz 자동 생성**
- 현 상태: Quiz 모델 있음, 수기 입력만 지원
- 추가: 교사가 Canva 슬라이드 붙이면 Export → 텍스트 추출 → QuizQuestion 자동 생성 (Design Editing API 사용 시 구조 추출 더 정확)
- 가치: 수업자료 재활용 — 기존 슬라이드 그대로 Kahoot-style 퀴즈화

### P2 (플랫폼 성숙도)

**⑤ Webhook 기반 카드 자동 refresh**
- 현 상태: 0 — 카드가 stale
- 추가: Canva Webhook 구독 → Aura-board가 썸네일 재생성 + 버전 증가
- 가치: 교사가 Canva에서 수정 → 보드 카드 자동 갱신

**⑥ LMS LTI 어댑터**
- 현 상태: 0
- 추가: LTI 1.3 provider 구현 (Google Classroom, Canvas 등에 Aura-board를 앱으로 등록)
- 가치: 이미 Canva for Education이 LTI로 깔린 학교에서 Aura-board 동시 채택 경로

---

## 5. 리스크 & 결정 대기 사항

| 항목 | 이슈 | 필요 결정 |
|---|---|---|
| Canva 플랜 제한 | Content Publisher Intent는 Pro/Business/Enterprise/**Education**/Nonprofits만 가능 (Education은 교사 무료) | 타겟 학교가 Canva Edu 가입 여부 확인 |
| 이미지 저장소 | 현재 `Card.imageUrl`만 있고 물리 저장소 미정 | Vercel Blob vs S3 vs 자체 호스팅 |
| Student ↔ User 매핑 | Student 모델은 User와 분리 (QR/textCode 로그인) | Canva OAuth 결과를 Student에 연결할 규칙 |
| 마켓플레이스 심사 | Content Publisher 공개 배포는 Canva 심사 필요 | 초기엔 프라이빗 배포로 학교 단위 파일럿 |
| PNG 단일 슬롯 | Publisher Intent는 단일 PNG — 다중 페이지 발표자료 한계 | 다페이지는 Connect Export(PDF)로 보조 |

---

## 6. 결론

Aura-board는 **이미 Canva Connect API의 기초 인프라(OAuth, Design 조회, Export, Folder)를 갖춘 상태**다. 남은 격차는 (a) 카드 렌더러에 라이브 임베드를 붙이는 짧은 작업, (b) Canva 에디터 쪽에서 보드로 역방향 게시하는 Apps SDK 앱, (c) 교사 업무 자동화용 Autofill — 이 3가지가 핵심 레버리지 포인트다. 교실 컨텍스트(Classroom/Student/Submission/Quiz)와 Canva 기능이 자연스럽게 매칭되는 지점이 많아, 단순 "디자인 도구 연동"을 넘어 **교실 전용 통합 워크플로우**로 차별화 가능하다.
