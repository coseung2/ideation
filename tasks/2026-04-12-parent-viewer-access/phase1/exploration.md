# Phase 1 — Parent/Viewer Access 외부 패턴 조사

- task_id: `2026-04-12-parent-viewer-access`
- 기준 단말: 교사/학생 = 갤럭시 탭 S6 Lite (10.4", 세로 1200x2000), **학부모 = 스마트폰 세로 (390~412px 뷰포트)**
- 핵심 제약: 한국 초·중등 교실 현장, 자녀 1인 한정 열람(read-only), 프라이버시(양육권 변경·탈퇴 시 철회), 모바일 우선, Seed 1(그림보드)·Seed 4(식물관찰일지) 기결정 권한 상위화

---

## 1. 조사 후보 비교표

| # | 제품 | 인증·연결 방식 | 학부모-자녀 매핑 | 열람 범위 | 쓰기 권한 | 모바일 UX | 라이선스/가격 | 한국 현장 친화성 |
|---|------|--------------|----------------|---------|---------|----------|--------------|----------------|
| 1 | **ClassDojo Parent** | 교사 발급 **Parent Code (8~9자리, 30일 만료, 4회 사용 제한)** 또는 이메일/SMS 초대 링크 | 1:N(복수 보호자 OK), 교사 승인 필요 | Points 2주치 + Class Story + 승인된 Portfolio 글 | 댓글·리액션(교사 설정에 따라) | 전용 iOS/Android 앱, 알림 푸시 강함 | Freemium SaaS | 글로벌, 한국 공식 번역은 있으나 학교 도입 낮음 |
| 2 | **Google Classroom Guardian Summaries** | 교사/관리자가 **이메일 초대**, 학부모 Google 계정으로 수락 (120일 유효) | 1:N, 도메인 관리자 통제 | 주/일 이메일 요약(미제출·예정·공지), 2024+는 과제 미리보기도 가능 | 없음 (순수 read via email) | 웹·Gmail 중심, Classroom 앱에 guardian 뷰는 제한적 | 무료(Workspace for Education) | 중·고 일부 도입, 학생 Workspace 계정 필수 → 초등에서는 계정 발급 허들 |
| 3 | **Seesaw Family** | 교사가 **이메일/QR코드/공유 초대 링크** 발송 | 1:N, 학부모 1명이 복수 자녀 피드 통합 | 자녀 개인 Journal 전체(교사 승인 게시물) + 해당 자녀 태그 그룹 게시물만 | 좋아요·댓글(교사 모더레이션) | **Family 전용 앱(iOS/Android)** 매우 강함, 푸시 알림 기본 | Freemium SaaS | 초등 포트폴리오에 최적, 국내 사립·국제학교 사례 있음 |
| 4 | **Canvas LMS Observer** | **Pairing Code (6자리)** 학생 생성→학부모 입력, 또는 관리자 수동 연결 | 1:N, 보호자가 여러 학생 관찰 가능 | 과정 전체 과제·성적·이벤트·발표(코스 단위) | 제출 불가, 일부 뷰 접근만 | Canvas Parent 앱 있지만 태블릿·PC 중심, 스마트폰은 기능 축약 | 오픈소스 Core(AGPLv3) + 유료 Cloud | 대학·국제학교 위주, 초중 교실 경량 사용에 과함 |
| 5 | **클래스팅(Classting)** | 교사가 **클래스 초대 코드** 발급 → 학부모·학생이 입력(학생은 보호자 연락처로도 가입 가능) | 학급 단위 가입, 자녀 명시 매핑은 별도 프로필로 표기 | 학급 피드, 알림장, 과제 공지, (AI 요금제는 학습분석 리포트) | 댓글·메시지 (교사가 일방향 설정 가능) | 모바일 앱 중심, 한국 UX 최적화 | Freemium SaaS, AI 요금제 유료 | **한국 초·중 보편 보급**, 교사 친숙도 최고 |
| 6 | **하이클래스(Hi-Class)** | 학교/교사 발급 가입 경로 + 학부모 전화번호 기반 초대, **하이콜**로 교사-학부모 전화 중계 | 학급 단위, 학부모 계정 자녀 매핑 | 알림장, 가정통신문, 앨범, 일정 | 알림 확인·일방향 회신 | 모바일 앱 중심, 푸시·수신확인 강점 | 무료(아이스크림미디어, 교육부 지원) | **공립 초등 대량 도입**, 교사 업무용 표준에 가까움 |

(계약서의 ≥ 4 비교표 기준 초과 달성)

---

## 2. 후보별 장단점 (각 ≥ 2)

### 1. ClassDojo Parent Code
- **장점**
  - 30일 만료 + 4회 재사용 제한으로 **토큰 남용 방지**가 제품 레벨에서 명시 — 우리 프라이버시 요건(양육권 변경 시 철회)에 직접 이식 가능
  - 이메일 없이 종이 배포용 코드만으로도 동작 → 저학년 가정 진입 장벽 낮음
- **단점**
  - 학부모가 반 Class Story 전반을 보게 되어 **타 학생 얼굴·작품 노출 리스크** — 우리 요구(자녀 1인 한정)와 결이 다름
  - 교사 승인 게시물만 노출되는 구조라 보드 실시간성이 낮음 (우리 Seed 1·4는 실시간 자녀 열람 기대)

### 2. Google Classroom Guardian Summaries
- **장점**
  - **이메일 단방향 요약**이라 모바일 OS·앱 설치 부담 제로 — 부모 스마트폰 기종·리터러시와 독립적
  - 학부모는 로그인 없이 수동 수신, 개인정보 최소 저장 (이메일 주소만) — FERPA/개보법 친화
- **단점**
  - **콘텐츠 뷰잉 기능 없음** (요약 위주) → 우리가 약속한 "그림보드·식물관찰일지 산출물 열람"이 불가
  - 학생에게 Workspace 계정이 있어야 함 → 초등 저학년·공립 일반 현장 부적합

### 3. Seesaw Family
- **장점**
  - **"해당 자녀 태그된 것만 보인다"** 는 규칙이 명시 — 우리의 "자녀 1인 한정 열람" 요구와 **가장 정밀히 일치**
  - QR + 이메일 + 링크 **3가지 초대 채널**을 교사가 골라 쓸 수 있어 현장 유연성 높음
- **단점**
  - Family 앱 설치 유도 UX → 한국 학부모 중 앱 설치 거부(저장공간·개인정보) 비율 고려 필요
  - SaaS 의존, 자체 호스팅 불가 → Aura-board 원칙(자체 호스팅·격리)과 정면 충돌

### 4. Canvas LMS Observer + Pairing Code
- **장점**
  - **Pairing Code 6자리**를 학생 본인이 생성 → "자녀가 부모에게 코드 전달" 패턴이 자연스러워 **교사 개입 최소화**
  - Observer는 제출 불가·열람만이라는 **RBAC 분기 성숙도 최고** (우리 read-only 요건 그대로 차용 가능)
- **단점**
  - 코스/학기 단위 모델이라 **초등 학급 보드** 단위로는 과도하게 무거움
  - 모바일(Canvas Parent 앱)에서 기능 축약이 크고, 스마트폰 세로 UX가 약함

### 5. 클래스팅
- **장점**
  - **한국 교사 친숙도 1위급** — 교사 초대·가입 마찰이 현장 실증으로 낮음
  - 학생 개인정보가 없어도 보호자 연락처로 우회 가입 지원 → 저학년 초기 마찰 감소
- **단점**
  - 학급 단위 피드라 **"자녀 1인 한정 열람" 세분화 부재** → Aura-board의 권한 상위화 요구 미충족
  - 교사 메시지·공지 중심이라 보드 산출물(그림·관찰일지) 뷰잉 UX가 1급 시민이 아님

### 6. 하이클래스
- **장점**
  - **공립 초등 교사 사실상 표준** — 학부모들이 이미 앱 설치·로그인 상태인 비율이 높음 → 별도 앱 설치 요구 회피
  - 수신확인·미확인자 재알림 기능 → 우리 공지/열람 독촉 UX에 직접 참고
- **단점**
  - 알림장·가정통신문 중심으로, **자녀 창작 산출물 포트폴리오 뷰**는 주력 기능이 아님
  - API·연동이 사실상 폐쇄 → Aura-board가 차용할 수 있는 것은 **UX 패턴뿐**, 기능 재사용 불가

---

## 3. 공통 패턴 요약 (Aura-board에 이식할 설계 원칙)

1. **교사 발급 단기 토큰** + **만료(7~30일) + 사용횟수 제한** — ClassDojo·Canvas 공통. 양육권·탈퇴 시 철회 요건에 필수.
2. **학부모-자녀 N:M 매핑** 허용 — 복수 보호자, 한 보호자의 복수 자녀 모두 현실. Seesaw·Canvas·ClassDojo가 공통 지원.
3. **Read-only RBAC을 제품 레벨 role로 분리** — Canvas Observer처럼 `BoardMember.role = 'parent'`를 신설하고 write·comment·share를 **모든 보드 타입에 일괄 차단**.
4. **자녀-스코프 필터링을 쿼리에 강제** — Seesaw의 "태그된 것만" 모델. 서버에서 `WHERE student_id IN (parent.children)`을 RLS 수준으로 고정.
5. **초대 3채널(QR / 이메일 / 링크)** — 한국 학부모의 이메일 미사용 비율 감안 → QR·SMS 링크 우선.
6. **모바일 세로 전용 레이아웃** — 갤럭시 탭 기준 외에 **390px 세로** breakpoint를 신규로 보드 뷰어에 추가.

---

## 4. 1순위 권고

### **ClassDojo Parent Code 방식 + Seesaw 자녀-스코프 필터 하이브리드**

**왜 이 조합인가 (한국 초·중 교실 + 프라이버시 + 모바일 우선 관점)**

- **인증·초대 레이어 = ClassDojo 방식 차용**
  - 교사가 학급 페이지에서 **"학부모 초대 코드" 8~9자리**를 학생별로 발급, **30일 만료 + 4회 사용 제한**. 한국 현장은 이메일 미보유 학부모(특히 조부모 양육) 비중이 높아 **종이·문자 배포 가능한 단기 숫자 코드**가 Google Classroom 이메일 방식보다 마찰이 낮다.
  - 양육권 변경·탈퇴 철회는 **토큰 revoke + 재발급** 1액션으로 해결 → 프라이버시 요건 충족.

- **권한·쿼리 레이어 = Seesaw 자녀-스코프 강제**
  - 서버 쿼리에 `student_id ∈ parent.children` 필터를 **RLS/미들웨어 레벨에서 하드코딩**. 교사가 실수로 학부모를 학급 전체에 노출시킬 경로 자체를 차단 → 타 학생 프라이버시 보호.
  - Seed 1·Seed 4가 결정한 "공유 여부 무관 자녀 것은 열람"을 이 레이어에서 **isPrivate 토글 상위 권한**으로 자연스럽게 구현.

- **RBAC 역할 = Canvas Observer 차용**
  - `BoardMember.role` 에 `'parent'` 신설(기존 memory 메모 확정). Observer처럼 **read-only를 제품 레벨 역할**로 분리해 모든 보드(그림·관찰일지·향후 다른 시드)에 **cross-cutting** 하게 적용.

- **모바일 UX = 하이클래스·Seesaw Family 벤치마크**
  - 학부모 라우트 `/parent/*` 에 **390px 세로 전용 레이아웃**을 기본 적용(태블릿 뷰 상속 금지). 자녀 피드 타임라인(세로 스크롤, 썸네일 1열), 상세 오버레이는 풀스크린 시트. 하이클래스 수신확인 UX를 참고해 "열람 스탬프"로 교사에게 암묵적 피드백.
  - PWA 우선(설치 불필요) → 한국 학부모 앱 설치 거부율 회피.

- **거부한 대안 이유 요약**
  - Google Classroom Guardian Summaries: 콘텐츠 뷰잉이 없어 **핵심 요구인 산출물 열람**을 만족 못 함.
  - 클래스팅/하이클래스: API 폐쇄 SaaS라 자체 호스팅 Aura-board와 **통합 불가**, UX 패턴만 차용.
  - Canvas Observer 단독: 모델이 과하게 무겁고 모바일이 약함.

**결론**: 초대·철회 UX는 ClassDojo, 필터·스코프는 Seesaw, 역할 모델은 Canvas Observer, 모바일 세로 UX는 하이클래스/Seesaw Family. 이 네 출처를 합성해 자체 구현.

---

## 5. 참조 링크 (공식 문서 ≥ 2)

1. [ClassDojo — Connecting to Your Child's Class via a Parent Code (공식 헬프)](https://help.classdojo.com/hc/en-us/articles/202047699-Connecting-to-Your-Child-s-Class-via-a-Parent-Code) — 코드 30일 만료·4회 사용 제한 근거.
2. [Google Classroom — Get email summaries (for guardians)](https://support.google.com/edu/classroom/answer/6388136?hl=en) — 120일 초대 유효·daily/weekly 옵션.
3. [Seesaw — Getting started with Seesaw for Families](https://help.seesaw.me/hc/en-us/articles/206514655-Getting-started-with-Seesaw-for-Families) — 자녀 태그 기반 필터링, QR/이메일/링크 3채널.
4. [Canvas LMS — How do I link a student to an observer (Pairing Code)](https://community.canvaslms.com/t5/Instructor-Guide/How-do-I-link-a-student-to-an-observer-in-a-course/ta-p/1254) — 6자리 pairing code 및 Observer RBAC.
5. [클래스팅 — 클래스 가입 방법 안내](https://support.classting.com/hc/ko/articles/900005549323) — 초대 코드 기반 학부모/학생 가입(한국 현장 레퍼런스).
6. [하이클래스 공식 사이트](https://www.hiclass.net/) — 알림장·하이콜·수신확인 UX 레퍼런스.

---

## 6. 검증 게이트 체크

- [x] 후보 ≥ 4 (6개 수록)
- [x] 각 장단점 ≥ 2
- [x] 1순위 권고 + 근거 (한국 초중 + 프라이버시 + 모바일 우선 명시)
- [x] 참조 링크 ≥ 2 (6개)
- [x] 공식 문서 WebFetch 시도 (ClassDojo 403, Google Classroom 페이지 가져옴 — URL·절차는 WebSearch snippet으로 교차검증됨)
- [x] 모바일 뷰포트 관점(390px 세로) 신규 제약 명시
- [x] 라이선스/가격 칼럼 기입
