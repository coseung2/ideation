# Aura-board 식물관찰일지 기능 로드맵

> 작성일: 2026-04-12
> 병렬 문서:
> - Canva 통합 로드맵: `ideation/plans/implementation-roadmap.md`
> - 태블릿 성능 로드맵: `ideation/plans/tablet-performance-roadmap.md`
> - 식물 카탈로그: `ideation/plans/plant-catalog.md`

---

## 0. 기능 요약

교사가 학급에 허용한 식물 리스트에서 학생이 1종 선택 → **지하철 노선도형 로드맵**을 따라 본인 식물의 성장 단계를 기록. 각 단계에 관찰 사진·메모 첨부. 교사는 **데스크톱 매트릭스 뷰**로 반 전체 진행도 스캔.

## 1. 확정된 설계 결정

| 결정 | 내용 | 이유 |
|---|---|---|
| 식물 선택 권한 | **교사 제안형** — 교사가 학급에 허용 리스트 설정, 학생은 그 안에서 선택 | 교과 진도·수업 설계와 일관 |
| 1인 식물 수 | **1식물** | 관찰 집중도, 성능 단순화 |
| 단계 도달 판정 | **학생 자기 선언** | 저마찰, 탐구형 학습 |
| 예상 기간 | **범위(min~max)** | 자연 편차 정직 반영 |
| 관찰 포인트 | **자유 텍스트** (불릿 허용) | 자기 선언 정책과 일관 |
| 참고 이미지 | **빈값 출시**, 교사가 Canva URL로 채움 | 저작권·교사 맞춤 |
| 사진 첨부 | **권장** (미첨부 시 사유 입력) | 교육 가치, 강제 아님 |
| 단계 건너뛰기 | **허용** | 현실 놓침 대응 |
| 과거 단계 수정 | **본인 것만 허용** | 교사 재판단 방지 |
| 단계당 사진 수 | **여러 장 (상한 10장)** | 하루 변화 기록 + 성능 |
| 식물 별명 | **허용** | 애착·식별 |
| 매트릭스 뷰 | **owner + 데스크톱 전용** | editor·viewer·태블릿 제외 |

## 2. 데이터 모델 (Prisma 초안)

```prisma
model PlantSpecies {
  id                String   @id @default(cuid())
  slug              String   @unique
  koreanName        String
  scientificName    String
  category          String
  cotyledon         String   // "쌍떡잎" | "외떡잎"
  totalStages       Int
  growingPeriodMin  Int
  growingPeriodMax  Int
  plantingSeason    String
  growingTips       String
  thumbnailUrl      String?
  stages            PlantStage[]
  classroomAllows   ClassroomPlantAllow[]
  studentPlants     StudentPlant[]
}

model PlantStage {
  id                String   @id @default(cuid())
  speciesId         String
  order             Int
  name              String
  expectedDaysMin   Int
  expectedDaysMax   Int
  observationPoints String
  referenceImageUrl String?

  species      PlantSpecies      @relation(fields: [speciesId], references: [id], onDelete: Cascade)
  observations PlantObservation[]

  @@unique([speciesId, order])
  @@index([speciesId])
}

// 교사가 학급에 허용한 식물 리스트
model ClassroomPlantAllow {
  id           String @id @default(cuid())
  classroomId  String
  speciesId    String

  classroom Classroom    @relation(fields: [classroomId], references: [id], onDelete: Cascade)
  species   PlantSpecies @relation(fields: [speciesId], references: [id], onDelete: Cascade)

  @@unique([classroomId, speciesId])
}

// 학생 1명 = 1 식물 인스턴스
model StudentPlant {
  id              String   @id @default(cuid())
  studentId       String
  classroomId     String
  speciesId       String
  nickname        String?              // 학생 별명
  startedAt       DateTime @default(now())
  currentStageId  String?              // 자기 선언한 현재 단계
  isDead          Boolean  @default(false)
  restartedFrom   String?              // 죽어서 다시 시작 시 이전 instance 참조

  student      Student           @relation(fields: [studentId], references: [id])
  species      PlantSpecies      @relation(fields: [speciesId], references: [id])
  observations PlantObservation[]

  @@index([studentId])
  @@index([classroomId])
}

model PlantObservation {
  id             String   @id @default(cuid())
  studentPlantId String
  stageId        String
  note           String?   @db.Text
  noPhotoReason  String?               // 사진 미첨부 사유
  observedAt     DateTime
  createdAt      DateTime @default(now())
  updatedAt      DateTime @updatedAt

  studentPlant StudentPlant         @relation(fields: [studentPlantId], references: [id], onDelete: Cascade)
  stage        PlantStage           @relation(fields: [stageId], references: [id])
  images       PlantObservationImage[]

  @@index([studentPlantId])
  @@index([stageId])
}

model PlantObservationImage {
  id            String @id @default(cuid())
  observationId String
  url           String
  order         Int    @default(0)

  observation PlantObservation @relation(fields: [observationId], references: [id], onDelete: Cascade)

  @@index([observationId])
}
```

**Board 통합**: `Board.layout`에 `"plant-roadmap"` 추가 — 해당 보드는 Card/Section 대신 위 엔티티를 렌더.

## 3. UX 뷰 3종

### 3.1 학생 뷰 (태블릿 + 데스크톱) — v2 (2026-04-13 업데이트)
- **세로 타임라인** — 좌측 레일에 단계 노드 (상→하), 각 노드 오른쪽에 **관찰 기록 인라인 펼침** (사진 썸네일 + 메모)
- 본인 식물의 현재 단계 노드 강조
- 별도 슬라이드 시트/모달 없이 페이지 내에서 기록 확인·추가
- 하단 "다음 단계로" 버튼 — 현재 단계에 사진 권장, 미첨부 시 사유 입력 모달
- 식물 선택 전 화면: 교사 허용 리스트에서 1종 선택 + 별명 입력

> v1의 가로 지하철 노선도 + `StageDetailSheet` 우측 슬라이드는 폐기. 세로 타임라인이 태블릿 세로 방향(학생 주요 단말)과 잘 맞고, 인라인 펼침이 탭/복원 상호작용을 줄여 태블릿 성능·UX에 유리.

### 3.2 교사 요약 뷰 (태블릿 + 데스크톱)
- 반 전체 학생의 **현재 단계 분포** 요약 (노선도 각 역에 도달자 수 배지)
- 학생별 리스트 (이름, 식물 종, 현 단계, 최근 관찰일)
- 정체된 학생 경고 (허용 기간 초과)

### 3.3 교사 매트릭스 뷰 (**owner + 데스크톱 전용**) — v2부터 **보조 뷰로 격하**
> v2에서 주 네비게이션은 §3.2 요약 뷰의 학생 리스트 클릭 → `/board/[id]/student/[studentId]` 에서 해당 학생의 세로 타임라인을 **교사가 편집 가능한 모드**로 열람. 매트릭스는 전체 현황 한눈 스캔 용도로만 유지.

- 행 = 공통 단계
- 열 = 학생 (별명 + 이름)
- 셀 = 해당 (학생, 단계) 관찰 썸네일
- 셀 클릭 = 확대 모달 (원본 사진 + 메모 + 교사 코멘트 입력)
- 가로 스크롤 + 열 가상화
- 태블릿/editor/viewer 접근 시 403

### 3.4 학부모 뷰 (Seed 7 통합 — v1 구현됨)
> 본 절은 **Seed 7 `seed_37b35654542f`** (`plans/parent-viewer-roadmap.md`)로 통합됐다. v2 예정이 아닌 **v1 초기 배포 범위**에 포함된다. 상세는 Seed 7 §5 자녀 범위 매트릭스 참조.

| 축 | 값 |
|---|---|
| 엔티티 | `StudentPlant` · `PlantObservation` · `PlantObservationImage` |
| 학부모 열람 | 자녀 본인 식물 **전 단계·전 관찰** — `isPrivate` 토글 무관. 교사 코멘트 포함 |
| 서버 필터 | `StudentPlant.studentId ∈ parent.children` → `PlantObservation` 조인 |
| 노선도 | 학생 뷰와 동일한 세로 타임라인, **읽기 전용** (`/parent/child/[id]/plant`) |
| 타 학생 관찰 | 반 공개 관찰이어도 학부모 뷰에서 제외 (서버 1차 + DOM 마스킹 보조) |
| 재시작 식물 | 죽은 이전 식물 아카이브도 열람 가능 (`restartedFrom` 체인) |
| 사진 | T0-④ 이미지 파이프라인 — presigned 썸네일 (< 200KB) |
| v1 | 읽기 전용. 응원 이모지·댓글은 Seed 7과 본 로드맵 모두 v2 파킹 |

## 4. API 초안

| 메서드 | 경로 | 용도 | 권한 |
|---|---|---|---|
| GET | `/api/species` | 전체 식물 카탈로그 | 공개 |
| GET | `/api/species/:id` | 특정 식물 + 단계들 | 공개 |
| GET | `/api/classrooms/:id/species` | 교사 허용 리스트 | 멤버 |
| POST | `/api/classrooms/:id/species` | 교사 허용 리스트 수정 | owner |
| POST | `/api/student-plants` | 학생이 식물 선택 (1인 1식물) | editor |
| PATCH | `/api/student-plants/:id` | 별명·현재 단계 수정 | 본인 |
| POST | `/api/student-plants/:id/restart` | 죽어서 재시작 | 본인 |
| POST | `/api/student-plants/:id/observations` | 관찰 추가 | 본인 |
| PATCH | `/api/observations/:id` | 관찰 수정 (본인만) | 본인 |
| DELETE | `/api/observations/:id` | 관찰 삭제 (본인만) | 본인 |
| GET | `/api/classrooms/:id/matrix` | 매트릭스 데이터 | owner + 데스크톱 |
| POST | `/api/observations/:id/comment` | 교사 코멘트 | owner |

## 5. 태블릿 성능 준수 항목

태블릿 성능 로드맵(`tablet-performance-roadmap.md`)의 모든 원칙 준수. 특히:

- 학생 뷰: 현재 본인 식물 단계 + 전후 단계만 상세 로드
- 교사 요약 뷰: 집계 쿼리(단계별 COUNT)로 N+1 회피
- 매트릭스 뷰: 데스크톱 전용이지만 **열 가상화** 필수 (30학생×10단계)
- 사진: 모든 썸네일 경유, 원본은 모달에서만. T0-④ 이미지 파이프라인 필수
- 동일 단계 사진 상한 10장

## 6. Canva 통합 시너지

| Canva 로드맵 항목 | 연결 |
|---|---|
| P0-① oEmbed | **`PlantStage.referenceImageUrl`**에 교사가 Canva URL 붙이면 라이브 임베드. 관찰 포인트 옆에 교사 도감 자동 노출 |
| P0-② Content Publisher Intent | 학생이 Canva에서 관찰 사진에 **눈금/메모/화살표 덧입힘** → "관찰일지에 게시" → `PlantObservation` 자동 생성 |
| P1-③ Autofill | 교사가 식물별 수업자료 카드(이름·단계 자동 주입) 30장 일괄 생성 |
| P1-④ Quiz 자동 생성 | 식물 한살이 퀴즈를 Canva 슬라이드 → `Quiz`로 자동 변환 |

## 7. 작업 분할 (구현 단위)

| # | 작업 | 의존 | 공수 |
|---|---|---|---|
| PJ-1 | 스키마 마이그레이션 (species/stage/studentPlant/observation) + seed 스크립트로 10종 카탈로그 로드 | - | 2일 |
| PJ-2 | 식물 선택 + 별명 설정 UI (학생) + 교사 허용 리스트 관리 UI | PJ-1 | 2일 |
| PJ-3 | 학생 노선도 뷰 (SVG + 탭 상호작용) | PJ-1 | 3일 |
| PJ-4 | 관찰 추가·수정 (사진 다중 업로드 + 메모 + 미첨부 사유) | PJ-1, T0-④ | 2일 |
| PJ-5 | 교사 요약 뷰 (집계 카드 리스트) | PJ-1 | 2일 |
| PJ-6 | **매트릭스 뷰** (데스크톱 전용, 열 가상화) | PJ-1, PJ-4 | 3일 |
| PJ-7 | Canva oEmbed 참고 이미지 붙이기 (P0-① 완료 후) | P0-① | 1일 |
| PJ-8 | 학부모 뷰 — **Seed 7 PV-7로 이관 완료** (`parent-viewer-roadmap.md`) | Seed 7 PV-1~PV-7 | - |

**1차 배포 범위**: PJ-1 ~ PJ-6 (2.5주 예상)

## 8. 수용 기준 (핵심)

- [ ] 학생이 교사 허용 리스트에서 식물 선택 후 별명 입력 → 본인 노선도 생성
- [ ] 단계 카드에 사진 여러 장(≤10) 업로드 + 메모 작성
- [ ] "다음 단계로" 선언 시 사진 없으면 사유 입력 프롬프트
- [ ] 이전 단계 본인 관찰만 수정/삭제 가능, 타 학생 불가
- [ ] 단계 건너뛰기 허용 (순서 강제 없음)
- [ ] 식물 죽으면 "다시 시작" 버튼으로 새 `StudentPlant` 생성 (과거 기록 보존)
- [ ] 교사 요약 뷰에서 반 전체 분포 한눈에
- [ ] 교사 매트릭스 뷰: editor/viewer/태블릿 모두 403
- [ ] 태블릿 성능 예산 준수 (학생 뷰 TTI < 3s)

## 9. 리스크 & 미결

| 항목 | 이슈 |
|---|---|
| 사진 저장소 | 이미지 파이프라인 T0-④ 결정 선행 필수 (Vercel Blob 권장) |
| 지역별 파종기 차이 | 카탈로그는 중부 기준 — 남부·북부 조정은 v2 |
| 교사 커스텀 단계 | 10종 외 교사 직접 추가? v2 |
| ~~학부모 인증~~ | **해소 — Seed 7 (`parent-viewer-roadmap.md`)**. Parent·ParentChildLink·ParentInviteCode·ParentSession 4종 모델 + 매직 링크 인증 + Crockford Base32 코드로 확정 |
| 지도 확대·축소 | 노선도 SVG 확대/축소 제스처는 PJ-3 내에서 결정 |

## 10. padlet 파이프라인 연결

각 PJ-# 작업을 padlet `feature` 파이프라인 task로 올릴 때 phase0 request 예시:

```json
{
  "type": "feature",
  "title": "식물관찰일지 — 학생 노선도 뷰 (PJ-3)",
  "goal": "학생이 본인 식물의 단계 진행을 지하철 노선도로 보고 상호작용",
  "context_ref": "ideation/plans/plant-journal-roadmap.md#pj-3",
  "catalog_ref": "ideation/plans/plant-catalog.md",
  "performance_budget_ref": "ideation/plans/tablet-performance-roadmap.md#2-성능-예산"
}
```

---

## 11. 변경 로그

| 날짜 | 시드 ID | 변경 |
|---|---|---|
| 2026-04-12 | `seed_37b35654542f` (Seed 7) | §3.4 학부모 뷰 v2 예정 → **v1 Seed 7 PV-7 이관**. PJ-8 작업을 Seed 7로 이관 완료 표시. 미결 "학부모 인증"은 Parent·ParentChildLink·ParentInviteCode·ParentSession 4종 + 매직 링크 + Crockford Base32 코드로 확정 해소. |
