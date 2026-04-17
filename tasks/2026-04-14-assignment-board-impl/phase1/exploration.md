# Phase 1 — Exploration: Roster-Bound Assignment Collection UX

- **task_id**: `2026-04-14-assignment-board-impl`
- **topic**: Aura-board에 학급 roster 기반 과제 수거 보드 신설 (학생별 카드 자동 인스턴스화, 제출/미제출 시각 구분)
- **탐색 축**: 라이선스·가격 · 학생 카드 자동 생성(roster join) · 제출/미제출 시각화 · 클릭→제출물 뷰 패턴 · 태블릿(갤럭시 탭 S6 Lite, Chrome Android) 적합성 · Aura-board 시너지
- **기준 단말**: 갤럭시 탭 S6 Lite (Chrome Android, S-Pen), 학급 N=30 카드 동시 렌더 가정

---

## 1. 후보 비교표

| 후보 | 라이선스/가격 | Roster → 학생 카드 자동 생성 | 제출/미제출 시각화 | 카드 클릭 → 제출물 뷰 | 태블릿 친화 (탭 S6 Lite) | Aura 시너지 |
|---|---|---|---|---|---|---|
| **① Google Classroom** | 상용 SaaS · 교육용 무료 (Workspace for Edu) · 비공개 | 과제 1건 생성 시 enrolled students N만큼 `StudentSubmission` 리소스 자동 인스턴스화 (공식 API 확인) | `Turned in` / `Assigned` / `Missing` / `Graded` / `Returned` 상태 라벨·카운트·필터 | 학생 이름 클릭 → 제출물 단일 뷰(사이드패널+본문) | 상(Android 네이티브 앱+모바일 웹 최적화) | 데이터 모델 직접 차용 가능 — **per-student submission slot = card** 매핑이 우리 요구와 1:1 |
| **② Microsoft Teams Assignments (for Education)** | 상용 SaaS · A1 Edu 라이선스 무료 tier · 비공개 | Team(class) 멤버십 기반, 과제 생성 시 roster 전체에 자동 배부 | grades 그리드: 행=과제, 열=학생 / 상태 셀(Turned in, Viewed, Returned, Past Due, Ready to Grade) | 그리드 셀 클릭 → 우측 grading pane (학생 1명 제출물 집중 뷰) | 중(데스크톱 최적, 태블릿 Chrome은 Teams 웹 렉 보고 다수) | grades 그리드 모델은 우리 matrix 뷰(owner+데스크톱 전용) 설계에 거의 그대로 적용 가능 |
| **③ Seesaw** | 상용 SaaS · Free tier + Plus 구독 · 비공개 | Activity(과제) 발행 시 class roster 전원에게 response slot 자동 생성, Activities View에 student×activity 매트릭스로 표시 | **체크마크 그리드** (학생 행 × 과제 열에 완료 체크·미완료 공란, 색상 힌트) | 셀 hover/click → activity 상세 + 개별 student response 포스트 | 상(K-5 중심 iPad/Android 태블릿 퍼스트 디자인) | **카드형 제출물 미리보기 UX**(썸네일·필기·음성 노트) — 학생(editor) 제출 카드의 시각 언어 참고 가치 큼 |
| **④ Canvas LMS (Instructure)** | 듀얼 라이선스 · Core는 AGPLv3 오픈소스(GitHub `instructure/canvas-lms`) · 클라우드 유료 | 과제 생성 시 course enrollment 기반 SpeedGrader에 학생 리스트 자동 주입 | SpeedGrader 상단 드롭다운으로 `Hasn't submitted yet` 등 상태 필터 · missing submission policy로 자동 0점 | 학생 선택 → SpeedGrader 전용 full-page (좌측 제출물, 우측 rubric) | 하~중 (SpeedGrader는 데스크톱 전용 최적, 태블릿 rubric 렌더 무거움) | 오픈소스 코드 참조 가능 — 특히 `Submission`·`Enrollment` 모델 스키마가 Aura의 Card·editor-권한 설계 벤치마크로 유용 |
| **⑤ Moodle Assignment** | **GPLv3 오픈소스** (공식 docs 하단 명시) · 자가 호스팅 무료 | Assignment 모듈 배치 후 `View all submissions` 그리드에 course enrolled users 자동 전원 출력 | Grading Table 열: Submission status(`No submission`/`Draft`/`Submitted`) · Grading status(`Not graded`/`Graded`/`Released`) · 상태별 필터 | 행 클릭 → 학생별 grading page (annotation + feedback) | 하 (Grading Table은 wide table, 태블릿 가로스크롤·줄바꿈 이슈 다수) | 오픈소스·다국어 성숙. 단 UI는 Aura의 보드형 UX와 톤 불일치 — 데이터 모델만 레퍼런스 |
| **⑥ Padlet "Submission" board (baseline)** | 상용 SaaS · 무료 tier + Backpack Edu 구독 | **Roster join 없음** — 학생이 익명 또는 자기 이름으로 수동 포스트. 링크 공유 방식 | 포스트 유무로만 판별. "누가 안 냈는지" 직관 지표 없음 · Gradebook 뷰에서 student별 post 그룹핑 가능(구독 필요) | 포스트 클릭 → 모달 확대 뷰 | 중(Padlet 자체 태블릿 렉 보고 — MEMORY 참조) | 현 Aura-board baseline. **우리가 해결하려는 갭 그 자체** — roster 바인딩 부재가 페인포인트 |
| **⑦ 트라이디스 (TryThis, trythis.co.kr)** ★사용자 경험 보유★ <br>⚠️ **2026년 초 UI 개편으로 현행 버전 ≠ 사용자 선호 버전. 현재 스크린샷/UX는 참조 시 주의** (사용자 선호는 개편 이전 UX 기반, 현행 UI·블로그 캡처로 설계 결정 역추적 금지) | 한국 교사 자작 SaaS · 비공개 소스 · **무료 tier**(수업 3·페이지 10·콘텐츠 5 제한) + 유료 무제한. 팀 스페이스로 학교 단위 공유 | **클래스룸 + 학급 일괄 등록**: "학급 정보 일괄 등록으로 모든 팀 클래스에서 일일히 학생 명단 만들 필요 없음". QR 로그인으로 학생 인증 → 보드 제출 이력 자동 기록 (roster↔제출 자동 바인딩) | 보드 기능에 **교사 영역 vs 학생 영역 명확 분리** — 교사는 가이드 포스트로 평가 방법 제시, 학생은 자기 영역에 산출물 제출. 패들렛과 달리 "평가를 위해 최적화된" 수합 뷰 (공식 블로그 표현) | 보드 내 학생 산출물 → 교사가 "한큐에 걷고 평가" → 포트폴리오(홈 → 마이스페이스 → 포트폴리오) 자동 수집, 드래그로 보드 그룹핑 | 상(한국 초등 교사 주 타겟, Chrome Android + 원클릭수업 공유 링크 UX 최적화) | **매우 높음** — "보드"라는 동일 은유 사용 + Padlet 대비 학생별 영역 분리·roster 일괄 등록·무제한 보드 개설 등 Aura가 해결하려는 갭을 이미 검증된 방식으로 풀고 있음 |

### 참고: 기준 단말 성능 맥락
- 갤럭시 탭 S6 Lite는 Helio P22T(2020)·4GB RAM 저사양. Padlet·Canva 모두 이 기기에서 스크롤·입력 지연 보고됨(프로젝트 MEMORY).
- N=30 카드 동시 DOM 렌더 시, 썸네일·S-Pen 필기 캔버스가 카드에 직접 얹히면 스크롤 프레임 드롭 위험 → **썸네일 가상화(virtualized list) + 클릭 시 lazy-load 모달** 패턴이 안전.
- Classroom·Seesaw는 모바일 네이티브 앱으로 이 문제를 피한다. 웹 기반 Aura는 **카드 경량화 설계**가 필수.

---

## 2. 후보별 장단점·Aura 적합 지점

### ① Google Classroom
- 장점 1: `StudentSubmission` auto-instantiation 모델이 표준화·검증됨 (Google Workspace Developers 공식 "each assignment is associated with N student submissions")
- 장점 2: 상태 enum (`Turned in`/`Missing`/`Graded`/`Returned`)이 교사 멘탈모델에 안착, 용어 차용 시 교사 러닝커브 최소
- 단점 1: 비공개 소스 — UI 세부·렌더 최적화는 역설계 불가
- 단점 2: 보드형 UX가 아닌 리스트형 — Aura의 공간적 카드 보드와 은유 불일치
- **Aura 적합 지점**: **데이터 모델 (1 Assignment → N StudentSubmission)과 상태 enum**을 그대로 차용. UI는 Aura 카드 은유로 번역

### ② Microsoft Teams Assignments
- 장점 1: grades 그리드(행=과제, 열=학생)는 owner+데스크톱 전용 matrix 뷰에 그대로 적용 가능한 패턴
- 장점 2: `Viewed` 상태(학생이 열었으나 미제출)를 별도 시각화 — 미제출 독려 정밀도 향상
- 단점 1: Teams 웹은 탭 S6 Lite에서 무거움 보고 다수 → 동일 접근 피해야
- 단점 2: 학생별 카드 개별화가 약함(그리드 셀 단위), Aura의 "카드=자기 자리" 은유에 약간 어긋남
- **Aura 적합 지점**: `Viewed` 같은 **중간 상태 아이디어** 차용 ("학생이 카드 열어봄" 배지로 독려 타이밍 조절)

### ③ Seesaw
- 장점 1: K-5 학년 저학년 포함 **학생 친화 UX**가 업계 최고 — 학생(editor)이 직관적으로 자기 카드를 찾음
- 장점 2: Activities View는 교사에게 **체크마크 매트릭스**(student × activity)를 제공, 미완료 공란 즉시 시각화
- 단점 1: 비공개 소스·가격 정책 변경 논란 있음 (참고만)
- 단점 2: 활동 단위 설계가 강해, 하나의 보드로 여러 과제를 통합하는 Aura식 "보드=과제 1건" 패턴과는 스케일 단위 다름
- **Aura 적합 지점**: **학생 카드의 시각 언어** — 썸네일·체크마크 배지·제출 완료시 색상 전환 — 을 Aura 카드 템플릿에 이식

### ④ Canvas LMS
- 장점 1: **AGPLv3 오픈소스** — `Submission`·`Enrollment` 스키마를 GitHub에서 직접 참조 가능 (Aura ERD 설계 시 벤치마크)
- 장점 2: `missing submission policy` — 마감 후 미제출 자동 처리 로직 검증된 사례
- 단점 1: SpeedGrader는 데스크톱 full-page 패러다임, 태블릿 UX는 Aura 참고 가치 낮음
- 단점 2: LMS 전체가 무거운 프레임 — 라이트한 보드형 Aura에 과도
- **Aura 적합 지점**: **오픈소스 데이터 모델 참조** + missing policy 아이디어(마감 후 상태 자동 전이)

### ⑤ Moodle Assignment
- 장점 1: **GPLv3 오픈소스** (Moodle docs 공식 확인) — 코드·UX 제약 없이 조사 가능
- 장점 2: `Submission status` + `Grading status` **이원 상태 모델** — Aura에서 제출 상태(editor 액션)와 교사 리뷰 상태(owner 액션)를 깔끔히 분리할 수 있는 레퍼런스
- 단점 1: Grading Table은 wide table UI — 태블릿·모바일 적합성 최악, UI는 차용 불가
- 단점 2: UX가 1990년대 wiki 톤 — Aura의 보드 은유와 불일치
- **Aura 적합 지점**: **이원 상태 모델** (submission_status × grading_status)을 카드 배지 2종으로 번역

### ⑥ Padlet baseline
- 장점 1: 현재 Aura-board가 서는 출발점 — 변경 비용 계산 용이
- 장점 2: 자유 포스트 포맷(텍스트·이미지·첨부·링크) 유연성 그대로 계승 가능
- 단점 1: **Roster join 부재 = 본 과제의 원인** — "누가 안 냈는지" 파악 불가
- 단점 2: 1인 다포스트 가능 → 과제 제출의 "1인 1슬롯" 요건과 충돌
- **Aura 적합 지점**: 포스트 콘텐츠 포맷은 계승, **slot 1:1 제약 + roster auto-binding**은 신규 보드 타입으로 추가

### ⑦ 트라이디스 (TryThis) ★사용자 경험 보유★
> **사용자 피드백**: "트라이디스 썼었는데 마음에 들었었어" — 실사용 후 선호 표명. 경험 기반 preference signal. Aura-board의 교사 플로우는 트라이디스에 **근접한 사용감**을 제공해야 품질 기준을 만족함.
- 장점 1: **"보드" 은유를 Padlet과 공유하면서도 교사 영역/학생 영역을 명확히 분리** — Aura처럼 owner(교사)/editor(학생) 권한 모델을 보드 레이아웃에 1:1 투영한 선례. 가이드 포스트(교사) ↔ 제출 영역(학생별)의 시각적 분리가 본 과제 요구와 정확히 일치
- 장점 2: **학급 일괄 등록 + QR 로그인으로 제출 자동 바인딩** — roster join 부재라는 Padlet 갭을 실전에서 해결한 검증된 UX. "일일히 학생 명단 만들 필요 없음"은 교사의 반복 페인을 제거한 핵심 가치
- 장점 3: **포트폴리오 자동 수집** — 학생별 산출물이 시간축으로 누적되는 구조, Aura의 장기 학급 운영에 이식 가치 큼 (학기말 평가·학부모 공유 시나리오)
- 장점 4: **한국 교사 타겟, Chrome Android 최적화** — 갤럭시 탭 S6 Lite에서 실사용 레퍼런스 다수, 태블릿 성능 제약을 실전 수준에서 해결
- 단점 1: 비공개 소스 — 구현 디테일은 역설계 불가, 행동 관찰로만 차용 가능
- 단점 2: 무료 tier 제한이 엄격(수업 3·페이지 10·콘텐츠 5) — 가격 모델 자체는 Aura에 부적합, 기능 UX만 차용
- 단점 3: SNS형 교사 간 공유는 Aura 스코프 밖 — 이 부분은 차용하지 않고 Aura는 학급 단위 중심 유지
- **Aura 적합 지점**:
  - **보드 레이아웃 패턴**: 교사 가이드(상단 고정 영역) + 학생별 카드 그리드(자기 영역). Padlet의 자유 배치와 Classroom의 리스트 사이 중간점
  - **학급 일괄 등록 UX**: 학급 roster 한 번 등록 후 보드마다 재사용 — Aura의 `roster_id` 외래키 설계에 UX 레이어로 이식
  - **포트폴리오 탭**: 학생 개인 뷰에서 자기 제출 이력 누적 확인 (editor 권한의 "내 카드 히스토리")
  - **원클릭수업 공유 링크**: QR/단축 URL로 저마찰 학생 접속 — MEMORY "저마찰 플로우" 원칙과 정합

---

## 3. 1순위 권고

### 3.1 북극성 (North-Star) — 5개 UX Primitive

> **출처**: 사용자 경험 기반 선호 (2026년 이전 트라이디스). 사용자가 직접 사용해보고 좋았다고 표명한 요소를 언어화한 결과.
> **중요 — 참조 소스 제약**: 현재 트라이디스는 2026년 초 UI 개편으로 과거 버전과 다를 수 있음. 따라서 **현행 트라이디스 UI·스크린샷·블로그 캡처를 설계 결정의 참조 소스로 삼지 말 것**. 북극성은 아래 5개 primitive의 서술만을 정규 소스로 인정하며, 현행 트라이디스는 "사용자가 선호했던 버전과 다른 별개 제품"으로 취급한다. 유사 구현 차용은 아래 1:1 매핑된 공개 레퍼런스(Classroom/Seesaw/Moodle/Canvas)에서 수행한다.

**Primitive 1 — 학급 로스터 기반 카드 자동 생성**
보드 생성 시 학급 roster를 선택하면 학생 N명만큼 카드가 자동 인스턴스화. 학생을 추가/제거하면 카드도 자동 동기화.
→ **유사 구현 차용처**: **Google Classroom의 `StudentSubmission` auto-instantiation 모델** (1 Assignment → N enrolled students에 대한 submission 리소스 자동 생성, API 공식 문서 검증됨). Aura의 `AssignmentBoard × ClassRoster → N StudentSubmissionSlot` 스키마의 정규 벤치마크.

**Primitive 2 — 제출/미제출 시각 구분 (색상·배지)**
카드 표면에서 한눈에 상태 판별. 제출 완료 = 채워진 시각 신호 / 미제출 = 공란 + 옅은 경고 / 중간 상태 = 차별 배지.
→ **유사 구현 차용처**: **Seesaw의 체크마크 그리드 + 썸네일 채움/공란 시각 언어**. 추가로 **Moodle의 이원 상태(`Submission status` × `Grading status`)**를 배지 2종으로 겹쳐 owner(교사) 리뷰 상태까지 표현.

**Primitive 3 — 카드 클릭 → 해당 학생 제출물 뷰 (모달/사이드패널)**
카드 탭/클릭으로 학생 1명의 제출물 집중 뷰 진입. 전체 페이지 전환이 아닌 모달/사이드패널로 컨텍스트 유지.
→ **유사 구현 차용처**: **Google Classroom의 "학생 이름 클릭 → 제출물 단일 뷰" 사이드패널 패턴**. Canvas SpeedGrader는 full-page이므로 태블릿 부적합 — Classroom의 모달형을 차용. 컨텐츠 렌더는 lazy-load.

**Primitive 4 — 교사 가이드 영역 + 학생 영역 분리 (상단 교사, 하단 학생 카드)**
보드 상단은 교사(owner) 전용 가이드 포스트(과제 설명·마감·제출 방법 샘플·평가 기준). 하단은 학생별 카드 그리드. 권한·레이아웃 모두 분리.
→ **유사 구현 차용처**: **Google Classroom의 Classwork 상단 안내문(instructions) + 하단 student work 리스트 2단 구성**. Aura는 이를 "보드 은유"로 번역하되 위/아래 영역 분리는 Classroom과 동일한 owner-only/editor 권한 경계를 따른다.

**Primitive 5 — 정형 격자 레이아웃, 번호순 5×6 (자리표처럼 고정 위치)**
자유 배치가 아닌 **번호순 고정 그리드** (5열 × 6행 = 30 카드). 학생 번호가 곧 카드 위치 → 교사가 자리표처럼 공간 기억으로 찾음.
→ **유사 구현 차용처**: **Moodle `View all submissions` grading table의 고정 행 순서**(학생 번호·성 기준 결정적 정렬)를 격자 2D 좌표로 번역. Seesaw Activities View의 student×activity 매트릭스도 결정적 정렬 사례로 보조 레퍼런스. CSS Grid `grid-template: repeat(6, 1fr) / repeat(5, 1fr)` + `order: {student_number}` 로 구현.

### 3.2 태블릿 제약 평가 — 5×6 정형 격자 30 카드, 갤럭시 탭 S6 Lite

정형 격자 30 카드는 **virtualization 없이도 렌더 가능성 높음**. Helio P22T · 4GB RAM 기준, DOM 노드 30개 + 각 카드가 학생명·상태 배지·썸네일 1장으로 제한되면 초기 레이아웃·스크롤 비용은 감내 가능 수준이며, 5×6 격자는 대부분 뷰포트 내에 들어와 스크롤 자체가 짧다(탭 S6 Lite 세로 1200px 기준 한 번의 스크롤로 전체 커버). 단 **결정적 위험은 썸네일 비용**: 카드당 원본 이미지 직렬 로드 시 30장 동시 디코딩으로 메모리 스파이크·Jank 발생 가능. 따라서 (a) 썸네일은 서버측 리사이즈된 저해상도(예: 160×120 WebP) 단일 장만 카드에 싣고, (b) `loading="lazy"` + `IntersectionObserver`로 뷰포트 밖 카드(특히 4~6행) 디코딩을 지연시키며, (c) S-Pen 필기 캔버스는 카드 표면이 아닌 클릭 후 모달에서만 렌더, (d) 카드 자체는 순수 CSS 상태 토글(제출/미제출 색상)로 React 리렌더 최소화 — 이 네 가지만 지키면 virtualization 도입 없이도 60fps 근접 달성 예상. N이 40을 넘거나 썸네일이 여러 장 필요해지는 고학년 시나리오가 생기면 그 시점에 `react-virtuoso` 등 도입 재검토.

### 3.3 하이브리드 권고 요약

**UX 북극성 = 위 5개 primitive 고정**. 데이터 모델·구현 패턴은 공개 레퍼런스에서 1:1 매핑대로 차용:
- **Primitive 1** ← Google Classroom `StudentSubmission` 자동 인스턴스화 (데이터 모델)
- **Primitive 2** ← Seesaw 썸네일·체크 시각 언어 + Moodle 이원 상태 배지
- **Primitive 3** ← Google Classroom 학생별 사이드패널 (모달 lazy-load)
- **Primitive 4** ← Google Classroom Classwork 상단 안내 + 하단 work 리스트 2단 구성
- **Primitive 5** ← Moodle grading table 결정적 정렬 + CSS Grid 고정 배치

matrix 뷰(owner+데스크톱 전용)는 **Teams식 grades 그리드**를 별도 레이어로 제공(MEMORY 제약 준수). Padlet은 baseline으로 포스트 콘텐츠 포맷만 계승. 현행 트라이디스는 참조하지 않는다.

**Phase 2에 넘길 핵심 결정 후보**:
1. `AssignmentBoard` 보드 타입 신설 vs 기존 Board에 `type` 컬럼 추가 — 보드 은유 통일 위해 기존 Board 확장 유력
2. `StudentSubmissionSlot` 엔티티 = 카드 1:1 매핑 여부 (vs 기존 Card 재사용 + `owner_student_id` 추가). Primitive 1·5 구현에는 slot 엔티티가 더 자연스러움
3. 상태 enum: `assigned | viewed | submitted | returned | graded` (Moodle식 이원 분리 여부) — Primitive 2 배지 체계 직결
4. **학급 roster 엔티티 승격** — Board 생성 워크플로의 선행 조건으로 `ClassRoster` 퍼스트클래스 엔티티화 (Primitive 1 차용)
5. **교사 가이드 영역** — Board의 고정 섹션(고유 슬롯)으로 모델링할지, 특수 Card 타입(일반 Card의 `role=guide`)으로 모델링할지 (Primitive 4)
6. **5×6 격자 고정 규약** — 학생 번호 1~30을 `(row, col) = (⌈n/5⌉, ((n-1) mod 5)+1)`로 결정화할지, 교사가 좌석 배치 커스터마이즈 가능하게 할지 (Primitive 5)
7. 카드 클릭 뷰 = 모달 vs 사이드패널 (Primitive 3) — 탭 S6 Lite 가로폭에서 사이드패널은 좁아질 수 있어 전체화면 모달 우세 가설
8. 썸네일 파이프라인 — 서버측 리사이즈 단일 저해상도 규약 (태블릿 제약 문단 결론)

---

## 4. 참조 링크

- [Google Classroom — View all your students' work (공식 도움말)](https://support.google.com/edu/classroom/answer/9157286)
- [Google Classroom API — Assignment workflows (StudentSubmission auto-instantiation 공식)](https://developers.google.com/workspace/classroom/tutorials/assignment-workflows)
- [Microsoft Teams — View and navigate your assignments (educator, 공식)](https://support.microsoft.com/en-us/topic/view-and-navigate-your-assignments-educator-9b20d2b4-a465-4136-9621-d09af2621c2d)
- [Seesaw — Using the Activities View in the Gradebook](https://help.seesaw.me/hc/en-us/articles/360060511831-Using-the-Activities-View-in-the-Gradebook)
- [Canvas LMS — GitHub `instructure/canvas-lms` (AGPLv3 오픈소스 저장소)](https://github.com/instructure/canvas-lms)
- [Moodle — Assignment activity (공식 docs, GPLv3 표기 확인)](https://docs.moodle.org/501/en/Assignment_activity)
- [Padlet — Gradebook and grade passback (baseline 참고)](https://padlet.blog/gradebook-and-grade-passback/)
- [트라이디스 (TryThis) 공식 홈 — 교사 자작 온라인 수업 플랫폼](https://trythis.co.kr/)
- [트라이디스 블로그 — 보드 기능·포트폴리오·팀 클래스 구조](https://slashpage.com/trythis/wy9e1xp2xz5yv27k35vz?lang=ko)
- [티처빌 — 교사들의 수업 저작/공유 플랫폼 트라이디스 소개](https://www.teacherville.co.kr/trythis/contents/13357.edu)
- [AskEdTech — 트라이디스 제품 카탈로그](https://www.askedtech.com/product/499679)
- [에듀넷 T-클립 — 트라이디스 플랫폼으로 온-오프라인 수업 콘텐츠 제작하기 연수](https://educator.edunet.net/local/ubmooc/view.php?id=780)
