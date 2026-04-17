# padlet feature 파이프라인 진입용 phase0 요청 템플릿

> 각 작업을 padlet `prompts/feature/_index.md` 파이프라인에 올릴 때 `tasks/{YYYY-MM-DD-slug}/phase0/request.json` 에 넣을 초안 모음.
> 컨텍스트 레퍼런스가 핵심 — planner(phase1)가 이 문서를 읽고 스코프를 결정.
> 사용법: 필요한 블록을 복사 → `request.json`으로 저장 → 상위 `padlet/` 쪽에서 파이프라인 시작.

---

## Canva 통합 (implementation-roadmap.md)

### P0-① oEmbed 라이브 Canva 카드
```json
{
  "type": "feature",
  "title": "Canva oEmbed 라이브 카드",
  "goal": "카드에 Canva 디자인 URL 붙이면 라이브 iframe 임베드 + 원본 수정 시 자동 반영",
  "context_ref": "ideation/plans/implementation-roadmap.md#p0-①-oembed-라이브-canva-카드",
  "performance_budget_ref": "ideation/plans/tablet-performance-roadmap.md#2-성능-예산",
  "blocking_on": ["T0-② iframe 가상화 선행"],
  "acceptance": [
    "canva.com/design/... URL 입력 시 3초 내 임베드 렌더",
    "원본 변경 30초 내 자동 반영",
    "viewer 역할도 보기 가능, 편집 불가",
    "비공개 디자인은 폴백 카드 표시",
    "보드당 동시 활성 iframe ≤ 3 (LRU)"
  ]
}
```

### P0-② Content Publisher Intent (역방향)
```json
{
  "type": "feature",
  "title": "Canva → Aura-board Content Publisher 앱",
  "goal": "Canva 에디터에서 \"Aura-board에 게시\" 클릭 → 교실·보드 선택 → 카드/제출 자동 생성",
  "context_ref": "ideation/plans/implementation-roadmap.md#p0-②-content-publisher-intent-앱-canva--aura-board",
  "deliverables": [
    "ideation/content-publisher-app/ (별도 Canva 앱)",
    "padlet: POST /api/external/cards 수신 엔드포인트 + 교사 PAT 인증"
  ],
  "acceptance": [
    "교사가 앱 연결 시 본인 교실 목록 표시",
    "학생 디자인 완성 → 게시 → 10초 내 카드 생성",
    "업로드 실패 시 Canva Toast 오류",
    "viewer 학생은 게시 버튼 비활성화"
  ]
}
```

### P1-③ Autofill 대량 생성
```json
{
  "type": "feature",
  "title": "Canva Autofill — 교사 대량 생성",
  "goal": "교사가 브랜드 템플릿 + 학생 명단 선택 → 수료증/이름표 일괄 생성 → 보드 카드 자동 배포",
  "context_ref": "ideation/plans/implementation-roadmap.md#p1-③-autofill--교사-대량-생성",
  "acceptance": [
    "교사 UI에서 Canva 템플릿 선택",
    "Classroom 학생 명단 자동 주입",
    "30장 기준 2분 내 완료 + 진행률 표시",
    "결과물이 지정 섹션에 일괄 배포"
  ]
}
```

### P1-④ Quiz 자동 생성
```json
{
  "type": "feature",
  "title": "Canva 슬라이드 → Quiz 자동 생성",
  "goal": "교사가 Canva 발표자료 URL 붙이면 텍스트·구조 추출 → QuizQuestion 자동 시드",
  "context_ref": "ideation/plans/implementation-roadmap.md#p1-④-quiz-자동-생성-canva-슬라이드--quizquestion",
  "acceptance": [
    "10페이지 이내 슬라이드 변환 가능",
    "파싱 실패 항목은 수동 편집 폴백",
    "저장 전 교사 확인·수정 가능"
  ]
}
```

### P2-⑤ Webhook 카드 refresh
```json
{
  "type": "feature",
  "title": "Canva Webhook — 카드 자동 refresh",
  "goal": "Canva 디자인 변경 이벤트 수신 → 보드 카드 썸네일·메타 갱신",
  "context_ref": "ideation/plans/implementation-roadmap.md#p2-⑤-webhook--카드-자동-refresh",
  "acceptance": [
    "Canva 수정 후 1분 내 참조 카드 갱신",
    "Webhook 서명 검증",
    "실패 이벤트 재시도 큐"
  ]
}
```

### P2-⑥ LMS LTI
```json
{
  "type": "feature",
  "title": "LMS LTI 1.3 provider",
  "goal": "Aura-board를 LTI provider로 등록 → Google Classroom·Canvas에서 학습 도구로 연결",
  "context_ref": "ideation/plans/implementation-roadmap.md#p2-⑥-lms-lti-어댑터",
  "acceptance": [
    "Canvas LMS에서 Aura-board 추가 가능",
    "LTI deep linking으로 특정 보드 지정",
    "Grade passback (Submission.grade → LMS)"
  ]
}
```

---

## 태블릿 성능 (tablet-performance-roadmap.md)

### T0-① 섹션 격리 Breakout 뷰
```json
{
  "type": "feature",
  "title": "섹션 격리 Breakout 뷰 (Padlet 대안)",
  "goal": "모둠 링크로 진입 시 해당 섹션만 로드. 다른 섹션 이벤트·DOM 격리",
  "context_ref": "ideation/plans/tablet-performance-roadmap.md#5-별도-작업-t0-①-섹션-격리-breakout-뷰-성능-우선",
  "acceptance": [
    "섹션 링크 진입 시 해당 섹션 카드만 수신",
    "iPad 9th 30명 동시, 카드 50장 TTI < 3s",
    "섹션 전환 시 이전 섹션 DOM/iframe 완전 언마운트",
    "다른 섹션 변경 이벤트 WebSocket 미도달",
    "교사 뷰는 통합 요약 + 섹션별 드릴다운"
  ]
}
```

### T0-② iframe 가상화 + 라이브 모드 토글
```json
{
  "type": "feature",
  "title": "iframe 가상화 (Canva oEmbed 선행 요건)",
  "goal": "다수 Canva iframe 동시 마운트 방지. 뷰포트·LRU 기반 활성 제어",
  "context_ref": "ideation/plans/tablet-performance-roadmap.md#6-별도-작업-t0-②-iframe-가상화--라이브-모드-토글",
  "acceptance": [
    "기본은 썸네일 + 재생 아이콘 오버레이",
    "탭 시에만 iframe 활성화",
    "보드 전체 동시 활성 iframe ≤ 3 (LRU 언마운트)",
    "뷰포트 밖 카드 iframe 즉시 언마운트",
    "카드 상단 '라이브' 토글 상태 표시"
  ]
}
```

### T0-③ WebSocket 페이로드 최적화
```json
{
  "type": "feature",
  "title": "WebSocket 경량 프로토콜",
  "goal": "섹션 채널 분리 + 델타 페이로드 + 디바운싱",
  "context_ref": "ideation/plans/tablet-performance-roadmap.md#7-별도-작업-t0-③-websocket-페이로드-최적화",
  "blocking_on": ["research: 실시간 엔진 결정 (Liveblocks/Yjs/자체 WS)"],
  "acceptance": [
    "카드 이동 메시지 < 200 바이트",
    "100ms 디바운싱",
    "섹션 채널 분리로 관련 없는 이벤트 0%",
    "서버→클라 메시지에 불필요 전체 객체 금지"
  ]
}
```

### T0-④ 이미지·썸네일 파이프라인
```json
{
  "type": "feature",
  "title": "이미지 파이프라인 (적응 해상도 + lazy)",
  "goal": "원본 응답 금지, 썸네일 경유, 뷰포트 lazy. Vercel Image Optimization 또는 동급.",
  "context_ref": "ideation/plans/tablet-performance-roadmap.md#8-별도-작업-t0-④-이미지썸네일-파이프라인",
  "acceptance": [
    "원본 해상도 서버 응답 금지",
    "뷰포트 진입 전 이미지 미로드",
    "srcset DPR 대응",
    "3G 에뮬레이션에서 카드 썸네일 < 500KB/장"
  ]
}
```

### 선행 research: 실시간 엔진 벤치
```json
{
  "type": "research",
  "title": "실시간 동기화 엔진 벤치 (Liveblocks vs Yjs vs 자체 WS)",
  "goal": "iPad 9th 30명 동시 접속 기준 태블릿 성능·비용·채널 격리 구현 난이도 평가 → ADOPT/REVISE/REJECT",
  "context_ref": "ideation/plans/tablet-performance-roadmap.md#9-선행-research-task-실시간-엔진-결정",
  "evaluation_axes": ["태블릿 성능", "채널 격리 구현 난이도", "30명 확장성", "외부 의존성"],
  "acceptance": [
    "실측 벤치(프로토타입) 데이터 존재",
    "3개 차원 정량 비교표",
    "decision.md에 ADOPT/REVISE/REJECT 판정"
  ]
}
```

---

## 식물관찰일지 (plant-journal-roadmap.md)

### PJ-1 스키마 + seed
```json
{
  "type": "feature",
  "title": "식물관찰일지 — 스키마 + 10종 seed",
  "goal": "PlantSpecies/PlantStage/ClassroomPlantAllow/StudentPlant/PlantObservation/PlantObservationImage 추가 + 카탈로그 10종 seed",
  "context_ref": "ideation/plans/plant-journal-roadmap.md#2-데이터-모델-prisma-초안",
  "seed_data_ref": "ideation/data/plant-species-seed.json",
  "acceptance": [
    "prisma migrate 성공",
    "npm run seed 실행 시 10종 + 단계들 전체 로드 (멱등)",
    "나팔꽃 slug='morning_glory' 존재 검증"
  ]
}
```

### PJ-2 식물 선택 + 허용 리스트 관리
```json
{
  "type": "feature",
  "title": "식물관찰일지 — 식물 선택 (학생) + 허용 리스트 (교사)",
  "goal": "교사가 학급 허용 식물 지정, 학생이 1종 선택 + 별명 부여",
  "context_ref": "ideation/plans/plant-journal-roadmap.md#4-api-초안",
  "blocking_on": ["PJ-1"],
  "acceptance": [
    "교사: /api/classrooms/:id/species POST로 허용 리스트 관리",
    "학생: 허용 리스트에서 1종 선택 시 StudentPlant 생성 (1인 1식물 유니크 제약)",
    "별명 입력 가능"
  ]
}
```

### PJ-3 학생 노선도 뷰
```json
{
  "type": "feature",
  "title": "식물관찰일지 — 학생 노선도 뷰 (지하철 스타일)",
  "goal": "학생이 본인 식물 단계 진행을 노선도 UI로 보고 상호작용",
  "context_ref": "ideation/plans/plant-journal-roadmap.md#31-학생-뷰-태블릿--데스크톱",
  "blocking_on": ["PJ-1", "PJ-2"],
  "acceptance": [
    "가로 SVG 노선도, 현재 마커 강조",
    "단계 탭 시 관찰 포인트 + 본인 사진/메모 표시",
    "단계 건너뛰기 허용",
    "iPad 9th TTI < 3s"
  ]
}
```

### PJ-4 관찰 추가·수정 (사진 + 메모)
```json
{
  "type": "feature",
  "title": "식물관찰일지 — 관찰 CRUD (사진·메모)",
  "goal": "학생이 단계에 사진(≤10) + 메모 추가. 본인 것만 수정·삭제.",
  "context_ref": "ideation/plans/plant-journal-roadmap.md#4-api-초안",
  "blocking_on": ["PJ-1", "T0-④"],
  "acceptance": [
    "다중 사진 업로드 (상한 10)",
    "'다음 단계로' 선언 시 사진 없으면 사유 입력",
    "본인 관찰만 수정·삭제, 타 학생 불가"
  ]
}
```

### PJ-5 교사 요약 뷰
```json
{
  "type": "feature",
  "title": "식물관찰일지 — 교사 요약 뷰 (반 전체 분포)",
  "goal": "반 전체 학생의 현재 단계 분포 + 정체 학생 경고",
  "context_ref": "ideation/plans/plant-journal-roadmap.md#32-교사-요약-뷰-태블릿--데스크톱",
  "blocking_on": ["PJ-1"],
  "acceptance": [
    "노선도 각 역에 도달자 수 배지",
    "학생별 리스트(이름/식물/현 단계/최근 관찰일)",
    "허용 기간 초과 학생 경고"
  ]
}
```

### PJ-6 매트릭스 뷰 (owner + 데스크톱 전용)
```json
{
  "type": "feature",
  "title": "식물관찰일지 — 교사 매트릭스 뷰 (데스크톱 전용)",
  "goal": "행=단계, 열=학생, 셀=관찰 썸네일. 셀 클릭 = 확대 + 교사 코멘트.",
  "context_ref": "ideation/plans/plant-journal-roadmap.md#33-교사-매트릭스-뷰-owner--데스크톱-전용",
  "blocking_on": ["PJ-1", "PJ-4"],
  "guard_rails": [
    "editor/viewer/태블릿 접근 시 403",
    "권한 2중 검증 (RBAC + UA/viewport)"
  ],
  "acceptance": [
    "매트릭스 렌더 및 열 가상화",
    "셀 클릭 시 원본 확대 + 코멘트 입력",
    "권한 게이트 통과 검증"
  ]
}
```

### PJ-7 Canva oEmbed 참고 이미지 (P0-① 의존)
```json
{
  "type": "feature",
  "title": "식물관찰일지 — 단계별 참고 이미지 Canva 연결",
  "goal": "교사가 PlantStage.referenceImageUrl에 Canva URL 붙이면 라이브 임베드로 노출",
  "context_ref": "ideation/plans/plant-journal-roadmap.md#6-canva-통합-시너지",
  "blocking_on": ["P0-①", "PJ-1"],
  "acceptance": [
    "교사가 URL 붙이기 UI 존재",
    "학생 노선도에서 참고 이미지가 라이브 임베드로 노출",
    "URL 비어있으면 플레이스홀더"
  ]
}
```

---

## Breakout Room 보드 (breakout-room-roadmap.md)

### BR-1 스키마 마이그레이션 (BreakoutTemplate · BreakoutAssignment · Membership)
```json
{
  "type": "feature",
  "title": "Breakout Room — 스키마 + Section role/accessToken 확장",
  "goal": "BreakoutTemplate·BreakoutAssignment·BreakoutMembership 3개 모델 추가, Section.role/accessToken 확장. BreakoutGroup 신설 금지 (Section 재활용)",
  "context_refs": [
    "ideation/plans/breakout-room-roadmap.md#2-데이터-모델-prisma-초안",
    "ideation/plans/tablet-performance-roadmap.md#5-별도-작업-t0-①-섹션-격리-breakout-뷰-성능-우선"
  ],
  "blocking_on": ["T0-① 섹션 격리 뷰 (Section.accessToken 필드 선행)"],
  "acceptance": [
    "prisma migrate 성공",
    "BreakoutTemplate 필드: id·name·tier·structure·recommendedVisibility·defaultGroupCount·defaultGroupCapacity·maxGroupCount·version",
    "BreakoutAssignment 필드: id·boardId·templateId·deployMode·groupCount·groupCapacity·visibilityOverride·status·isPublic",
    "BreakoutMembership @@unique([sectionId, studentId])",
    "Section.accessToken 중복 마이그레이션 없음 (T0-① 이미 생성)"
  ]
}
```

### BR-2 시스템 템플릿 8종 seed
```json
{
  "type": "feature",
  "title": "Breakout Room — 시스템 템플릿 8종 seed",
  "goal": "KWL·브레인스토밍·아이스브레이커(Free 3) + 찬반 토론·Jigsaw·모둠 발표 준비·갤러리 워크·6색 모자(Pro 5) 총 8종 등록. 월드카페는 v2 파킹.",
  "context_refs": [
    "ideation/plans/breakout-room-roadmap.md#16-v1-템플릿-카탈로그-71종-v2-파킹-1종"
  ],
  "seed_data_ref": "ideation/data/breakout-template-seed.json",
  "blocking_on": ["BR-1"],
  "acceptance": [
    "npm run seed:breakout 멱등 실행",
    "recommendedVisibility own-only 5종·peek-others 2종 정확 매핑",
    "Free 3종은 requiresPro=false, 나머지 5종 requiresPro=true",
    "각 템플릿 structure.sectionsPerGroup[]에 섹션 스펙 존재"
  ]
}
```

### BR-3 교사 개설 플로우 UI (템플릿 피커 + 옵션 선택)
```json
{
  "type": "feature",
  "title": "Breakout Room — 교사 개설 플로우",
  "goal": "템플릿 피커 → 모둠 수/정원 → deployMode/visibilityOverride 선택 → Draft BreakoutAssignment 생성",
  "context_refs": [
    "ideation/plans/breakout-room-roadmap.md#3-작업-분할-br-1--br-9",
    "ideation/plans/drawing-board-library-roadmap.md"
  ],
  "blocking_on": ["BR-1", "BR-2"],
  "notes": "drawing-board-library의 '템플릿 고르기' 모달 패턴 재사용 검토 — 공통 컴포넌트 추출",
  "acceptance": [
    "Free 교사에게 Pro 템플릿 5종은 잠금 배지 + 업그레이드 CTA 표시",
    "기본값 4모둠/정원 6/상한 10모둠 자동 적용",
    "deployMode 3종·visibility 2종 드롭다운 동작",
    "Draft 상태로 저장 후 '개설' 버튼으로 live 전환"
  ]
}
```

### BR-4 모둠 섹션 복제 엔진
```json
{
  "type": "feature",
  "title": "Breakout Room — 모둠 섹션 복제 엔진",
  "goal": "template.structure.sectionsPerGroup[]를 groupCount만큼 복제해 Section INSERT + teacher-pool은 보드 레벨 단일 섹션으로 1회만 생성",
  "context_refs": [
    "ideation/plans/breakout-room-roadmap.md#14-복제-방식--복사독립",
    "ideation/plans/drawing-board-library-roadmap.md"
  ],
  "blocking_on": ["BR-3"],
  "acceptance": [
    "N모둠 × 섹션스펙M개 = Section N×M개 + teacher-pool 1개 생성",
    "각 Section에 role·groupIndex 설정",
    "카드는 섹션별 독립 복사본 (1모둠 수정이 2모둠 전파 X)",
    "BreakoutTemplate.structure 수정이 기존 파생 Board에 역전파되지 않음"
  ]
}
```

### BR-5 배포모드 3종 구현
```json
{
  "type": "feature",
  "title": "Breakout Room — deployMode 3종 (link-fixed/self-select/teacher-assign)",
  "goal": "link-fixed(accessToken 발급) · self-select(학생 초기 1회 선택 + capacity soft limit) · teacher-assign(교사 드래그 배정)",
  "context_refs": [
    "ideation/plans/breakout-room-roadmap.md#12-배포모드-3종--열람모드-2종",
    "ideation/plans/tablet-performance-roadmap.md#5-별도-작업-t0-①-섹션-격리-breakout-뷰-성능-우선"
  ],
  "blocking_on": ["BR-4", "T0-①"],
  "acceptance": [
    "link-fixed: 모둠별 고유 URL(accessToken) 발급 + 링크 진입 시 해당 섹션만 로드",
    "self-select: 초기 선택 후 변경은 교사 승인 필요",
    "teacher-assign: 교사 UI에서 드래그 배정 + 학생 신고 버튼 제공",
    "모든 모드에서 Card.sectionId 고정 (학생 이동 시에도 이전 카드 원래 섹션 유지)"
  ]
}
```

### BR-6 열람모드 2종 + 교사 전체 접근 RBAC
```json
{
  "type": "feature",
  "title": "Breakout Room — visibility own-only/peek-others + 교사 상위 접근",
  "goal": "학생 기준 own-only/peek-others를 viewSection 권한에 매핑. 교사는 모든 템플릿·모드에서 전 모둠 전체 접근 보장(상위 집합, 직교 축)",
  "context_refs": [
    "ideation/plans/breakout-room-roadmap.md#12-배포모드-3종--열람모드-2종",
    "ideation/plans/tablet-performance-roadmap.md#5-별도-작업-t0-①-섹션-격리-breakout-뷰-성능-우선"
  ],
  "blocking_on": ["BR-4", "T0-①"],
  "acceptance": [
    "own-only 학생은 본인 섹션만 WS 이벤트 수신",
    "peek-others 학생은 전 모둠 섹션 읽기 가능",
    "교사는 두 모드 모두에서 전체 쓰기 가능",
    "visibilityOverride가 템플릿 recommendedVisibility를 교사 선택으로 덮어씀"
  ]
}
```

### BR-7 "모든 모둠에 이 카드 복제" 단발 액션
```json
{
  "type": "feature",
  "title": "Breakout Room — '전 모둠 일괄 배포' 단발 액션 버튼",
  "goal": "교사가 교사-pool 또는 임의 모둠 카드에서 '모든 모둠에 복제' 클릭 → 확인 모달 → N모둠 섹션에 INSERT 반복. 단발 액션(자동 동기화 X)",
  "context_refs": [
    "ideation/plans/breakout-room-roadmap.md#14-복제-방식--복사독립"
  ],
  "blocking_on": ["BR-4"],
  "acceptance": [
    "확인 모달에 대상 모둠 수 명시",
    "서버 트랜잭션 또는 결과 보고(성공/실패 모둠)",
    "복제 후 대상 섹션에서 독립 편집 가능 (원본 수정 시 전파 X)",
    "되돌리기는 v2 파킹 (본 iteration 미포함)"
  ]
}
```

### BR-8 교사 커스텀 템플릿 저장 (Free 3개 / Pro 무제한)
```json
{
  "type": "feature",
  "title": "Breakout Room — 커스텀 템플릿 저장 플로우",
  "goal": "기존 Board → '템플릿으로 저장' → 구조 JSON 직렬화 → BreakoutTemplate scope=teacher 저장. Free 3개 상한, Pro 무제한",
  "context_refs": [
    "ideation/plans/breakout-room-roadmap.md#15-tier-매트릭스"
  ],
  "blocking_on": ["BR-2", "BR-3"],
  "acceptance": [
    "Free 4번째 저장 시도 시 업그레이드 CTA 모달",
    "구조 JSON에 섹션 role·카드 seed·권장 visibility 포함",
    "저장된 커스텀 템플릿은 본인 템플릿 피커에만 표시"
  ]
}
```

### BR-9 학교 공용 템플릿 등록 (Pro 전용)
```json
{
  "type": "feature",
  "title": "Breakout Room — 학교 공용 템플릿 (scope=school)",
  "goal": "Pro 교사가 커스텀 템플릿을 학교 공용으로 승격 → 학교 관리자 승인 → 같은 학교 교사에게 노출",
  "context_refs": [
    "ideation/plans/breakout-room-roadmap.md#15-tier-매트릭스"
  ],
  "blocking_on": ["BR-8"],
  "acceptance": [
    "Pro 전용 게이팅 (Free 교사에게 옵션 숨김)",
    "관리자 승인 큐 UI",
    "승인된 템플릿은 학교 소속 교사 피커에 별도 섹션으로 노출"
  ]
}
```

---

## 학부모 읽기 전용 뷰어 액세스 (parent-viewer-roadmap.md)

### PV-1 스키마 마이그레이션 + RLS 정책
```json
{
  "type": "feature",
  "title": "Parent Viewer — 스키마 4종 + RLS",
  "goal": "Parent·ParentChildLink·ParentInviteCode·ParentSession 4종 Prisma 모델 추가 + BoardMember.role 유니언에 \"parent\" 추가 + RLS 정책(parent_id=auth.parent_id() 단방향)",
  "context_refs": [
    "ideation/plans/parent-viewer-roadmap.md#2-데이터-모델-prisma-최종-확정",
    "ideation/plans/seeds-index.md#-seed-7-학부모-읽기-전용-뷰어-액세스-2026-04-12-추가"
  ],
  "acceptance": [
    "prisma migrate 성공",
    "Parent·ParentChildLink·ParentInviteCode·ParentSession 4종 필드 완전 반영 (emailSummaryOptOut·deletedAt·failedAttempts·revokedAt 포함)",
    "BoardMember.role 유니언에 \"parent\" 포함 (read-only)",
    "RLS: ParentChildLink SELECT는 parent_id=auth.parent_id() 단방향만 허용",
    "ParentChildLink @@unique([parentId, studentId])"
  ]
}
```

### PV-2 Crockford Base32 코드 생성 + 교사 발급 UI + QR
```json
{
  "type": "feature",
  "title": "Parent Viewer — 코드 생성기 + 학생 카드 '학부모 초대' + QR",
  "goal": "crypto.randomBytes 기반 CSPRNG로 Crockford Base32 6자리 코드 생성(O/0·I/1·L 제외, 대문자 고정). 교사가 학생 카드 드롭다운 '학부모 초대' → 코드 + QR 모달(복사/공유)",
  "context_refs": [
    "ideation/plans/parent-viewer-roadmap.md#31-교사-발급--학부모-등록"
  ],
  "blocking_on": ["PV-1"],
  "acceptance": [
    "코드 32^6 ≈ 10^9 조합, Math.random 금지 (crypto.randomBytes)",
    "O/0·I/1·L 제외 대문자 고정 (알파벳 집합 검증)",
    "POST /api/parent-invite-codes { studentId } → expiresAt=now+48h, maxUses=3",
    "학생 카드 드롭다운 '학부모 초대' 메뉴 진입",
    "모달에 6자리 코드 + QR PNG/SVG + 공유 CTA",
    "만료/소진 후 동일 학생 카드에서 재발급 버튼 노출"
  ]
}
```

### PV-3 /api/parent/redeem + rate limit + 매직 링크 발송
```json
{
  "type": "feature",
  "title": "Parent Viewer — 코드 교환 + 이중 rate limit + 매직 링크",
  "goal": "학부모 /parent/enter에서 코드+이메일 제출 → 검증 → 매직 링크 이메일 발송(React Email + Resend, 15분 유효)",
  "context_refs": [
    "ideation/plans/parent-viewer-roadmap.md#31-교사-발급--학부모-등록",
    "ideation/plans/parent-viewer-roadmap.md#12-revoke--격리"
  ],
  "blocking_on": ["PV-1", "PV-2"],
  "acceptance": [
    "IP당 실패 5회/15분 초과 시 429 + 15분 잠금 (Upstash Redis)",
    "코드당 failedAttempts++ 10 도달 시 expiresAt=now() 즉시 만료",
    "성공 시 ParentChildLink UPSERT + ParentInviteCode.usedCount++",
    "매직 링크 토큰 유효 15분, 1회 소비",
    "이메일 발송은 Resend + React Email 렌더",
    "응답에 코드 정확성 유출 없음(일반 오류 메시지)"
  ]
}
```

### PV-4 매직 링크 검증 + 세션 발급
```json
{
  "type": "feature",
  "title": "Parent Viewer — 매직 링크 verify + ParentSession 발급",
  "goal": "GET /parent/verify?token=... → 토큰 검증 → ParentSession INSERT(expiresAt=now+7d) → httpOnly 쿠키 세팅 → /parent/home 리다이렉트",
  "context_refs": [
    "ideation/plans/parent-viewer-roadmap.md#11-페어링--인증"
  ],
  "blocking_on": ["PV-3"],
  "acceptance": [
    "토큰 15분 만료 후 접근 시 재발송 CTA",
    "ParentSession.token은 CSPRNG, 쿠키는 httpOnly + Secure + SameSite=Lax",
    "세션 만료(7일) 후 재방문 시 이메일만으로 재인증 (코드 재입력 불필요)",
    "동일 이메일 재가입 시 기존 ParentChildLink 재사용 (중복 Parent 생성 금지)"
  ]
}
```

### PV-5 parentScopeMiddleware + 403/404 분기
```json
{
  "type": "feature",
  "title": "Parent Viewer — parentScopeMiddleware (핵심 보안 게이트)",
  "goal": "모든 /parent/* API 엔드포인트에 미들웨어 적용. session.revokedAt/expiresAt/Parent.deletedAt 검증 + studentId 귀속 검증. 403(타 학생)·404(타 학부모)·401(세션 무효) 분기",
  "context_refs": [
    "ideation/plans/parent-viewer-roadmap.md#4-parentscopemiddleware-설계",
    "ideation/plans/parent-viewer-roadmap.md#12-revoke--격리"
  ],
  "blocking_on": ["PV-1"],
  "acceptance": [
    "세션 revokedAt IS NOT NULL 또는 Parent.deletedAt IS NOT NULL → 401",
    "studentId 포함 요청: ParentChildLink active 검증 실패 시 403",
    "타 Parent의 ParentChildLink 조회 시도 → 404 (존재 불인식)",
    "RLS auth.parent_id() 세팅 이중 방어",
    "ESLint 룰: /parent/* API 핸들러는 반드시 parentScope로 래핑"
  ]
}
```

### PV-6 /parent/* PWA 쉘 + 홈 + child 라우트
```json
{
  "type": "feature",
  "title": "Parent Viewer — PWA 쉘 (/parent/* 라우트 + manifest)",
  "goal": "스마트폰 포트레이트 PWA. /parent/home(자녀 N 카드 최대 5) + /parent/child/[id]/(drawings|plant|events|breakout|homework) + manifest.json + SWR 60s 폴링",
  "context_refs": [
    "ideation/plans/parent-viewer-roadmap.md#6-parent-pwa-구조"
  ],
  "blocking_on": ["PV-5"],
  "acceptance": [
    "iOS Safari·Android Chrome에서 PWA 설치 가능 (manifest.json + apple-touch-icon)",
    "320~430px 포트레이트 레이아웃 (Tailwind breakpoint)",
    "TTI < 2s LTE / < 3s 3G 에뮬레이션",
    "첫 뷰포트 < 500KB, 썸네일 < 200KB",
    "iframe 마운트 0건 (DOM snapshot E2E)",
    "SWR 60s 폴링, WebSocket 비활성",
    "390px 세로 Lighthouse Mobile ≥ 90"
  ]
}
```

### PV-7 자녀 범위 서버 필터 (§5 매트릭스 일괄 적용)
```json
{
  "type": "feature",
  "title": "Parent Viewer — 자녀 범위 서버 필터 (cross-cutting)",
  "goal": "StudentAsset·PlantObservation·EventSignup·BreakoutMembership·Submission 각 /parent/* 전용 API에 자녀 범위 매트릭스(Seed 7 single source of truth) 서버 필터링 적용",
  "context_refs": [
    "ideation/plans/parent-viewer-roadmap.md#5-cross-cutting-자녀-범위-매트릭스--single-source-of-truth",
    "ideation/plans/drawing-board-library-roadmap.md",
    "ideation/plans/plant-journal-roadmap.md",
    "ideation/plans/event-signup-roadmap.md",
    "ideation/plans/breakout-room-roadmap.md"
  ],
  "blocking_on": ["PV-5", "PV-6"],
  "acceptance": [
    "StudentAsset: studentId ∈ parent.children (isSharedToClass·isPrivate 무관)",
    "PlantObservation: StudentPlant.studentId ∈ parent.children + 교사 코멘트 포함",
    "EventBoard: classroomId=child.classroomId 허용하되 EventSignup 응답에서 studentId ≠ child 제거",
    "BreakoutMembership: session.studentId ∈ parent.children, teacher-pool 섹션 제외",
    "Submission: userId=child.userId 또는 studentId ∈ parent.children",
    "Quiz 점수 v1 비노출",
    "썸네일은 presigned URL 또는 RLS-scoped query (URL 추측 차단)",
    "타 학부모 식별 정보 응답 미포함"
  ]
}
```

### PV-8 교사 관리 UI — 학부모 액세스 탭
```json
{
  "type": "feature",
  "title": "Parent Viewer — 교사 학급 설정 '학부모 액세스' 탭",
  "goal": "전체 ParentChildLink 리스트(학부모명·이메일·마지막 접속일·발급 코드 상태) + 1-click revoke 모달('최대 1분 내 차단' 문구)",
  "context_refs": [
    "ideation/plans/parent-viewer-roadmap.md#32-revoke-흐름"
  ],
  "blocking_on": ["PV-1"],
  "acceptance": [
    "학급 설정 내 '학부모 액세스' 탭 존재",
    "전체 ParentChildLink 리스트 표시 (교사만 학부모 이름·이메일 노출)",
    "1-click revoke 버튼 + 확인 모달 '최대 1분 내 차단됩니다'",
    "탈퇴 학부모는 '연결 해제됨 (탈퇴)' 라벨, 이름·이메일 비표시",
    "revokedAt·revokedReason 감사 필드 SET"
  ]
}
```

### PV-9 Revoke SLA ≤ 60s + 클라이언트 자동 로그아웃
```json
{
  "type": "feature",
  "title": "Parent Viewer — Revoke ≤60s + 401 자동 로그아웃",
  "goal": "교사 revoke → ParentSession.revokedAt 일괄 SET → SWR 60s 폴링에서 401 수신 → 클라이언트 자동 로그아웃 + '접근이 해제되었습니다' 화면 전환",
  "context_refs": [
    "ideation/plans/parent-viewer-roadmap.md#12-revoke--격리",
    "ideation/plans/parent-viewer-roadmap.md#32-revoke-흐름"
  ],
  "blocking_on": ["PV-5", "PV-8"],
  "acceptance": [
    "revoke 시점 이후 60초 이내 학부모 세션 401 수신 (E2E 측정)",
    "401 수신 시 쿠키 정리 + /parent/revoked 화면 전환",
    "Redis 블랙리스트 기반 즉시 revoke(<1s)는 v2 파킹 명시"
  ]
}
```

### PV-10 주간 이메일 요약 (Pro 전용) + Vercel Cron
```json
{
  "type": "feature",
  "title": "Parent Viewer — 주간 이메일 요약 (Pro 전용)",
  "goal": "Vercel Cron 매주 월 00:00 UTC(=KST 09:00) → 자녀별 집계 + 대표 썸네일(presigned 7일) + 교사 피드백 1~3 bullet + CTA 딥링크. 활동 0건 주 발송 스킵",
  "context_refs": [
    "ideation/plans/parent-viewer-roadmap.md#13-알림--tier"
  ],
  "blocking_on": ["PV-7"],
  "acceptance": [
    "Pro 학부모에게만 발송 (Free는 스킵)",
    "emailSummaryOptOut=true면 스킵",
    "활동 0건 주는 lastSummarySkippedAt 기록 후 스킵",
    "React Email + Resend 렌더",
    "썸네일 presigned URL 유효 7일",
    "개별 발송 (BCC 금지) — 동일 자녀 다른 학부모 간 식별 노출 X",
    "섹션: 헤더 → 집계(0건 항목 숨김) → 대표 썸네일 1장 → 교사 피드백 bullet → CTA",
    "Quiz 점수·타 학생 정보·학급 전체 공지 제외"
  ]
}
```

### PV-11 학부모 탈퇴 + 90일 익명화 Cron
```json
{
  "type": "feature",
  "title": "Parent Viewer — 탈퇴 플로우 + 90일 익명화",
  "goal": "/parent/settings '계정 탈퇴' → Parent.deletedAt SET + 모든 ParentSession 즉시 무효 + 링크 self_withdraw 처리. 90일 후 Cron 익명화 (email SHA-256, displayName='탈퇴한 학부모')",
  "context_refs": [
    "ideation/plans/parent-viewer-roadmap.md#14-탈퇴--감사"
  ],
  "blocking_on": ["PV-4", "PV-9"],
  "acceptance": [
    "/parent/settings 탈퇴 CTA + 확인 모달",
    "즉시: Parent.deletedAt=now() + ParentSession revokedAt=now() + ParentChildLink status=revoked + revokedReason=self_withdraw",
    "90일 후 anonymize-parents Cron: email → SHA-256 hash, displayName → '탈퇴한 학부모'",
    "ParentChildLink 레코드는 감사 보존 (삭제 X)",
    "90일 이내 재가입: 동일 이메일로 계정 복구, 기존 링크 복원 X (교사 재발급 경유)",
    "교사 UI에서 탈퇴 학부모 표시는 '연결 해제됨 (탈퇴)'"
  ]
}
```

### PV-12 E2E 보안 게이트 테스트
```json
{
  "type": "feature",
  "title": "Parent Viewer — E2E 보안 게이트 (phase9 QA 필수)",
  "goal": "보안 핵심 경로 6종 E2E 자동화: 403(타 학생 API)·404(타 학부모 링크)·≤60s revoke·rate limit(IP·코드)·썸네일 직접 접근 403·90일 익명화",
  "context_refs": [
    "ideation/plans/parent-viewer-roadmap.md#8-수용-기준-seed-acceptance_criteria-그대로",
    "ideation/tasks/2026-04-12-parent-viewer-access/phase3/decisions.md#5-검증-게이트-phase9-신규-추가"
  ],
  "blocking_on": ["PV-1", "PV-2", "PV-3", "PV-4", "PV-5", "PV-6", "PV-7", "PV-8", "PV-9", "PV-10", "PV-11"],
  "acceptance": [
    "parent 토큰으로 타 학생 studentId API 직접 호출 → 403",
    "parentA 토큰으로 parentB의 ParentChildLink 조회 → 404",
    "타 학생 썸네일 URL 직접 접근 → 403 (presigned 검증)",
    "교사 revoke 후 60초 이내 학부모 세션 401 수신",
    "IP 5회 실패 시 15분 잠금 + 코드 10회 실패 시 즉시 만료",
    "iframe 마운트 0건 (DOM snapshot)",
    "주간 이메일 활동 0건 주 발송 스킵",
    "90일 경과 후 익명화 Cron 실행"
  ]
}
```

---

## 학부모 페어링 v2 — 학급 코드 + 셀프매칭 + 교사 승인 (refinement · parent-viewer-roadmap.md v2)

> **Refinement origin**: `seed_6d7077aac472` (parent_seed_id=`seed_37b35654542f`). `task_id=2026-04-13-parent-class-invite-refine`, ambiguity 0.10.
> **위 PV-1~PV-12는 v1 시드 역사 보존 용도이며, 본 v2 블록이 padlet 활성 진입 계약이다.**
> 상세 설계: `ideation/plans/parent-viewer-roadmap.md` (in-place v2 갱신). PV-1·2·3·5·8·12 개정 + PV-13~16 신설.

### PV-v2-BUNDLE — 학부모 페어링 v2 통합 진입 (refinement)
```json
{
  "type": "feature",
  "title": "Parent Viewer v2 — 학급 코드 + 셀프매칭 + 교사 승인 게이트",
  "refinement": true,
  "parent_seed_id": "seed_37b35654542f",
  "seed_id": "seed_6d7077aac472",
  "task_id": "2026-04-13-parent-class-invite-refine",
  "goal": "학생별 개별 코드 발급(v1)을 학급 단위 단일 Crockford Base32 8자리 ClassInviteCode + 학부모 셀프매칭(학급 명단에서 자녀 선택) + 교사 승인 게이트로 전환. 교사 발급 부담을 학생 수→1로 축소하고 사칭 차단을 승인 게이트로 유지. v1 미배포 상태이므로 완전 폐기 즉시 전환.",
  "context_refs": [
    "ideation/plans/parent-viewer-roadmap.md",
    "ideation/plans/parent-viewer-roadmap.md#15-승인-게이트-흐름-v2-신규",
    "ideation/plans/parent-viewer-roadmap.md#16-교사-ui-v2--학급-설정-학부모-액세스-탭-d-34d-35",
    "ideation/plans/parent-viewer-roadmap.md#7-작업-분할-v2--pv-1--pv-16-총-33-34일",
    "ideation/plans/parent-viewer-roadmap.md#11-변경-로그",
    "ideation/plans/seeds-index.md#-seed-7-v2-학부모-페어링-v2--학급-코드--셀프매칭--교사-승인-2026-04-13-refinement-seed_6d7077aac472",
    "ideation/tasks/2026-04-13-parent-class-invite-refine/phase3/decisions.md",
    "ideation/tasks/2026-04-13-parent-class-invite-refine/phase4/seed.yaml"
  ],
  "supersedes": ["PV-1", "PV-2", "PV-3", "PV-5", "PV-8", "PV-12"],
  "retains_from_v1": ["PV-4", "PV-6", "PV-7", "PV-9", "PV-10", "PV-11"],
  "new_cards": ["PV-13", "PV-14", "PV-15", "PV-16"],
  "total_effort": "33~34일 (v1 27일 대비 +6~7일)",
  "blocking_on": [],
  "acceptance": [
    "ClassInviteCode 엔티티 신설 (classroomId·Crockford Base32 8자리·학기말 자동 만료 + 교사 수동 회전·무제한 maxUses). ParentInviteCode 생성 안 함 (v1 테이블 완전 폐기)",
    "ParentChildLink.status가 'pending'|'active'|'rejected'|'revoked' 4값 유니언으로 전이 (pending→active approve / pending→rejected reject|auto_expire|code_rotated / active→revoked)",
    "감사 필드 6종 (requestedAt·approvedAt·approvedById·rejectedAt·rejectedById·rejectedReason) 및 revokedReason에 'rejected_by_teacher'·'auto_expired_pending'·'code_rotated' 추가. 기존 v1 값은 'teacher_revoked'·'year_end'·'parent_self_leave'로 재명명",
    "rejectedReason enum 신규 'wrong_child'|'not_parent'|'other' 3종 드롭다운 (자유 텍스트 v2 파킹)",
    "동일 자녀에 부·모 각각 별도 승인 가능 (@@unique([parentId, studentId])만 적용, 자녀당 active 학부모 상한 없음)",
    "ParentSession은 signup 시점 생성, BoardMember는 approve 시점 생성 (RLS 오염 방지)",
    "학부모 signup → 학급 명단 조회('반·번호+성+O+끝글자' 마스킹 형식 '3반 12번 김O민', 프로필 사진 비노출, 반·번호 정렬) → 자녀 1건 선택 → pending 신청",
    "학부모당 동시 pending 상한 3건 / pending TTL 7일 / 권고 응답 SLA 24h (hard SLA 없음)",
    "pending 응답은 HTTP 200 OK + {'status':'pending'} payload flag (401/403 금지). 클라이언트 /parent/pending 렌더",
    "교사 UI '학부모 액세스' 탭 3-섹션: (a) 초대 코드 — 현재 코드 표시·QR/링크 복사·회전 버튼·회전 히스토리 / (b) 승인 인박스 — pending 리스트 + D+N 일자 배지(회색/노랑/빨강) + 일괄 승인 + 개별 승인·거부(사유 드롭다운 3종) + 검색 / (c) 연결된 학부모 — 학부모-자녀 쌍 + revoke + 최근 접속일. 학생 카드 드롭다운 '학부모 초대' 제거",
    "알림 스케줄: D+0 배지(회색) / D+3 교사 이메일 리마인더 ('[Aura-board] N명의 학부모가 승인 대기 중') / D+6 교사 경고 이메일 + 배지(빨강) / D+7 Vercel Cron (UTC 17:00 = KST 02:00 일 1회) pending 7일 초과 auto_expired_pending rejected 처리 + 교사 요약 이메일",
    "거부/만료 학부모 이메일 공통 규칙: 교사 이름·이메일·전화번호 미노출 (학교 대표 연락처만) + 재신청 deep link 포함 + 선택된 사유 문구만 본문 삽입. 교사 자유 메시지 입력 v1 미제공",
    "rejected_by_teacher 이메일 본문은 wrong_child/not_parent/other 사유 3종 중 택일 (확정 문구 고정). auto_expired_pending 이메일은 7일 고정 문구",
    "동일 학부모 이메일 거부 누적 3회 초과 시 24h 재신청 차단 (쿨다운)",
    "ClassInviteCode 회전 시 해당 학급 pending 건 일괄 rejected (revokedReason='code_rotated') 처리 + 학부모에 재신청 이메일. active 링크는 유지",
    "3축 rate limit: IP 5회 실패/15분 잠금 + 코드당 50회/일 + 학급당 100회/일. 코드 실패 10회 자동 만료 트리거 제거 (DoS 벡터)",
    "parentAuthOnlyMiddleware 신규 (매칭 전 /api/parent/signup·/match/code·/match/students·/match/request 전용). parentScopeMiddleware는 매칭 후 /parent/child/*·/home·/settings에 적용하되 status='active' 조건 추가",
    "RLS 모든 콘텐츠 테이블에 studentId ∈ parent.children WHERE status='active' 조건 적용 (pending·rejected·revoked 차단)",
    "Free/Pro 발급 한도 개념 폐지 (학급 코드 체제에서 의미 붕괴). Pro 혜택은 주간 이메일(월 09:00 KST) 전용으로 재정의",
    "v1 엔드포인트 /api/parent/redeem·/api/parent-invite-codes 등은 410 Gone 반환 (v1 미배포로 호출 없음 가정). Prisma migration 단일 — ClassInviteCode CREATE + status 유니언 확장 + 감사 필드 6종 추가",
    "parent_seed_id=seed_37b35654542f 체인 유지. seed_37b35654542f는 archive 이관 후보 (phase7 dispatcher 판정)",
    "seeds-index.md Seed 7 표기 'superseded by seed_6d7077aac472'. parent-viewer-roadmap.md v2 in-place 갱신",
    "유지 결정: 매직링크 15분·ParentSession 7일·자녀 5명 상한·BCC 금지·3중 격리·Revoke 60s·Pro 월 09:00 KST 주간 이메일·90일 익명화·Crockford CSPRNG·v1 읽기 전용(v1 history 페이지 잔존 — 삭제 X)",
    "E2E 신규 시나리오: pending 상태에서 자녀 콘텐츠 열람 시도 → 403 · 교사 거부 후 학부모 이메일 사유 3종 중 택일 문구 확인 · Cron 실행 후 D+7 auto_expired_pending 검증 · 코드 회전 시 pending 일괄 rejected·active 유지 검증 · IP/코드/학급 3축 rate limit · 3회 초과 거부 쿨다운 · 동명이인 부·모 각각 승인 가능",
    "작업 공수 v1 27일 → v2 33~34일 범위 내 완료"
  ],
  "v2_parked": [
    "교사 자유 메시지 입력 (거부 이메일 커스텀 문구)",
    "에스컬레이션 경로 (승인 SLA 초과 시 학급장/관리자 알림)",
    "사칭 감지 SOP 고도화 (거부율 임계 알고리즘·패턴 감지)",
    "실시간 push / 카카오톡 알림톡",
    "Kakao/Google OAuth · 비밀번호 로그인",
    "Redis 블랙리스트 기반 즉시 revoke (<1s)",
    "학부모 앱 네이티브",
    "부/모 합산 권한 통합 (각각 별도 승인 유지)"
  ]
}
```

---

## Canva Publisher 수신 (canva-publisher-receiver-roadmap.md)

> Seed 8 `seed_26af361e92b7`. implementation-roadmap P0-②의 **서버(padlet) 측 수신** 전담.
> 상위 기획: `ideation/plans/implementation-roadmap.md#p0-②-content-publisher-intent-앱-canva--aura-board`
> 상세 설계: `ideation/plans/canva-publisher-receiver-roadmap.md`

### CR-1 스키마 & 마이그 Stage 1 (nullable tokenPrefix)
```json
{
  "type": "feature",
  "title": "ExternalAccessToken — tokenPrefix/label/lastUsedAt/scopeBoardIds 필드 추가 (Stage 1 nullable)",
  "goal": "prisma/schema.prisma에 tokenPrefix String? @unique + label String + lastUsedAt DateTime? + scopeBoardIds String[] 추가. nullable로 안전 마이그.",
  "context_refs": [
    "ideation/plans/canva-publisher-receiver-roadmap.md#11-엔티티--신규-테이블-없음-externalaccesstoken-확장만",
    "ideation/plans/canva-publisher-receiver-roadmap.md#12-3-stage-마이그레이션-d4--r2"
  ],
  "blocking_on": [],
  "acceptance": [
    "prisma migrate dev 통과, 기존 row 영향 없음",
    "tokenPrefix unique 인덱스 생성 (nullable)",
    "label 기본값 '' 또는 'unnamed' 할당",
    "scopeBoardIds 기본 []",
    "lastUsedAt null 허용",
    "CI schema 검증 통과"
  ]
}
```

### CR-2 src/lib/external-auth.ts (PAT 생성·해싱·검증)
```json
{
  "type": "feature",
  "title": "PAT 생성·해싱·timing-safe 검증 유틸",
  "goal": "generatePat() → aurapat_{8}_{40}, hashPat(secret)=SHA-256(secret‖PEPPER), resolvePatByPrefix(prefix)=O(1) DB lookup + dummy hash timing-safe 비교",
  "context_refs": [
    "ideation/plans/canva-publisher-receiver-roadmap.md#11-엔티티--신규-테이블-없음-externalaccesstoken-확장만"
  ],
  "blocking_on": ["CR-1"],
  "acceptance": [
    "generatePat() — base62 id(8) + secret(40), PAT_PEPPER 환경변수 사용",
    "hashPat — Node crypto SHA-256 단방향",
    "resolvePatByPrefix — prefix 존재 여부와 무관하게 일정 시간 (R5 timing side-channel)",
    "unit test — 유효 PAT 통과, 잘못된 secret 실패, 존재하지 않는 prefix 실패",
    "성능 — 각 요청 < 10ms"
  ]
}
```

### CR-3 POST /api/external/cards (핵심 수신 엔드포인트)
```json
{
  "type": "feature",
  "title": "POST /api/external/cards — Canva 앱 수신 엔드포인트",
  "goal": "Content-Length 4MB 가드 → PAT 파싱/검증 → scope/tier 체크 → 3축 rate limit → Zod strict body → boardId allowlist → Blob 스트리밍 put → Card INSERT → lastUsedAt 갱신 → 200 {id,url}",
  "context_refs": [
    "ideation/plans/canva-publisher-receiver-roadmap.md#16-요청응답-계약-d14d15d16",
    "ideation/plans/canva-publisher-receiver-roadmap.md#17-에러-코드-표",
    "ideation/plans/canva-publisher-receiver-roadmap.md#18-card-기본값-acceptance",
    "ideation/tasks/2026-04-12-canva-publisher-receiver/phase3/decisions.md"
  ],
  "blocking_on": ["CR-1", "CR-2", "CR-4", "CR-5"],
  "acceptance": [
    "Zod strict 4필드 검증 (boardId cuid / title 1–200 / imageDataUrl data:image/png;base64 / sectionId? cuid|null) — unknown key 422 invalid_data_url",
    "성공 200 {id, url: 'https://aura-board-app.vercel.app/board/<slug>#c/<cardId>'}",
    "에러 {error:{code,message}} 통일 포맷",
    "Content-Length > 4.0MB → 413 payload_too_large (body parse 전)",
    "Free 토큰 cards:write → 402 tier_required + upgrade link",
    "boardId ∉ scopeBoardIds(빈 배열 제외) → 403 forbidden_board",
    "p95 < 2000ms (3MB 업로드 기준)",
    "Card 기본값 width=240 height=160 content='' authorId=token.teacherId sectionId=body.sectionId ?? null",
    "통합 테스트 — 유효/각 에러 코드 분기 커버"
  ]
}
```

### CR-4 Upstash Redis 3축 rate limit
```json
{
  "type": "feature",
  "title": "Upstash sliding window 3축 rate limit 헬퍼",
  "goal": "@upstash/ratelimit sliding window — per-token 60/min · per-teacher 300/hour · per-IP 300/min. 429 + Retry-After. Upstash 장애 fail-open 기본.",
  "context_refs": [
    "ideation/plans/canva-publisher-receiver-roadmap.md#14-rate-limit-3축-d12"
  ],
  "blocking_on": [],
  "acceptance": [
    "3축 OR 판정 — 하나라도 초과 시 429",
    "Retry-After 헤더에 남은 초 반환",
    "Upstash 장애 시 fail-open 기본 (RL_FAIL_MODE=close 옵션으로 fail-close 전환)",
    "Redis key 네이밍 rl:pat:{tokenId}:1m · rl:teacher:{teacherId}:1h · rl:ip:{ipHash}:1m",
    "IP는 sha256 해시 저장 (원문 IP DB 금지)",
    "load test — 60·300·300 한도 경계에서 정확히 429"
  ]
}
```

### CR-5 Vercel Blob 스트리밍 업로드
```json
{
  "type": "feature",
  "title": "imageDataUrl → Vercel Blob 스트리밍 put",
  "goal": "base64 prefix 제거 후 Readable.from() chunk stream → @vercel/blob put(key, stream, {multipart:true}). 메모리 버퍼 X. key 규약 external-cards/{boardId}/{cardId}.png",
  "context_refs": [
    "ideation/plans/canva-publisher-receiver-roadmap.md#15-p95--2000ms--스트리밍-업로드-d13--r1"
  ],
  "blocking_on": [],
  "acceptance": [
    "3MB PNG 업로드 p95 < 2000ms (phase3 D13)",
    "메모리 사용량 < 10MB (스트리밍 검증)",
    "key 규약 external-cards/{boardId}/{cardId}.png 준수",
    "Blob 업로드 실패 시 500 internal_error 반환 + Card INSERT 롤백",
    "BLOB_READ_WRITE_TOKEN 환경변수 문서화"
  ]
}
```

### CR-6 /api/tokens CRUD (발급·조회·폐기·라벨 변경)
```json
{
  "type": "feature",
  "title": "교사 PAT CRUD — GET/POST /api/tokens · PATCH/DELETE /api/tokens/[id]",
  "goal": "교사 인증(NextAuth)으로 자기 토큰만 조회/발급/라벨 수정/revoke. POST 응답에 secret 1회만 포함. DELETE는 soft revoke(revokedAt=now()).",
  "context_refs": [
    "ideation/plans/canva-publisher-receiver-roadmap.md#2-신규변경-파일-맵"
  ],
  "blocking_on": ["CR-1", "CR-2"],
  "acceptance": [
    "GET /api/tokens — 교사 본인 토큰 목록(활성만 또는 all 쿼리)",
    "POST /api/tokens — {label, scopes, scopeBoardIds, expiresAt} 입력, 응답에 secret 1회만 포함",
    "PATCH /api/tokens/[id] — label 수정만 허용 (scopes/scopeBoardIds 변경은 새 발급 강제)",
    "DELETE /api/tokens/[id] — soft revoke revokedAt=now()",
    "타 교사 토큰 조회/수정 → 404 (기록 비노출)",
    "Free 교사가 scopes:['cards:write'] 요청 → 402 tier_required (발급 단계 차단)"
  ]
}
```

### CR-7 교사 UI /(teacher)/settings/external-tokens
```json
{
  "type": "feature",
  "title": "교사 PAT 관리 UI — 갤탭 S6 Lite 최적화",
  "goal": "목록(label·prefix·scopes·scopeBoardIds·expiresAt·lastUsedAt·Revoke) + FAB '+ 새 토큰' + 발급 모달(label·scope·보드범위·유효기간 드롭다운) + 1회 공개 모달(Copy + Download .txt)",
  "context_refs": [
    "ideation/plans/canva-publisher-receiver-roadmap.md#3-교사-ui--teachersettingsexternal-tokens-갤탭-s6-lite",
    "ideation/plans/tablet-performance-roadmap.md#2-성능-예산"
  ],
  "blocking_on": ["CR-6"],
  "acceptance": [
    "갤탭 S6 Lite 1200×800 포트레이트/가로 대응",
    "터치 타겟 ≥ 44px",
    "유효기간 드롭다운 1/30/90(기본)/365/무기한 + '권장: 90일 회전' 주석",
    "1회 공개 모달 — Copy(clipboard) + Download(.txt, 파일명 aura-token-{label}-{YYYYMMDD}.txt)",
    "모달 닫기 전 '복사하셨나요?' 확인",
    "Free 계정 — cards:write 스코프 체크박스 disabled + 잠금 배지 + '내 모든 보드'/'특정 보드' 라디오 UI",
    "Playwright E2E — 발급→공개→복사→재접근 시 secret 비노출 확인",
    "모달 재표시 불가 (secret 페이지 재마운트 시 복원 X)"
  ]
}
```

### CR-8 마이그 Stage 2·3 (legacy revoke + NOT NULL)
```json
{
  "type": "feature",
  "title": "ExternalAccessToken 레거시 revoke + tokenPrefix NOT NULL 전환",
  "goal": "Stage 2 — 레거시 row 일괄 revokedAt=now() + 교사별 Resend 이메일 'PAT 재발급 필요' 발송. Stage 3 — 7일 유예 후 tokenPrefix String @unique NOT NULL 전환.",
  "context_refs": [
    "ideation/plans/canva-publisher-receiver-roadmap.md#12-3-stage-마이그레이션-d4--r2"
  ],
  "blocking_on": ["CR-1", "CR-2", "CR-6", "CR-7"],
  "acceptance": [
    "Stage 2 — 레거시(tokenPrefix IS NULL) row 100% revokedAt SET",
    "Stage 2 — 교사별 이메일 발송 증거 로그 (발송 0명 누락 검증)",
    "Stage 2와 Stage 3 사이 최소 7일 유예",
    "Stage 3 — tokenPrefix NOT NULL 제약 추가 전 '남은 null row 0' 검증 쿼리 실행",
    "Stage 3 — 운영 DB에 오류 없이 적용",
    "각 stage는 별도 PR (MG-A / MG-B / MG-C)"
  ]
}
```

### CR-9 /api/external/healthz 모니터
```json
{
  "type": "feature",
  "title": "/api/external/healthz — Upstash·Blob·DB 라이브니스",
  "goal": "GET /api/external/healthz 시 Upstash ping + Blob head + DB SELECT 1 동시 체크. Vercel Cron 1분 주기 + 실패 시 알림 (Resend).",
  "context_refs": [
    "ideation/plans/canva-publisher-receiver-roadmap.md#14-rate-limit-3축-d12"
  ],
  "blocking_on": ["CR-4", "CR-5"],
  "acceptance": [
    "각 의존성 down 시 503 + JSON {status:'down', failed:['upstash'|'blob'|'db']}",
    "모든 의존성 OK → 200 {status:'ok'}",
    "Vercel Cron 1분 주기 등록",
    "2회 연속 실패 시 Resend 알림 발송",
    "응답 시간 < 500ms p95"
  ]
}
```

### CR-10 E2E 보안 게이트 (phase9 QA 필수)
```json
{
  "type": "feature",
  "title": "Canva Publisher 수신 — E2E 보안 게이트 10종",
  "goal": "무효 PAT 401·Free 토큰 402·boardId 범위 위반 403·4MB 초과 413·3축 rate limit 429·p95 < 2000ms·1회 모달 재표시 불가·secret scanner 정규식·prefix miss timing 일정·Upstash fail-open 10개 시나리오 자동화",
  "context_refs": [
    "ideation/plans/canva-publisher-receiver-roadmap.md#5-수용-기준-seedyaml-acceptance_criteria",
    "ideation/tasks/2026-04-12-canva-publisher-receiver/phase3/decisions.md#3-리스크-해소-상태"
  ],
  "blocking_on": ["CR-1","CR-2","CR-3","CR-4","CR-5","CR-6","CR-7","CR-8","CR-9"],
  "acceptance": [
    "시나리오 1: 무효 PAT → 401 invalid_token (timing-safe)",
    "시나리오 2: Free 교사 토큰 → 402 tier_required + upgrade link",
    "시나리오 3: boardId ∉ scopeBoardIds → 403 forbidden_board",
    "시나리오 4: Content-Length 4.5MB → 413 payload_too_large (body parse 전)",
    "시나리오 5: 61 req/min → 429 + Retry-After",
    "시나리오 6: 3MB 업로드 p95 < 2000ms (50회 측정)",
    "시나리오 7: 발급 모달 닫은 후 재마운트 시 secret 비노출",
    "시나리오 8: 발급 secret이 GitHub secret scanner 정규식 ^aurapat_[0-9a-zA-Z]{8}_[0-9a-zA-Z]{40}$ 매칭",
    "시나리오 9: 존재하지 않는 prefix 호출 시 응답 시간이 유효 prefix와 ±10% 이내 (R5)",
    "시나리오 10: Upstash 장애 주입 시 fail-open 200, RL_FAIL_MODE=close 시 503 rate_limit_unavailable"
  ]
}
```

---

## gongmun-assistant v1 — CLI 공문 붙임 자동 생성 (gongmun-assistant-v1-roadmap.md)

> **외부 프로젝트 주의**: 이 블록들은 **Aura-board/padlet과 무관**. `gongmun-assistant/tasks/{YYYY-MM-DD-slug}/phase0/request.json` 에 복사. `destination: gongmun-assistant`.

### GM-1 프로젝트 스캐폴드 + 모듈 경계
```json
{
  "type": "feature",
  "title": "gongmun-assistant 스캐폴드 + 레이어 경계 정적 검사",
  "goal": "poetry 기반 Python 3.11+ 프로젝트 스캐폴드, src/gongmun/{cli,core,templates,llm,hwpx_io,config,types} 빈 패키지, entry_points 등록. 레이어 임포트 규칙(core→cli 금지, table_fill→llm 금지) 정적 검사 CI",
  "destination": "gongmun-assistant",
  "context_refs": [
    "ideation/plans/gongmun-assistant-v1-roadmap.md#1-모듈-구조-cli-단일-런타임-코어-중립",
    "ideation/tasks/2026-04-12-gongmun-attachment-autofill/phase4/seed.yaml"
  ],
  "acceptance": [
    "poetry install 성공, Python 3.11+ 강제",
    "pyproject.toml entry_points: gongmun = gongmun.cli.app:app",
    "import-linter 또는 pydeps로 core→cli 금지·table_fill→llm 금지 단언",
    "CI에서 레이어 규칙 위반 시 빌드 실패"
  ]
}
```

### GM-2 hwpx_io — python-hwpx 래퍼 + linesegarray 제거
```json
{
  "type": "feature",
  "title": "hwpx_io — reader·writer·zip_rules",
  "goal": "python-hwpx v2.9+ 래퍼. extract_text·find_slots·fill_by_path·strip_linesegarray·atomic_save. hwpx-master 스킬 §3 (<linesegarray> 전량 제거) + §4 (mimetype ZIP_STORED + 첫 엔트리) 체크리스트 단위 테스트",
  "destination": "gongmun-assistant",
  "context_refs": [
    "ideation/plans/gongmun-assistant-v1-roadmap.md#2-8단계-파이프라인-의사코드",
    "gongmun-assistant/skills/hwpx-master.SKILL.md"
  ],
  "blocking_on": ["GM-1"],
  "acceptance": [
    "extract_text(hwpx_path) → section0 + header 전체 텍스트",
    "fill_by_path(doc, xpath, value) — lxml.etree API만 사용 (문자열 조작 금지)",
    "strip_linesegarray 후 section0.xml의 <linesegarray> 요소 0개 단언",
    "atomic_save 후 mimetype 엔트리가 zip 첫 위치 + ZIP_STORED 압축 모드",
    "Contents/section0.xml 이외 파일 편집 시 AssertionError"
  ]
}
```

### GM-3 llm provider ABC + Anthropic + Ollama + privacy 스캐너
```json
{
  "type": "feature",
  "title": "LLM provider 추상 + 두 구현체 + 개인정보 스캐너",
  "goal": "LLMProvider ABC (analyze_context / suggest_attachment_type / fill_narrative) 시그니처 통일. AnthropicProvider(Sonnet 기본·Haiku 폴백) + OllamaProvider(qwen2.5:14b-instruct-q4 기본). privacy.py로 주민번호·전화·이메일·학번 패턴 스캔 → 매치 시 exit 5",
  "destination": "gongmun-assistant",
  "context_refs": [
    "ideation/plans/gongmun-assistant-v1-roadmap.md#4-개인정보-경계-정책",
    "ideation/tasks/2026-04-12-gongmun-attachment-autofill/phase3/decisions.md#d4-llm-프로바이더--claude-api-기본--ollama-옵션-답변-라우팅-에이전트-보안-정책-준수"
  ],
  "blocking_on": ["GM-1"],
  "acceptance": [
    "두 provider가 동일 ABC 시그니처 구현, config.llm.provider로 주입 전환",
    "Anthropic 재시도·백오프·타임아웃·토큰 상한, Haiku 폴백 로직",
    "privacy.assert_no_pii가 주민번호 13자리·010-xxxx-xxxx·이메일·학번 5~8자리 매칭",
    "매치 시 PrivacyPatternDetected 예외 → CLI exit code 5"
  ]
}
```

### GM-4 템플릿 catalog + 시스템 번들 6개 + 사이드카 YAML
```json
{
  "type": "feature",
  "title": "catalog.resolve + 시스템 번들 6개 + 사용자 템플릿 병합",
  "goal": "catalog.resolve(type, school_level, template_id?) → (path, slot_manifest, schema). 시스템 번들 P0 3종 × 2 학교급 = 6 hwpx + catalog.json 매니페스트. 사용자 ~/.gongmun/templates/user/{type}/{name}.hwpx + {name}.yaml 사이드카 자동 병합 (user-prefixed id). 로드 실패 시 경고 후 스킵(크래시 금지). scripts/validate_template.py로 3항목(슬롯 존재·zip 무결성·mimetype 위치) 검증",
  "destination": "gongmun-assistant",
  "context_refs": [
    "ideation/plans/gongmun-assistant-v1-roadmap.md#31-p0-붙임-3종--2-학교급--시스템-템플릿-6개",
    "ideation/tasks/2026-04-12-gongmun-attachment-autofill/phase3/decisions.md#d3-템플릿-관리--하이브리드-번들--사용자-디렉토리-규약-답변-라우팅-에이전트"
  ],
  "blocking_on": ["GM-2"],
  "acceptance": [
    "시스템 번들 6개가 validate_template.py 통과",
    "사용자 템플릿 로드 성공 시 user:{name} id로 catalog에 merge",
    "사용자 템플릿 손상 시 stderr 경고 + 해당 항목 스킵 + CLI 크래시 없음",
    "catalog.resolve 미존재 template_id → TemplateNotFound (exit 4)",
    "슬롯 토큰 {{snake_case}} · * 접두 필수 슬롯 규약 파서 단위 테스트"
  ]
}
```

### GM-5 core.table_fill — 결정론 표 채움 + 정적 격리
```json
{
  "type": "feature",
  "title": "결정론 표 채움 + LLM 호출 0회 단언",
  "goal": "fill_table_by_label(doc, rows) — CSV 파싱 → 헤더 라벨 자동 탐지 → 결정론 hwpx 표 채움. schema 검증(필드 누락·타입 불일치 → exit 5 이전 단계). test_privacy_boundary.py로 (1) core.table_fill 모듈의 llm.* 임포트 금지 정적 단언 (2) mock Anthropic 클라이언트에서 CSV 행 문자열이 payload에 0회 단언",
  "destination": "gongmun-assistant",
  "context_refs": [
    "ideation/plans/gongmun-assistant-v1-roadmap.md#4-개인정보-경계-정책",
    "ideation/tasks/2026-04-12-gongmun-attachment-autofill/phase3/decisions.md#d5-개인정보-처리-정책--로컬-고정마스킹-로그자동-삭제-없음-답변-라우팅-에이전트-기존-보안-정책-준수"
  ],
  "blocking_on": ["GM-2"],
  "acceptance": [
    "명단표 템플릿 + 30행 CSV → 결정론 채움 + LLM 호출 0회",
    "schema 불일치(필수 컬럼 누락 등) → SchemaError 명확 메시지",
    "AST 수준 정적 검사: core.table_fill은 anthropic·ollama 미임포트",
    "E2E 파이프라인 실행 시 mock 클라이언트에 CSV 행 문자열 누출 0회"
  ]
}
```

### GM-6 core.pipeline Stage 1-8 + narrative_fill
```json
{
  "type": "feature",
  "title": "파이프라인 오케스트레이션 + 서술 슬롯 채움",
  "goal": "generate_attachment(main_input, att_type, csv_data, template_id, config) — Stage 1 입력 수렴(hwpx/txt/stdin/prompt) → Stage 2 LLM 맥락 분석 → Stage 3 유형 결정(auto면 LLM) → Stage 4 catalog.resolve → Stage 5 표 채움(table_fill) → Stage 6 서술 슬롯(narrative_fill, LLM) → Stage 7 linesegarray 제거 + atomic_save → Stage 8 validate. exit code 0/2/3/4/5. .tmp/ 수명 관리 (실패 시 유지, 성공 시 정리). parent-letter 이외에서 --prompt 사용 시 거부",
  "destination": "gongmun-assistant",
  "context_refs": [
    "ideation/plans/gongmun-assistant-v1-roadmap.md#2-8단계-파이프라인-의사코드"
  ],
  "blocking_on": ["GM-3", "GM-4", "GM-5"],
  "acceptance": [
    "Stage 1에서 4가지 입력이 main_text: str로 단일 수렴",
    "att_type != 'parent-letter' 인데 --prompt 사용 → InvalidInputPath (exit 1)",
    "LLM temperature=0 기준 동일 입력 재실행 시 bit-identical 출력",
    "Stage 5에서 LLM 호출 0회, Stage 2·3·6만 LLM 호출",
    "각 stage 실패 시 pipeline_stage 번호가 오류에 포함"
  ]
}
```

### GM-7 config 스키마 + 첫 실행 셋업 마법사
```json
{
  "type": "feature",
  "title": "~/.gongmun/config.yaml 스키마 + 셋업 마법사",
  "goal": "pydantic 스키마로 school(name·level·department) / officer(name·role·contact) / approval_chain[] / llm(provider·model) 필드 정의. 첫 실행 시 ~/.gongmun/ 디렉토리 생성 + 대화형 셋업. 기본은 LLM에 결재자 실명 미전달(로컬 치환), opt-in 플래그 --share-officer-to-llm 시만 전달",
  "destination": "gongmun-assistant",
  "context_refs": [
    "ideation/plans/gongmun-assistant-v1-roadmap.md#5-배포운영",
    "ideation/tasks/2026-04-12-gongmun-attachment-autofill/phase3/decisions.md#d6-결재라인기관-정보-범위--gongmunconfigyaml-로컬-고정-답변-라우팅-에이전트-기존-결정-준수"
  ],
  "blocking_on": ["GM-1"],
  "acceptance": [
    "pydantic 검증 통과하는 config.yaml 로드",
    "config 누락 필수 필드 있을 시 셋업 마법사 재실행 제안",
    "--share-officer-to-llm 부재 시 LLM 프롬프트에 officer.name 미포함 단언",
    "--share-officer-to-llm 사용 시 문서에 리스크 명시 로그 출력",
    "llm.provider: anthropic|ollama 스위치로 provider 주입 정상 전환"
  ]
}
```

### GM-8 CLI typer 커맨드 + 대화형 + gongmun.bat
```json
{
  "type": "feature",
  "title": "gongmun attach 커맨드 + 인자 없이 실행 시 대화형 + Windows 더블클릭",
  "goal": "typer 기반 `gongmun attach --main ... --type ... --data ... --template ... --output-dir ... --share-officer-to-llm --debug`. 인자 없이 `gongmun` 실행 시 inquirer 프롬프트 전환(hwpx 경로 먼저 묻고 비우면 유형 선택 → 간단 설명 → parent-letter 폴백). Windows gongmun.bat 래퍼로 탐색기 더블클릭 시 터미널 진입",
  "destination": "gongmun-assistant",
  "context_refs": [
    "ideation/plans/gongmun-assistant-v1-roadmap.md#52-실행-예시",
    "ideation/tasks/2026-04-12-gongmun-attachment-autofill/phase3/decisions.md#d1-실행-모드--v1--cli-고정-답변-라우팅-에이전트"
  ],
  "blocking_on": ["GM-6", "GM-7"],
  "acceptance": [
    "`gongmun attach --help` 타입·설명 포함 자동 생성",
    "`gongmun` (인자 없음) → 대화형 프롬프트 진입 + hwpx 경로 비우면 유형 선택 폴백",
    "Windows 탐색기에서 gongmun.bat 더블클릭 → cmd 창 + 대화형 진입",
    "`--type parent-letter --prompt '...'` 본문 없이 성공, `--type roster --prompt` 실패"
  ]
}
```

### GM-9 validate.py + E2E 테스트 + 한컴 수동 QA
```json
{
  "type": "feature",
  "title": "hwpx 검증기 + P0 3종 × 2 학교급 E2E + 한컴오피스 수동 QA 가이드",
  "goal": "validate.run(hwpx_path) — 4항목(zip 무결성·mimetype ZIP_STORED + 첫 엔트리·XML well-formedness·linesegarray=0). 실패 시 validate_report.txt 기록. 원본 템플릿 불변 보장(atomic save). E2E 테스트: P0 3종(명단표·동의서·가정통신문) × 2 학교급(초/중고) 각각 대표 시나리오 1개씩. 한컴오피스에서 '보안설정' 경고 없이 열리는지 수동 QA 체크리스트 문서화",
  "destination": "gongmun-assistant",
  "context_refs": [
    "ideation/plans/gongmun-assistant-v1-roadmap.md#8-리스크-요약",
    "gongmun-assistant/skills/hwpx-master.SKILL.md"
  ],
  "blocking_on": ["GM-6"],
  "acceptance": [
    "validate.run이 4항목 전부 OK일 때만 report.ok=True",
    "P0 6개(3 type × 2 school_level) E2E 각 bit-identical 2회 재실행(temperature=0)",
    "생성된 hwpx 6개를 한컴오피스 2024에서 보안경고 없이 열림 (수동 QA 체크리스트 통과)",
    "validate 실패 시 exit 2 + validate_report.txt 생성",
    "원본 템플릿 mtime/해시 불변 단언"
  ]
}
```

### GM-10 PyPI 배포 + 설치 가이드 + 템플릿 제작 가이드
```json
{
  "type": "feature",
  "title": "pipx install gongmun-assistant 배포 + 문서",
  "goal": "PyPI 배포 파이프라인(GitHub Actions or poetry publish). docs/install-windows.md (pipx 설치 + .bat PATH 등록 가이드). docs/template-authoring.md (슬롯 {{snake_case}} · * 접두 필수 · 사이드카 YAML 스키마). README.md (교사 대상 간결 사용법)",
  "destination": "gongmun-assistant",
  "context_refs": [
    "ideation/plans/gongmun-assistant-v1-roadmap.md#5-배포운영",
    "ideation/tasks/2026-04-12-gongmun-attachment-autofill/phase3/decisions.md#d7-배포-모드--개인-설치-pippipx-답변-라우팅-에이전트-스코프-정책-준수"
  ],
  "blocking_on": ["GM-8", "GM-9"],
  "acceptance": [
    "`pipx install gongmun-assistant` 성공 후 `gongmun --version` 출력",
    "Windows에서 pipx 설치 경로의 gongmun.bat이 PATH에 자동 등록",
    "docs/template-authoring.md에 P0 3종 각 필수/선택 슬롯 목록 명시",
    "README가 비개발자 교사가 15분 내 첫 실행 가능한 수준",
    "v1 범위(P0 3종·pip 개인 설치)와 비포함(GUI·CRUD·학교 서버) 명시"
  ]
}
```

---

## 과제 배부 보드 (assignment-board-roadmap.md, Seed 11)

> destination: **padlet**. 시드 `seed_38c34e91bf28` (ambiguity 0.083, interview_20260414_131412).
> 핵심 산출물: `Board.layout="assignment"` 확장 + `AssignmentSlot` 신규 엔티티 + 30-slot 5×6 정형 격자 UI + 상태 머신(submissionStatus × gradingStatus) + 전체화면 모달(반려 사유 필수) + Roster 수동 동기화.
> 블록을 padlet의 `tasks/{YYYY-MM-DD-slug}/phase0/request.json` 으로 복사해 feature 파이프라인에 투입.

### AB-1 Prisma 마이그레이션 (Board.layout 확장 + AssignmentSlot)
```json
{
  "type": "feature",
  "title": "assignment-board — AssignmentSlot 엔티티 + Board 확장 마이그레이션",
  "goal": "Board.layout='assignment' 확장 + 3 필드(assignmentDueAt·assignmentAllowLate·assignmentGuideText) + AssignmentSlot 신규 엔티티 1종 마이그레이션 적용",
  "context_refs": [
    "ideation/plans/assignment-board-roadmap.md#1-board-확장-layout--assignment",
    "ideation/plans/assignment-board-roadmap.md#2-신규-엔티티-assignmentslot",
    "ideation/tasks/2026-04-14-assignment-board-impl/phase4/seed.yaml",
    "ideation/tasks/2026-04-14-assignment-board-impl/phase3/decisions.md",
    "ideation/tasks/2026-04-14-assignment-board-impl/phase1/exploration.md#3-1순위-권고"
  ],
  "blocking_on": [],
  "acceptance": [
    "Board.layout enum에 'assignment' 추가 (기존 'freeform'·'event-signup'·'breakout'와 공존)",
    "AssignmentSlot 테이블에 @@unique([boardId,studentId]) + @@unique([boardId,slotNumber]) 제약 존재",
    "submissionStatus 기본값 'assigned', gradingStatus 기본값 'not_graded'",
    "returnReason 컬럼 VARCHAR(200) nullable",
    "기존 Submission·Card·Classroom·Student 테이블 무수정",
    "기존 event-signup·breakout 보드에 영향 없음 (회귀 테스트)"
  ]
}
```

### AB-2~AB-10 통합 (과제 보드 feature 본체)
```json
{
  "type": "feature",
  "title": "assignment-board — 학급 로스터 기반 과제 수거 보드 v1",
  "goal": "교사가 학급(N≤30)으로 과제 보드 생성 시 학생 번호순 5×6 정형 격자에 AssignmentSlot 자동 인스턴스화. 학생은 본인 slot에만 제출, 교사는 모달에서 반려(사유 필수)/리뷰 완료. Seesaw 썸네일 + Moodle 이원 상태 배지로 제출/미제출 시각 구분.",
  "context_refs": [
    "ideation/plans/assignment-board-roadmap.md",
    "ideation/plans/tablet-performance-roadmap.md#2a-30-카드-5x6-정형-격자-성능-예산-assignment-board-seed-11",
    "ideation/plans/event-signup-roadmap.md#submission-엔티티-공유--assignment-board-seed-11-2026-04-14",
    "ideation/plans/parent-viewer-roadmap.md#5-cross-cutting-자녀-범위-매트릭스--single-source-of-truth",
    "ideation/tasks/2026-04-14-assignment-board-impl/phase4/seed.yaml",
    "ideation/tasks/2026-04-14-assignment-board-impl/phase3/decisions.md",
    "ideation/tasks/2026-04-14-assignment-board-impl/phase2/sketch.md",
    "ideation/tasks/2026-04-14-assignment-board-impl/phase1/exploration.md"
  ],
  "performance_budget_ref": "ideation/plans/tablet-performance-roadmap.md#2a-30-카드-5x6-정형-격자-성능-예산-assignment-board-seed-11",
  "blocking_on": ["AB-1 Prisma 마이그레이션 선행"],
  "acceptance": [
    "교사가 학급(Classroom) 선택 → 보드 생성 시 N≤30 검증 통과 후 N개 AssignmentSlot 트랜잭션 insert (slotNumber = Student.number 스냅샷)",
    "N>30 학급은 생성 차단 + '분반 또는 v2 대기' 안내 메시지",
    "보드 뷰 상단에 owner-only assignmentGuideText 영역, 하단에 5×6 정형 격자 (CSS Grid + order: slotNumber)",
    "StaticSlotCard 컴포넌트 사용 (DraggableCard 아님 · react-dnd/dnd-kit import 0)",
    "썸네일 160×120 WebP 서버 리사이즈 + loading='lazy' + IntersectionObserver (원본 응답 금지)",
    "카드 클릭 시 전체화면 모달 오픈 (사이드패널 없음). 모달 닫기 시 DOM에서 즉시 언마운트",
    "submissionStatus 전이: assigned→viewed(옵션)→submitted→returned→submitted→reviewed",
    "재제출 매트릭스 서버 검증: not_graded+마감 전=in-place / 마감 후=allowLate 플래그 / graded·released=차단 / returned=허용(gradingStatus 리셋)",
    "반려(returned) 액션은 모달 내부에서만 가능 (격자 롱탭·컨텍스트 메뉴 금지). returnReason zod 1~200자 필수",
    "학생 모달 재진입 시 returnReason 상단 고정 배너 표시. 격자 뷰 returned 카드에 '!' 배지",
    "타 학생 slot 읽기 차단 — API 403 + DOM 미렌더 + RLS (3중 방어, E2E 테스트 필수)",
    "미제출 필터 → 일괄 독려 버튼 → 인앱 배지 알림 (외부 이메일·푸시 채널 없음, parent-viewer 주간 이메일과 분리 유지)",
    "Roster 동기화 버튼: 기존 slot 보존 + 신규 학생 slot 추가 + 삭제된 학생 slot은 submissionStatus='orphaned' 마킹(삭제 금지)",
    "학부모 뷰(PV-7 연동): 자녀 slot 단일 카드만. 격자 컴포넌트 자체 미렌더",
    "matrix/grid 뷰 v1 제외 (owner+데스크톱 전용 별도 라우트는 v2)",
    "갤럭시 탭 S6 Lite Chrome 실측 30장 격자 TTI < 3s, 1시간 사용 후 메모리 < 500MB",
    "WebSocket 채널 'board:${id}:assignment' 단일, 메시지 평균 < 200B + 100ms 디바운싱",
    "카드 표면 S-Pen 이벤트 리스너 0개 (필기는 모달 내부에서만 마운트)"
  ]
}
```

---

## 수행평가 자동채점 파이프라인 (assessment-autograde-roadmap.md, Seed 12)

> destination: **padlet**. 시드 `seed_0badf1e571bc` (ambiguity 0.10, interview_20260415_224854).
> 핵심 산출물: `Board.layout="assessment"` 확장 + 7 신규 엔티티(AssessmentTemplate·AssessmentQuestion·AssessmentSubmission·AssessmentAnswer·GradebookEntry·ProctorEvent·FeatureFlag) + Classroom·AssignmentSlot 확장. MCQ 결정론 매칭 + SHORT Gemini 2.5 Flash. 자동 화면이탈 잠금(Supabase 영속) + 교사 매트릭스(owner+데스크톱) + `/aura-web/gradebook` 학생·학부모 뷰 + Pro 전용 + `FeatureFlag.assessmentTierGate` 런칭 플래그.
> 블록을 padlet의 `tasks/{YYYY-MM-DD-slug}/phase0/request.json` 으로 복사해 feature 파이프라인에 투입.

### AA-1 Prisma 마이그레이션 (7 신규 엔티티 + Classroom·FeatureFlag 확장 + RLS 3분화)
```json
{
  "type": "feature",
  "title": "assessment-autograde — Prisma 마이그레이션 (7 신규 엔티티 + RLS 3분화)",
  "goal": "AssessmentTemplate·AssessmentQuestion·AssessmentSubmission·AssessmentAnswer·GradebookEntry·ProctorEvent·FeatureFlag 7 엔티티 + Board.layout='assessment' + Classroom.gradebookReleasePolicy·schoolManagedDevices 확장 + teacher/student/parent RLS 3분화 정책 일괄 적용",
  "context_refs": [
    "ideation/plans/assessment-autograde-roadmap.md#1-prisma-최종-확정-스키마",
    "ideation/plans/assessment-autograde-roadmap.md#5-송신-채널-d1공통-supabase--rls--realtime--pgmq",
    "ideation/tasks/2026-04-16-performance-assessment-autograde/phase4/seed.yaml",
    "ideation/tasks/2026-04-16-performance-assessment-autograde/phase3/decisions.md"
  ],
  "blocking_on": [],
  "acceptance": [
    "Board.layout enum에 'assessment' 추가 (freeform·event-signup·breakout·assignment와 공존)",
    "AssessmentTemplate.tierGate Boolean @default(false) + FeatureFlag 'assessmentTierGate' 런칭 플래그 존재",
    "AssessmentSubmission.isLocked Boolean @default(false) + lockedReason String? 영속 컬럼",
    "AssessmentSubmission @@unique([templateId, studentId])",
    "AssessmentQuestion @@unique([templateId, order])",
    "AssessmentAnswer @@unique([submissionId, questionId])",
    "GradebookEntry.releasedAt DateTime? nullable (teacher_manual 릴리스 정책)",
    "ProctorEvent.type enum: visibility_hidden | fullscreen_exit | focus_lost | teacher_manual | teacher_resume",
    "RLS: teacher=Classroom owner / student=본인 visibleToStudent / parent=ParentChildLink.status='active' + GradebookEntry.releasedAt IS NOT NULL 3분화",
    "unlock API RLS: teacher(Classroom owner) 또는 본인 student만 호출 허용"
  ]
}
```

### AA-2~AA-10 통합 (assessment-autograde feature 본체)
```json
{
  "type": "feature",
  "title": "assessment-autograde — 수행평가 자동채점 파이프라인 v1",
  "goal": "교사가 MCQ + SHORT 문항으로 AssessmentTemplate 출제 → 학생 응시 (IndexedDB + Supabase autosave 이중화, S-Pen 800×400 고정 캔버스) → MCQ 결정론 매칭 + SHORT Gemini 2.5 Flash 채점 (PGMQ 재시도) → 교사 매트릭스 뷰(owner+데스크톱)에서 확정·릴리스 → 학생·학부모 /aura-web/gradebook 실시간 공개. 화면이탈 시 자동 잠금(Supabase 영속) + 교사 '재개 승인' + 학생 '돌아왔습니다' 해제 2경로.",
  "context_refs": [
    "ideation/plans/assessment-autograde-roadmap.md",
    "ideation/plans/tablet-performance-roadmap.md#2b-최신-제약--수행평가-응시-화면-assessment-autograde-seed-12",
    "ideation/plans/assignment-board-roadmap.md",
    "ideation/plans/parent-viewer-roadmap.md",
    "ideation/tasks/2026-04-16-performance-assessment-autograde/phase4/seed.yaml",
    "ideation/tasks/2026-04-16-performance-assessment-autograde/phase3/decisions.md",
    "ideation/tasks/2026-04-16-performance-assessment-autograde/phase2/sketch.md",
    "ideation/tasks/2026-04-16-performance-assessment-autograde/phase1/exploration.md"
  ],
  "performance_budget_ref": "ideation/plans/tablet-performance-roadmap.md#2b-최신-제약--수행평가-응시-화면-assessment-autograde-seed-12",
  "blocking_on": ["AA-1 Prisma 마이그레이션 + RLS 3분화 선행"],
  "acceptance": [
    "AssessmentTemplate 생성 UI에서 MCQ·SHORT 2종만 문항 유형 드롭다운 노출 (OX·NUMERIC·ESSAY 차단, Zod validation gate)",
    "MCQ 채점 시 correctChoiceIds ↔ selectedChoiceIds 서버 결정론적 매칭, LLM 호출 없음 (회귀 테스트 LLM 호출 카운트 0)",
    "SHORT 채점 시 Gemini 2.5 Flash 호출, payload(modelAnswers+keywords+partialCredit) + template.rubricText + question.rubric 전체 프롬프트 주입",
    "partialCredit=false 시 이진(전부맞음/전부틀림) 채점, true 시 부분점수",
    "KR-SBERT 코사인 < 0.25 또는 10자 미만 답안 자동 0점(LLM 호출 스킵)",
    "화면이탈(Page Visibility API / fullscreenchange / focus_lost) 감지 시 즉시 isLocked=true + lockedReason 기록 + 교사 대시보드 Realtime 배지",
    "isLocked가 AssessmentSubmission DB에 영속화되어 새로고침 후에도 잠금 UI 복원 (E2E: 새로고침 후 잠금 지속 검증)",
    "학생 '돌아왔습니다' 모달 클릭 → POST /api/assessment/[submissionId]/unlock?by=student → isLocked=false 해제",
    "교사 '재개 승인' 클릭 → POST /api/assessment/[submissionId]/unlock?by=teacher&teacherId=X → Realtime broadcast로 학생 UI 즉시 해제 + ProctorEvent(type=teacher_resume) 기록",
    "잠금 상태에서 타이머 계속 진행 (endAt = startedAt + durationMin 고정), UI dim + '잠금 중 — 시간은 계속 흐릅니다' 텍스트",
    "ProctorEvent 전수 기록 (type, durationMs 포함)",
    "FeatureFlag.assessmentTierGate=false 시 전 사용자 접근, true 전환 시 Pro tier만 접근 (발급+수신 이중 재검증)",
    "교사가 '릴리스' 버튼 클릭 전까지 학생·학부모 GradebookEntry 비노출 (releasedAt IS NULL 응답 필터)",
    "릴리스 후 Supabase Realtime broadcast로 학생·학부모 뷰 p95 < 500ms 내 성적 표시",
    "PGMQ grading_retry 큐 2·4·8·16·32s exponential backoff, 5회 실패 시 status='retry_exhausted' 배지 + 교사 수동 재채점 경로",
    "SHORT 교사 UI 4필드 폼 (모범답안 필수≥1 / 키워드 선택 / 부분점수 체크박스 / 루브릭 자연어 선택)",
    "unlock API RLS: teacher(Classroom owner) 또는 본인 student만 호출 가능 (타 학생 unlock 403)",
    "AssessmentSubmission.status='submitted' 이후 PATCH 거부 (재응시 락 L3)",
    "동의서 미제출 학생 손글씨 입력 비활성(키보드만 허용)",
    "감사 로그 보관: Free=1학기, Pro=학년 + CSV/PDF export. 이후 Supabase cron 자동 파기",
    "갤럭시 탭 S6 Lite Chrome 응시 화면 TTI < 3s, 1시간 사용 후 메모리 < 500MB",
    "응시 화면 iframe 0 (DOM snapshot 검증, Tesseract.js import 0)",
    "S-Pen 캔버스 tldraw/perfect-freehand 800×400 고정 px, 60fps, throttle 없음",
    "문항 lazy 마운트 — 현재 ±1개만 DOM 유지 (30문항 일괄 렌더 금지)",
    "IndexedDB 로컬 draft + Supabase autosave 300ms debounce, localDraftHash 무결성 검증",
    "Realtime 구독 스코프: 학생=자기 submission / 교사 proctor 대시보드=templateId (보드 전체 ChangeFeed 금지)",
    "교사 매트릭스 뷰는 owner+데스크톱 전용 (editor·viewer·태블릿 차단, 학생 × 문항 > 30×30 시 react-virtual)",
    "학부모 뷰: ParentChildLink.status='active' + GradebookEntry.releasedAt IS NOT NULL 이중 조건만 노출 (parent-viewer v2 §5 매트릭스 승계)",
    "Promptfoo SHORT 채점 회귀 기준선 통과, /admin/llm-cost 관리자 전용 대시보드 동작"
  ]
}
```

---

## mallang-ranch-p2e (mallang-ranch-p2e-roadmap.md) — 외부 신규 P2E 프로젝트

> **주의**: 본 프로젝트는 padlet 파이프라인으로 가지 **않는다**. Seed 10은 외부 destination(`../mallang-ranch/INBOX/`)으로 dispatcher가 직접 배송하며, padlet phase0 진입 JSON 블록을 사용하지 않는다.
>
> phase0 request 포맷·acceptance·blocking_on 구조는 **외부 destination(`mallang-ranch`) 자체 진입 규약을 따르며, 본 ideation phase6 handoff에서 별도 템플릿으로 처리한다**. 본 파일 상위 블록(Seed 1~9)과 양식 호환 목적의 placeholder는 의도적으로 두지 않는다.
>
> 핵심 작업 묶음 참조: `plans/mallang-ranch-p2e-roadmap.md` §7 (9개월 마일스톤 M0–M9). 컨트랙트 6개·게임 루프·교배 시스템·Land 세일 단계별 작업 분해는 외부 저장소 자체 task tracker에서 관리한다.

---

## 사용 지침

1. 작업 시작 시 해당 블록을 대상 프로젝트의 `tasks/{YYYY-MM-DD-slug}/phase0/request.json` 에 복사.
   - Aura-board 작업(Seed 1~8): `padlet/tasks/...`
   - gongmun-assistant 작업(Seed 9, GM-*): `gongmun-assistant/tasks/...`
   - mallang-ranch 작업(Seed 10): **본 파일에 진입 JSON 미수록**. phase6 handoff가 외부 destination 자체 규약으로 처리.
2. `blocking_on` 항목이 있으면 선행 작업 완료 후 착수.
3. `context_ref` / `context_refs` 파일들은 planner(phase1)가 반드시 읽어야 함 — 누락 시 스코프 결정 부정확.
4. `acceptance`는 QA phase9의 수용 기준 매트릭스 시드로 사용.
5. 스키마 변경(PJ-1)은 마이그레이션 커밋 분리 권장.
6. `destination` 필드가 있는 블록은 **Aura-board 외부** 프로젝트이므로 padlet 저장소에 올리지 않도록 주의.
