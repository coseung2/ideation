# Phase 1 — Breakout Room 보드 탐색

> task_id: `2026-04-12-breakout-room-board`
> 작성일: 2026-04-12
> 기준 단말: 갤럭시 탭 S6 Lite (Chrome Android, Wi-Fi 50Mbps 공유, 30대 동시 접속)

## 조사 범위

"브레이크아웃룸 보드 + 템플릿 갤러리 기반 개설 + 모둠별 섹션·링크 일괄 배포" 플로우를 구현하는 기존 도구·패턴을 수집·비교. 벤치마크 대상 Padlet Breakout rooms에 더해 화상회의 계열(Zoom), 협업 화이트보드 계열(Miro / FigJam), 교실 프리젠테이션 계열(Nearpod Collaborate), 초경량 대안(Ziteboard)까지 5개 후보를 축적했다.

---

## 1. 비교표

| # | 후보 | 유형 | 라이선스 / 가격 | 템플릿 갤러리 | 모둠 배포 UX | 태블릿 친화 (갤탭 S6 Lite) | Aura-board 적합성 |
|---|---|---|---|---|---|---|---|
| 1 | **Padlet Breakout rooms** | SaaS 게시판형 보드 | 독점 / Free 3보드→Pro 월 $12 | 있음 (`padlet.com/site/templates` 프리셋 다수, 단 Breakout 전용 템플릿 결합 약함) | 섹션마다 고유 공유 링크, 뷰어 권한 분리. **교사가 매번 섹션 수동 복제** | **렉 심각 — 전체 보드 상태 후 클라 필터, WS 브로드캐스트가 모든 섹션에 전파** (roadmap §2) | 직접 벤치마크 대상. 기능성 ○, 성능 ✕ → Aura가 극복해야 할 기준선 |
| 2 | **Miro Breakout frames (BETA)** | SaaS 무한 캔버스 | 독점 / Business·Education·Enterprise 전용 ($20+/user/mo, Edu 100명 무료) | 있음 (5,000+ 커뮤니티 템플릿, Breakout Group 템플릿 내장) | 퍼실리테이터가 프레임에 참가자 자동 배정·잠금, 한 보드 내부에서 프레임별 격리 | 무거운 캔버스 엔진, 30대 저가 태블릿 동시 렌더 시 FPS·메모리 우려 | 기능 모델 참고에 유용 (프레임 = 섹션 잠금 사상) but 라이선스·성능 둘 다 부적합 |
| 3 | **FigJam sections + 템플릿** | SaaS 화이트보드 | 독점 / 무료 3파일→Professional $3/편집자, Education 무료 | 강력 (300+ 템플릿, 교육 전용 라이브러리 `figma.com/@education`) | Sections로 영역 구획만 가능, **"섹션당 링크" 개념 없음** — 보드 전체 공유만 | Figma 엔진 무거움, 태블릿 Chrome 3+ 동시에 부담 | 템플릿 갤러리 UX 벤치마크로 최적. 단 섹션 격리 기능은 없어 그대로는 부적합 |
| 4 | **Zoom Breakout Rooms** | 화상회의 내장 | 독점 / 미팅 라이선스 부가 | 없음 (회의실 = 빈 공간) | 3가지 모드: 자동 배정 / 수동 배정 / **Self-Select (학생이 직접 방 선택)** v5.3.0+ | 네이티브 앱 중심, 브라우저 보드와 결이 다름 | 배포 UX의 "Self-Select" 패턴은 Aura 모둠 입장 UX에 이식 가치 ◎ |
| 5 | **Nearpod Collaborate Board** | SaaS 수업 플랫폼 | 독점 / Silver 무료→Gold $159/yr | 레슨 템플릿 중심, Collaborate 자체는 단일 보드 | **5자리 코드**로 학생 입장, 모둠 분리는 별도 보드 복제 필요 | 제한적 상호작용만 있어 가볍고 태블릿 친화 ○ | 코드 기반 저마찰 입장(5자 코드)은 QR+토큰 링크와 병행 고려 가치 ○ |
| 6 | **Ziteboard** (레퍼런스용) | SaaS 초경량 whiteboard | 독점 / Free + Pro $8/mo | 약함 | 링크 공유만 | **저사양 기기 최적화 명시** — iPad Safari·Chrome 대응, 구형 기기까지 고려 | 성능 철학 레퍼런스. 기능(섹션·템플릿)이 부족해 후보보다 참고용 |

> 라이선스 관점: 후보 1–5 전부 **독점(closed-source SaaS)**. 오픈소스·자체 호스팅 솔루션은 이 도메인(섹션 격리 + 템플릿 갤러리 + 모둠 배포 통합)에서 발견되지 않음. → Aura-board가 **자체 구현(GPL 격리 설계)** 하는 것이 합리적이며, 위 5개는 UX·패턴 레퍼런스로만 활용.

---

## 2. 후보별 장단점 + Aura 적합 지점

### 1) Padlet Breakout rooms
- **장점**
  - 섹션 = 독립 링크 = 독립 뷰라는 **멘탈 모델이 단순**, 교사가 직관적으로 이해.
  - Wall/Timeline/Grid 레이아웃에서 그대로 붙는 확장형 설계.
- **단점**
  - 전체 보드를 클라이언트에서 받고 섹션만 필터 → 갤탭 S6 Lite에서 **초기 TTI 6–10s 체감, 렉**.
  - **템플릿 → Breakout 자동 파생이 없음.** 교사가 섹션 N개 수동 생성·복제·리네이밍.
- **Aura 적합 지점**: roadmap T0-① "섹션 격리 Breakout 뷰"가 정확히 이 UX를 **성능 개선판으로 재구현** 선언. 기능 스코프는 모범답안, 구현은 반대로(섹션별 채널 분리) 가야 함.

### 2) Miro Breakout frames (BETA)
- **장점**
  - **Frame = 잠금 가능한 격리 영역** 개념이 깔끔. 퍼실리테이터가 참가자를 프레임에 배정·락 가능.
  - 템플릿 = Breakout frames 상태까지 저장 → **"템플릿 하나로 모둠 구조까지 재현"** 이라는 Aura 목표의 완성형.
- **단점**
  - 무한 캔버스 렌더링 비용이 큼. 저가 태블릿 30대 동시 접속은 Miro 공식 권장 사양 밖.
  - Business/Education 플랜 한정 — 30명 교실 전원에 Edu 무료 배포 가능하지만 한국 학교 SSO·개인정보 이슈.
- **Aura 적합 지점**: **"템플릿 저장 시 Breakout 구조까지 직렬화"** 패턴을 `boardTemplate.sections[]` 스키마로 이식. 프레임 잠금 = rbac `viewSection` 매핑.

### 3) FigJam Sections + 템플릿 갤러리
- **장점**
  - **300+ 교육 템플릿**, 아이스브레이커·브레인스토밍·그래픽 오거나이저 카탈로그가 광범위.
  - Sections 생성 UX(⇧S 단축키, 드래그 생성)가 교사에게 빠름.
- **단점**
  - Sections는 시각적 그룹핑만, **접근 격리·고유 링크 발급 없음**. "모두가 모든 섹션을 본다".
  - Figma 엔진 자체가 태블릿 저사양에선 무거움(WebGL·Canvas 비용).
- **Aura 적합 지점**: **템플릿 갤러리 UI 패턴** (검색·카테고리·미리보기·"템플릿으로 시작")을 그대로 벤치마크. 격리 기능은 Padlet 사상으로 보완.

### 4) Zoom Breakout Rooms
- **장점**
  - **Self-Select 모드(v5.3.0+)** — 학생이 직접 방(모둠)을 고르는 UX, 교사 배정 부담 ↓.
  - 수동 배정 / 자동 배정 / 자율 선택 **3모드가 모두 한 UI에** 녹아 있어 교실 상황 대응력 ◎.
- **단점**
  - 도메인이 화상회의여서 **지속 산출물(보드) 없음** → Aura 본질과 불일치.
  - Breakout 관리용 공식 API가 부재 (Zoom 개발자 포럼 Feature Request 수년째).
- **Aura 적합 지점**: **"자율 선택 vs 교사 배정" 토글**을 Aura 모둠 링크 배포 UX에 이식. 학생이 첫 진입 시 모둠 선택 화면을 보게 할지, 링크로 고정할지 선택권.

### 5) Nearpod Collaborate Board
- **장점**
  - **5자리 코드 입장**이라는 초저마찰 UX (태블릿에서 긴 URL 입력은 에러율 높음 — 이 패턴은 귀중).
  - 교사 주도 페이스(Live) vs 학생 페이스(Student-Paced) 이원화, Aura의 "교사 통합 뷰 vs 학생 섹션 뷰" 분기와 대응.
- **단점**
  - Collaborate Board 자체는 단일 화면, 모둠 분리하려면 **레슨을 통째로 복제** → Aura가 풀려는 문제가 그대로 존재.
  - 텍스트·이미지만 지원, 풍부한 카드/iframe 없음.
- **Aura 적합 지점**: **입장 메커니즘 혼합 (QR + 5자 코드 + accessToken 링크)**. 갤탭에서 QR 못 찍는 상황 대비 백업 경로.

---

## 3. 1순위 권고 + 근거

### 권고: **Padlet Breakout rooms의 UX 모델을 유지하되, 구현·템플릿화·배포 레이어는 Miro Breakout frames 사상과 FigJam 템플릿 갤러리 UX를 합성하여 자체 구현**

**근거 (Aura-board 제약과 직결)**:

1. **성능(태블릿·GPL 격리)**: Padlet의 "전체 로드 후 클라 필터" 구조가 갤탭 S6 Lite × 30대에서 명백한 병목. roadmap T0-①이 `viewSection=sectionId`로 **서버단 프리필터 + 섹션별 WS 채널 분리**를 명시한 이유가 바로 이것. Miro의 프레임 잠금 개념(잠금된 영역 = 별도 상태 구독)을 차용하면 `sectionChannel:{boardId}:{sectionId}` 토픽만 구독해 브로드캐스트 비용을 O(N)→O(1)로 낮출 수 있다. iframe LRU 3개 정책도 섹션 단위에서 자연스럽게 유지.

2. **템플릿 갤러리**: FigJam의 교육 템플릿 UX(카테고리·미리보기·"이 템플릿으로 시작")가 교사 3분 이내 개설 목표에 직접 기여. 단 Aura는 **Breakout 전용 템플릿**(토의·브레인스토밍·정리·발표 프리셋)만 카탈로그로 노출해 의사결정 피로도 ↓. `boardTemplate.sections[]` 스키마에 모둠 섹션 구조까지 직렬화하면 Miro식 "템플릿 저장 시 Breakout 상태 보존"이 단일 INSERT로 해결됨.

3. **모둠 배포**: Zoom Self-Select의 "학생이 직접 모둠을 고른다" 토글 + Nearpod의 "5자 코드 입장"을 **accessToken 링크 + QR 일괄 발급 + 공통 랜딩 페이지**로 합성. 교실 현실(QR 스캔 실패·링크 오타)을 5자 코드로 백업. 기존 seed_43fdf181262f (행사 신청 보드) 의 accessToken·owner·collaborator 패턴과 곧장 재사용 호환.

4. **라이선스·tier**: 오픈소스 기성품이 이 조합(격리+템플릿+모둠배포)으로는 존재하지 않음 → 자체 구현이 유일 해법. Free/Pro tier 규칙(Free 1반·iframe 제한)은 `breakoutTemplate` 레벨에서 `requiresPro: boolean` 필드로 자연스럽게 승계 가능.

**정리**: 단일 솔루션 채택이 아니라 **"UX = Padlet + Zoom + Nearpod, 구현 = Miro frame 모델, 탐색 = FigJam 템플릿 갤러리"** 의 하이브리드가 1순위 권고안. Aura의 차별화 축("저가 태블릿에서 렉 없는 교실 보드")을 이 조합이 정확히 타격한다.

---

## 4. 참조 링크

1. [Padlet Help — Breakout rooms / Share links](https://padlet.help/l/en/article/9l0pv8a2si-share-links) — 섹션별 링크 발급 메커니즘 공식 문서 (WebFetch 검증 완료)
2. [Miro Help — Breakout frames (BETA)](https://help.miro.com/hc/en-us/articles/4408994822546-Breakout-frames-BETA) — 프레임 잠금·참가자 배정 사양
3. [Figma — Organize your FigJam board with sections](https://help.figma.com/hc/en-us/articles/4939765379351-Organize-your-FigJam-board-with-sections) — 섹션 생성·템플릿 연동
4. [UAB Learning — Zoom Self-Select Breakout Rooms](https://www.uab.edu/elearning/news/technology-tips-updates-news/new-zoom-feature-self-select-breakout-rooms) — v5.3.0+ Self-Select 토글
5. [Nearpod — Build a Collaborate Board](https://nearpod.zendesk.com/hc/en-us/articles/360048806572-Build-a-Collaborate-Board) — 5자 코드 입장 UX
6. [Miro Help — Education plan](https://help.miro.com/hc/en-us/articles/360017730473-Education-plan) — Edu 100명 무료, 라이선스 조건
7. 내부: `ideation/plans/tablet-performance-roadmap.md` §5 T0-① 섹션 격리 Breakout 뷰 (전제 승계)
