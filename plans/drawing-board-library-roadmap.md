# 그림보드 × 학생 라이브러리 — 설계 노트

> 작성일: 2026-04-12
> 관련 문서: `tablet-performance-roadmap.md`, `implementation-roadmap.md`, `plant-journal-roadmap.md`
> 톤: 아이디어 정교화. 정식 스펙 아님.

---

## 해결하려는 페인 포인트

```
❌ 현재 (이비스페인트 + 패들렛)
그리기 → 갤러리 저장 → 앱 전환 → 패들렛 → 업로드 → 파일 찾기 → 제출

✅ 목표
그리기 → "저장"(자동 라이브러리) → 다른 보드에서 "내 라이브러리" → 원클릭 삽입
```

학생 창작물이 **Aura-board 내부에서 순환**. 로컬 갤러리 경유 제거.

---

## 확정 결정

| 결정 | 내용 |
|---|---|
| 드로잉 도구 | **Drawpile** (자체 호스팅 + 포크) |
| 그림보드 형태 | **전용 보드 레이아웃** (`Board.layout = "drawing"`) — 미술 수업 전용 |
| 라이센스 격리 | Drawpile 포크는 별도 GPL-3.0 레포. Aura-board 본체는 네트워크 경계 너머에서 호출 → GPL 전파 없음 |
| 개발 공수 | 감수. 에이전트가 구현 |
| 기준 단말 | 갤럭시 탭 S6 Lite (iOS 아님) |
| 이어그리기 | **덮어쓰기**. 버전 히스토리 안 둠. 라이브러리 원본 갱신, 제출된 카드 사본은 항상 불변 |
| 공유 범위 | **반 공유 기본** + 학생이 개별 애셋에 '비공개' 토글 가능. 교사는 반 학생 전체 열람(관리). **학부모(role="parent") 열람 범위는 Seed 7 매트릭스(`parent-viewer-roadmap.md#5`)로 일원화** — 자녀 본인 `StudentAsset` 전체 노출(isSharedToClass·isPrivate 무관), 타 학생 자산은 서버 필터링으로 제거 |
| 협업 모드 | **v1은 단일 사용자 모드만**. 협동 세션은 v2+ 파킹 |
| 용량 정책 | **Tier 기반 요금제** (백엔드 스토리지 원가 연동). 교사 1인 구독 단위, 학생 무료 참여. 구체 구간·무료 한도 별도 결정 |
| 과금 단위 | **교사 1인 구독** — 쿼터 귀속 Teacher(User). 반 분배는 교사 재량 |
| 라이브러리 정리 | **태그 + 폴더 둘 다**. 자동 태그(저장 출처·날짜) + 학생 자유 태그. 교사 기본 폴더 세트(숙제/미술/자유작 등) + 학생 개인 폴더 추가. 한 작품 여러 태그 가능, 폴더는 하나 |
| 교사 템플릿 | **보드당 여러 ORA 템플릿 등록**, 학생이 '템플릿 고르기' 모달에서 1개 선택 (빈 캔버스 옵션 포함). 교사는 본인 템플릿함에 누적 |

---

## 왜 Drawpile

- 200+ 브러시, 레이어, 블렌드 모드 — 이비스페인트 대체 수준
- ORA(OpenRaster) 저장 → 레이어 보존 → **이어 그리기** 가능
- 오픈소스, 자체 호스팅 가능
- 웹 클라이언트 존재, iPad/Android 브라우저 지원
- S-Pen 필압 Pointer Events로 수신 가능
- **Kleki가 기능 부족하다는 현실** 감안

단점: GPL-3.0, WebSocket 서버 필요, 저장 → 라이브러리 자동 경로는 우리가 만들어야 함 (이 셋 모두 감수 결정).

---

## 아키텍처 스케치

```
[Aura-board (Next.js)]
  ↓ iframe
[drawpile.aura-board.app (자체 호스팅, nginx + TLS)]
  ↓ WebSocket (TLS)
[Drawpile 서버 (VPS: Railway/Fly.io/Hetzner)]
  ↓ 저장
[볼륨: ORA/DPREC 영속 저장]

부모(Aura-board) ↔ iframe(Drawpile) postMessage 브리지
```

Drawpile 포크 레포: `aura-board/drawpile-fork` (GPL-3.0 공개)
- 최소 패치: postMessage 저장 이벤트, 세션 자동 로그인, UI 한국어/학생 톤

---

## postMessage 프로토콜 (우리가 정의)

```
iframe → 부모
  { type: "drawpile:ready" }
  { type: "drawpile:save", png: Blob, ora: Blob, title: string }
  { type: "drawpile:stats", strokes: number, layers: number, timeMs: number }

부모 → iframe
  { type: "aura:init", studentId, classroomId, assetToResume? }
  { type: "aura:save-complete", assetId }
```

보안: `event.origin === "https://drawpile.aura-board.app"` 검증 필수.

---

## 데이터 모델 초안

```prisma
model StudentAsset {
  id                 String   @id @default(cuid())
  studentId          String
  classroomId        String?
  kind               String   // "drawing" | "photo" | "upload" | "imported"
  sourceBoardId      String?  // 어느 그림보드에서 만들었나
  url                String   // PNG/JPG (표시·제출용)
  projectUrl         String?  // ORA/DPREC (재편집용)
  thumbnailUrl       String
  title              String
  tags               String?  // 쉼표 구분
  createdWith        String   // "drawpile" | "upload" | "import"
  drawpileSessionId  String?
  strokeCount        Int?
  isSharedToClass    Boolean  @default(false)
  createdAt          DateTime @default(now())
  updatedAt          DateTime @updatedAt

  student     Student            @relation(fields: [studentId], references: [id], onDelete: Cascade)
  attachments AssetAttachment[]

  @@index([studentId])
  @@index([classroomId])
}

model AssetAttachment {
  id        String @id @default(cuid())
  assetId   String
  cardId    String?           // Card에 붙음
  observationId String?       // 또는 PlantObservation 등
  boardId   String
  createdAt DateTime @default(now())

  asset       StudentAsset      @relation(fields: [assetId], references: [id], onDelete: Cascade)
  card        Card?             @relation(fields: [cardId], references: [id], onDelete: SetNull)
  observation PlantObservation? @relation(fields: [observationId], references: [id], onDelete: SetNull)

  @@index([assetId])
  @@index([cardId])
  @@index([observationId])
}
```

**참조 vs 복사**: 카드에 붙을 땐 **복사 기본**(제출물 안정성). 라이브러리에서는 역참조(`AssetAttachment`)로 "이 애셋을 쓴 카드 목록" 제공.

---

## 학생 워크플로우

### 창작
1. 학생: 그림보드 진입 (Board.layout="drawing")
2. Drawpile iframe 로드 + `aura:init` postMessage로 자동 로그인
3. 그리기 (레이어, S-Pen 필압, 브러시 전환)
4. 2분마다 자동 스냅샷 (세션 유실 복구)
5. "라이브러리에 저장" 클릭 → `drawpile:save` 이벤트 → 부모가 `StudentAsset` 생성

### 재사용
1. 다른 보드(숙제 제출 등)에서 카드 작성 중
2. "사진 첨부" 옆 **"내 라이브러리"** 버튼
3. 모달: 썸네일 그리드, 태그/검색, 정렬(최신/제목/사용횟수)
4. 선택 → `AssetAttachment` 생성 + 카드에 이미지 박힘

### 이어 그리기
1. 라이브러리 항목 우클릭(또는 롱프레스) → "이어 그리기"
2. 그림보드 iframe 열림 + `assetToResume`로 ORA 로드
3. 수정 후 저장 → 기존 애셋 덮어쓰기 또는 새 버전 생성(학생 선택)

---

## 그림보드 레이아웃의 UX

일반 freeform 보드와 다름:
- 카드 드래그 없음. 보드 자체가 갤러리·작업실
- 2개 뷰 토글:
  - **작업실 뷰**: Drawpile iframe 풀화면, 현재 작업 중 캔버스
  - **갤러리 뷰**: 학급 학생들이 이 보드에서 만든 애셋 썸네일 그리드
- 교사 설정:
  - 주제 제시 ("바다생물 그리기")
  - 템플릿 ORA 배포 (밑그림)
  - 완성 기한
  - 동료 평가 허용 여부

---

## 라이브러리 UX 포인트

- **사이드바 어디서든 토글** — 학생이 언제 어느 보드에 있든 열림
- 썸네일 그리드, 무한 스크롤
- 태그 편집·검색
- 정렬: 최신/제목/사용횟수
- "이 애셋을 쓴 카드 목록" (역참조)
- 공개 범위 토글: 본인 / 반 공유(원작자 자동 표시)
- 삭제 → 30일 휴지통 → 영구 삭제
- 정렬 기본 최신순

---

## 태블릿(갤럭시 탭 S6 Lite) 체크리스트

- Drawpile WASM 멀티스레드 = Cross-Origin-Isolation 헤더(`COOP: same-origin`, `COEP: require-corp`) 필수
- 캔버스 해상도 기본 2048×2048, 상한 설정 UI에서 조정
- 레이어 수 권고 8, 상한 16
- Chrome Android 탭 kill 대응 → 자동 스냅샷 2분 주기
- S-Pen 필압 Pointer Events (`pointerType === "pen"`, `pressure`)
- 라이브러리 썸네일은 T0-④ 이미지 파이프라인 경유 (<100KB)
- iframe 가상화 규칙 적용 — 동시 Drawpile iframe 1개로 제한 (풀스크린 전용)

---

## 인프라 추가 필요 사항

| 컴포넌트 | 선택지 |
|---|---|
| Drawpile 서버 | Railway / Fly.io / Hetzner VPS (Vercel Functions 범위 밖) |
| TLS + nginx | Caddy 또는 Cloudflare 프론트 |
| 애셋 스토리지 | Vercel Blob (PNG/ORA 양쪽) |
| 도메인 | `drawpile.aura-board.app` (서브도메인) |

---

## 진입 단계 (작업 분할)

| 단계 | 내용 |
|---|---|
| D-0 | Drawpile 포크 레포 생성 (GitHub, GPL-3.0 유지) |
| D-1 | 스키마: `StudentAsset`, `AssetAttachment` 추가 + 마이그레이션 |
| D-2 | Drawpile 서버 자체 호스팅 + HTTPS 설정 |
| D-3 | 포크에 postMessage bridge + 자동 로그인 패치 |
| D-4 | `Board.layout="drawing"` 레이아웃 + Drawpile iframe 통합 |
| D-5 | "라이브러리에 저장" → `StudentAsset` 생성 파이프라인 |
| D-6 | 학생 라이브러리 사이드바 모달 + 썸네일 그리드 |
| D-7 | 기존 카드 작성 UI에 "내 라이브러리" 버튼 |
| D-8 | 이어 그리기 (`assetToResume`) |
| D-9 | 공유 범위·태그·역참조·휴지통 |
| D-10 | 갤러리 뷰·템플릿 배포·교사 설정 |

---

## 파킹 (나중에 꺼내기)

- 반 단위 실시간 협업 그리기 세션 (Drawpile 본연의 강점)
- 교사 템플릿 ORA 배포
- 외부 이비스페인트 업로드 경로 (Share Sheet 통합)
- 버전 히스토리 (갤럭시 탭에서 저장 반복 시 버전 누적)
- 학부모 뷰에서 자녀 포트폴리오 **PDF 내보내기** — 자녀 전체 자산 열람 자체는 Seed 7 (`parent-viewer-roadmap.md`)에서 v1 구현됨. PDF 내보내기만 v2 파킹

---

## 미결 사항

- Drawpile postMessage API 현재 지원 여부 → 실기기 + DevTools로 검증 필요 (사용자 수행)
- Drawpile 서버 비용 감 잡기 (동시 세션 30명 기준)
- ORA 파일 평균 크기 → 저장 용량 전체 예상
- "그림보드 하나 = 학급 하나" vs "주제별 여러 그림보드" 운영 규칙
- **Tier 구체 구간**: 무료/베이직/프로 구간별 용량·기능 매트릭스
- 갤러리 뷰 ↔ 작업실 뷰 전환 UX (탭·토글·별도 페이지)
- 외부 이비스페인트 업로드 경로 (iOS/Android Share Sheet 통합 시점)

## 인터뷰 결과 (2026-04-12, Ouroboros interview_20260412_064317)

최종 ambiguity 0.18 → seed-ready 상태. 주요 결정 위 표에 반영됨. 미결 중 "Tier 구체 구간"은 제품 수익 모델과 직결되어 별도 논의 필요.

---

## 재사용 포인트 (Seed 6 Breakout Room 참조, 2026-04-12 추가)

`plans/breakout-room-roadmap.md`가 본 로드맵의 두 가지 패턴을 명시적으로 승계한다:

1. **복사(independent) 패턴** — `AssetAttachment`의 "카드에 붙을 땐 복사 기본(제출물 안정성), 라이브러리는 역참조" 원칙을 BreakoutTemplate 구조 복제에 1:1로 적용. 템플릿 원본 수정은 기존 파생 Board에 역전파하지 않고, 모둠 간 완전 독립 작업 공간을 보장.
2. **"템플릿 고르기" 모달 UX** — 학생용 "템플릿 고르기(빈 캔버스 옵션 포함)" 모달 패턴이 Breakout Room 교사 개설 플로우(BR-3)의 템플릿 피커에 재사용된다. 공통 컴포넌트 추출 후 두 플로우에서 참조하는 방향으로 설계 정합성을 맞춘다.

향후 이 로드맵의 "교사 ORA 템플릿 배포" 파킹 항목이 다시 꺼내질 경우, Breakout Room의 BreakoutTemplate 구조 모델(`{structure, recommendedVisibility, version}`)을 드로잉 템플릿에도 적용 가능한지 검토한다.

---

## 학부모 열람 범위 (Seed 7 통합, 2026-04-12)

본 로드맵의 "학부모(viewer)는 자녀 전체 열람" 언급은 **Seed 7 `seed_37b35654542f`** (`plans/parent-viewer-roadmap.md`)의 **자녀 범위 매트릭스 §5**로 일원화됐다. 드로잉 라이브러리 영역의 최종 규칙은 다음과 같다:

| 축 | 값 |
|---|---|
| 엔티티 | `StudentAsset` (drawing/photo/upload/imported 전 kind) |
| 학부모 열람 | 자녀 본인 자산 **전체** — `isSharedToClass` 여부 무관, `isPrivate` 토글 무관 |
| 서버 필터 | `StudentAsset.studentId ∈ parent.children` (parentScopeMiddleware + Prisma where + RLS 3중) |
| 썸네일 | T0-④ 이미지 파이프라인 경유 **presigned URL** — 학부모 URL 추측 공격 차단 |
| 타 학생 자산 | API 응답에서 제거(1차) + DOM 마스킹(보조). 반 공개 자산이어도 학부모 뷰에서는 비노출 |
| 공유 범위 변경 영향 | 없음 — 자녀 본인 자산이면 공유 설정 무관 열람 |
| v1 | 읽기 전용. `AssetAttachment` 역참조 목록 열람 X, 댓글·응원 이모지 X |
| 진입점 | `/parent/child/[id]/drawings` — 스마트폰 PWA 썸네일 그리드 |

구현 작업은 Seed 7의 **PV-7 (자녀 범위 서버 필터)** 및 **PV-6 (/parent/* PWA 쉘)** 에서 처리한다. 본 로드맵의 D-1~D-10 작업은 학부모 뷰 로직을 직접 구현하지 않고, Seed 7의 서버 필터에 데이터 소스만 제공한다.

### 변경 로그
| 날짜 | 시드 ID | 변경 |
|---|---|---|
| 2026-04-12 | `seed_37b35654542f` | §확정 결정 "공유 범위" 행 학부모 관련 문구를 Seed 7 매트릭스 참조로 교체. §파킹 "학부모 포트폴리오 열람"을 "PDF 내보내기만 v2"로 축소. §학부모 열람 범위 절 신규 추가. |
