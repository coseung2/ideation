# Exploration — 수행평가 자동채점 파이프라인 (Aura-board → Aura 웹앱)

> task_id: `2026-04-16-performance-assessment-autograde`
> 작성: 2026-04-16 (phase1 / explorer)
> 기준 단말: **갤럭시 탭 S6 Lite** (Snapdragon 720G, 4GB RAM, Chrome Android, S-Pen)
> 선행 맥락: `plans/assignment-board-roadmap.md` (제출물 수거), `plans/canva-publisher-receiver-roadmap.md` (PAT 패턴), `plans/parent-viewer-roadmap.md` v2 (RLS 3중 격리), `plans/tablet-performance-roadmap.md` §2a (30카드 격자 게이트), `plans/seeds-index.md` Seed 11(`seed_38c34e91bf28`).
> 범위: **다축 복합 주제** — 단일 후보 비교가 아니라 5개 기능 축 각각 ≥ 3 후보.

---

## 0. 핵심 질문

> 교사가 Aura-board에서 수행평가를 배부하면, 학생이 **갤럭시 탭 S6 Lite + S-Pen**으로 객관식 마킹·서술형 필기로 응시하고, 제출 즉시 **객관식 100% 자동 + 서술형 AI 1차 채점 제안(교사 확정 전)**이 생성되어 **Aura 웹앱 성적 탭으로 실시간 송신**되도록 하려면 어떤 조합을 채택해야 하는가?

5개 결정 축:
- **A** 객관식/OMR 수집·채점 엔진
- **B** 서술형 답안 입력 + 필기 인식(OCR/Digital Ink)
- **C** 서술형 자동채점 AI/루브릭 엔진
- **D** 성적 동기화 채널 (Aura-board → Aura 웹앱)
- **E** 성적 기록부 UI 패턴 (Aura 웹앱 신규 탭)

---

## 축 A — 객관식/OMR 수집·채점 엔진

> **1줄 요약**: "종이 OMR 스캐너"는 본 주제 대상 아님 — 탭에서 터치 마킹 기반의 디지털 객관식(선다·단답·T/F). 교사 UX는 Aura-board 내부 통합, 학생 UX는 태블릿 터치 한 번에 마킹.

### A 비교표

| # | 이름 | 유형 | 라이선스·가격 | 태블릿 친화 | Aura 적합성 | 한국어 |
|---|---|---|---|---|---|---|
| A1 | **SurveyJS Form Library + Scored Quiz** | 클라이언트 JS 라이브러리 (React/Vue/Angular/Vanilla) | MIT (Form Library) | ★★★ | ★★★ | ★★★ (UI 번역 JSON) |
| A2 | H5P Quiz (Multiple Choice / Single Choice / Question Set) | LGPL 2.1 (core) + 일부 MIT. 자체 호스팅 무료, h5p.com SaaS 별도 | ★★ | ★★ | ★★ (한국어 UI 가능, 콘텐츠는 작성) |
| A3 | Moodle Quiz 모듈 | GPL-3.0, 자체 호스팅 무료 | ★ (LMS 프론트는 폼 위주, Aura 내장 어려움) | ★ | ★★★ (완전 한글화) |
| A4 | Google Forms Quiz | SaaS 무료 (구글 계정) | ★★ (iframe) | ★ | ★★★ |
| A5 | Quizizz / Wayground | SaaS Freemium (개인 $216/년) | ★★ | ★ | ★★ |
| A6 | LimeSurvey | GPL-2.0+, 자체 호스팅 | ★ (PHP 서버 별도) | ★ | ★★ |

### A 후보 상세

#### A1. SurveyJS Form Library + Scored Quiz (**1순위**)
- 장점(1): MIT 라이선스 — Aura-board Next.js 번들에 `survey-core` 직접 import 가능, **iframe 불필요** → tablet-performance §2a "iframe v1 금지" 규칙 준수.
- 장점(2): JSON 스키마 기반 (`questions[].correctAnswer`, `score`) → 교사 UI에서 폼 작성 후 그대로 DB 저장, 학생 응시 화면은 같은 JSON을 다른 컴포넌트(Survey 렌더러)로 렌더. `onCompleting` 이벤트로 클라이언트 즉시 채점 + 서버 재검증.
- 장점(3): 한국어 UI 로케일 내장, 폼 빌더(`survey-creator` — 상용 라이선스 주의)는 교사측 선택 옵션.
- 단점(1): `survey-creator`(드래그 폼 빌더)는 **상용 라이선스**. 교사가 드래그로 출제하려면 유료 혹은 자체 폼 빌더 UI 구현 필요.
- 단점(2): 정답 검증을 클라이언트에서 하면 학생이 개발자 도구로 점수 조작 가능 → 반드시 서버 재채점 레이어 필요(정답은 서버만 보유 + 응시 중엔 암호화 토큰화).
- Aura 적합 지점: `Board.layout="assessment"` 확장 + `Question` JSON을 서버 원본, 학생 탭엔 정답 필드 제거한 sanitized JSON 전송. 30 객관식 × 30명 기준 DOM 노드 ≤ 300, Snapdragon 720G TTI < 2s 확보.

#### A2. H5P Quiz (Question Set / Multiple Choice)
- 장점(1): 수백 개 학교·EduTech 스택에서 검증된 퀴즈 UX. xAPI 통계 호환.
- 장점(2): 질문 유형 풍부(드래그매치·빈칸 등) — 향후 서술형 경량 유형까지 확장 용이.
- 단점(1): **iframe 임베드 중심** 아키텍처. tablet-performance `iframe 동시 마운트 ≤ 3` 예산을 학생당 1개 문제집만 띄워도 빠듯. 보드 내 여러 퀴즈 동시 배치 불가.
- 단점(2): LGPL core — Aura-board에 정적 링크 시 라이선스 격리 부담. iframe 격리로 해결 가능하나 그 자체가 위 제약에 걸림.
- Aura 적합 지점: 낮음. 교사가 H5P로 별도 제작 후 링크 카드로만 연결하는 보조 옵션.

#### A3. Moodle Quiz 모듈
- 장점(1): GPL-3.0 + 한국 교육청 상당수 도입한 LMS — 신뢰성·한글화 최고 수준.
- 장점(2): 문항은행·랜덤화·부분점수 등 기능 성숙.
- 단점(1): Moodle은 **전체 LMS** — Aura 내부 통합 현실적 불가, 외부 SSO 링크로만. 학생 컨텍스트 이동 발생.
- 단점(2): Aura-board의 카드·슬롯 모델과 UX 단절 (학생이 Moodle 사이트로 점프).
- Aura 적합 지점: 낮음. 참조용·기능 비교 벤치마크로만 활용.

#### A4. Google Forms Quiz
- 장점(1): 교사 진입 장벽 0, 한국 교사 보편 도구.
- 장점(2): 자동 채점·CSV 내보내기 무료.
- 단점(1): iframe 임베드만 가능, 커스터마이징 0. 학생 학번·자동 매칭 불가(본인 인증 구글 계정 필요).
- 단점(2): 성적의 Aura 웹앱 송신은 **Google Sheets API 폴링** 방식 외엔 없음 — 실시간 아님, Google 정책 변경 리스크.
- Aura 적합 지점: 중간. MVP 단계에서 "Google Forms 링크 붙이기" 카드로 임시 대응 가능하나 자동채점·성적 송신 루프 미완.

#### A5. Quizizz / Wayground
- 장점(1): 게임성 — 초등 저학년 몰입 강함.
- 장점(2): LTI 1.3 Canvas·Schoology·Classroom 연동.
- 단점(1): SaaS 유료($216/년 개인, 학교 단위 별도). Aura 수익 모델과 중복 과금.
- 단점(2): 게임 진행이 교사 주도·시간 제한형 — "수행평가(평가 기준 반영, 개별 속도)"와 어긋남.
- Aura 적합 지점: 낮음. 플레이형 퀴즈는 Aura-board에 도입할 가치 적음 (평가보다 형성평가용).

#### A6. LimeSurvey
- 장점(1): GPL-2.0+ 완전 오픈소스 + 한국어 지원.
- 장점(2): 복잡한 분기·랜덤화 지원.
- 단점(1): PHP 스택 별도 배포 필요 — Aura의 Vercel/Next.js 원자 배포 모델과 불일치.
- 단점(2): 교사 UI가 IT 친화적·학교 현장 수용성 낮음.
- Aura 적합 지점: 낮음.

### A 축 권고

**A1 SurveyJS Form Library (MIT)** — JSON 기반 단일 스키마를 Aura DB에 직접 저장, React 네이티브 번들, 태블릿 친화, 한국어 OK, iframe 불필요. 교사 출제 UI는 초기에 **심플 폼**(문항 타입 드롭다운 + 보기 4개 + 정답 선택)만 자체 구현하고, v1.5에서 SurveyJS Form Library 렌더러를 확대한다. `survey-creator` 상용 유료 구간은 회피.

---

## 축 B — 서술형 답안 입력 + 필기/OCR

> **1줄 요약**: 갤럭시 탭 S6 Lite + S-Pen에서 학생이 답을 **손으로 쓰고**, 서버엔 텍스트로 저장돼야 AI 채점이 가능하다. 선택지: (a) 온디바이스 Digital Ink, (b) 클라우드 OCR, (c) 키보드 입력만 허용(회피).

### B 비교표

| # | 이름 | 유형 | 라이선스·가격 | 태블릿 친화 | Aura 적합성 | 한글 손글씨 |
|---|---|---|---|---|---|---|
| B1 | **Google ML Kit Digital Ink (한국어 `ko`)** | 온디바이스 SDK (Android/iOS Native) | 무료 (Google Play Services) | ★★★ | ★★ (Chrome Web 미지원 → Capacitor/WebView 브릿지 필요) | ★★ |
| B2 | **Naver CLOVA OCR (General + Document)** | SaaS REST API | 유료 (종량제, 한글 손글씨 지원) | ★★ | ★★★ (이미지 업로드) | ★★★ |
| B3 | **Upstage Document OCR** | SaaS REST API | 유료 (per-page) | ★★ | ★★★ | ★★ (인쇄체 강점, 손글씨 미지원 공식 문구) |
| B4 | MyScript iink SDK 4.3 | 상용 SDK + SaaS | 유료 (학교 단위 라이선스) | ★★★ | ★ (CJK 공식 로드맵 未성숙, 한글 미출시) | ✗ |
| B5 | Tesseract.js | WASM 오픈소스 | Apache-2.0, 무료 | ★ (대용량 WASM) | ★ | ★ (인쇄체 ○, 손글씨 ✗) |
| B6 | GPT-4o Vision / Gemini 2.5 Flash image input | LLM 비전 | 유료 API (축 C와 공유) | ★★★ | ★★★ | ★★★ (LLM 한글 손글씨 인식 2025~ 급상승) |
| B7 | 키보드 입력 강제 (fallback) | 브라우저 기본 | 무료 | ★★ (S-Pen 무용화) | ★★ | N/A |

### B 후보 상세

#### B1. Google ML Kit Digital Ink Recognition (ko)
- 장점(1): **완전 온디바이스** — 네트워크 단절 대응(known_constraint). 갤럭시 기기에서 Samsung Handwriting과 동일 계열 기술, S-Pen 최적화.
- 장점(2): 300+ 언어 — 한국어(`ko`, 제스처 변형 `ko-x-gesture`) 공식 지원. 모델 다운로드 후 추가 요금 없음.
- 단점(1): **Android Native SDK** — Aura-board가 Chrome PWA라면 Capacitor 래핑 혹은 "Aura Board 태블릿 앱" 별도 셸 필요. 순수 웹만으로 사용 불가.
- 단점(2): 모델별 정확도 공개치 없음, 한글 초등 저학년 불완전 자소 인식은 베타 수준일 가능성. 실측 필요.
- Aura 적합 지점: **학교 배포 시 Aura Board 학생 앱(Capacitor)** 버전을 계획한다면 최강 옵션. 순수 웹 유지면 포기.

#### B2. Naver CLOVA OCR (**클라우드 2순위**)
- 장점(1): 한국어 손글씨 **공식 지원**. 국내 문서에 특화된 엔진.
- 장점(2): REST API 간단 — 학생이 S-Pen으로 캔버스에 쓴 결과를 PNG로 서버 전송 → CLOVA 호출 → 텍스트 결과.
- 단점(1): **종량 과금**. 2021년 참조 기준 $6,000/100K 요청(현재 가격 재확인 필요). 학급당 월 수천 요청 발생 시 비용 부담.
- 단점(2): 클라우드 — 네트워크 단절 시 제출 큐에 쌓아야 함. PII 유출 리스크(학생 필체).
- Aura 적합 지점: 서버측 프록시(`/api/external/ocr` 유사)로 일괄 처리, Supabase Storage 업로드 후 비동기 OCR job queue.

#### B3. Upstage Document OCR
- 장점(1): 한국어 레이아웃·표 인식 탁월.
- 장점(2): Solar 계열 자체 LLM 연계로 OCR→grading 단일 벤더 가능.
- 단점(1): Document Parse는 **손글씨 미지원**(공식). 수행평가 서술형이 손글씨면 부적합, 키보드·태블릿 입력 텍스트면 애초에 OCR 불필요.
- 단점(2): 가격 불투명(요청 견적 필요).
- Aura 적합 지점: 종이 답안지 스캔 경로가 포함될 경우만. 태블릿 중심이면 B2 우위.

#### B4. MyScript iink SDK 4.3
- 장점(1): iinkJS 순수 웹 라이브러리 — Aura-board에 import 가능.
- 장점(2): 2026-01 iink SDK 4.3 출시 — 18MB 단일 멀티 언어 엔진.
- 단점(1): **한국어(CJK)는 공식 로드맵상 개발 중** — 현재 Latin·기호·수식만. 2026-04 기준 한글 미지원.
- 단점(2): 상용 라이선스 — 학교 단위 계약 필요.
- Aura 적합 지점: 한국어 출시 전까지 불가. 수식·도형 입력 필요 시 재검토.

#### B5. Tesseract.js
- 장점(1): 완전 오픈소스(Apache-2.0), 클라이언트 WASM.
- 장점(2): 데이터 주권 100% 유지.
- 단점(1): **손글씨 인식 매우 약함**, 인쇄체 전용. 수행평가엔 부적합.
- 단점(2): WASM 30MB+ — Snapdragon 720G 초기 로딩·메모리 부담.
- Aura 적합 지점: 매우 낮음.

#### B6. LLM Vision(GPT-4o/Gemini Flash) (**현실적 1순위**)
- 장점(1): 2025년 이후 한글 손글씨 인식 품질이 OCR 전용 서비스를 추월한 보고 다수. 축 C 채점 LLM과 **동일 API 콜로 OCR+채점 합침** → 1 round-trip, 비용 절약.
- 장점(2): 루브릭·맥락까지 prompt에 함께 주입 가능 → "답안이 읽히지 않으면 '판독 불가' 표기 후 교사 재확인" 같은 정책 자연스러움.
- 단점(1): 판독 실패 케이스 디버그 어려움 (OCR 분리 시 오류 출처 분리 가능 vs 통합 시 블랙박스).
- 단점(2): 클라우드 의존 — 단절 시 큐잉 필수.
- Aura 적합 지점: **v1 권고 경로**. 학생은 tldraw/perfect-freehand 기반 캔버스에 S-Pen 필기 → 제출 시 PNG(또는 SVG→PNG 서버 렌더) → Gemini 2.5 Flash로 OCR+채점 한 번에.

#### B7. 키보드 입력 강제 (fallback)
- 장점(1): OCR 리스크·비용 0.
- 장점(2): 데이터 즉시 텍스트 → 채점 파이프라인 단순.
- 단점(1): S-Pen 태블릿 활용도 0 — 갤탭 도입 목적 훼손.
- 단점(2): 초등 저학년 타이핑 느림.
- Aura 적합 지점: 중고등 대상 서술형 시험에는 기본 옵션으로 제공. 초등은 필기 병행.

### B 축 권고

**v1: B6 LLM Vision (Gemini 2.5 Flash) + B7 키보드 토글 제공**
- 학생은 캔버스 필기(S-Pen) 또는 키보드 입력 중 선택. 제출 시 필기는 PNG→LLM(Vision+채점 통합), 키보드는 텍스트→LLM(채점만).
- **v1.5 fallback 대안**: CLOVA OCR을 별도 프리-프로세스 단계로 두어 LLM 비용·오인식 문제 완화 옵션.
- **v2 고려**: Capacitor "Aura Board Tablet" 앱 배포 시 ML Kit Digital Ink를 전경 스테이지로, 네트워크 단절 대응 강화.

---

## 축 C — 서술형 자동채점 AI / 루브릭 엔진

> **1줄 요약**: 교사가 정한 루브릭(채점 기준표)을 프롬프트에 주입하고, 학생 답안을 LLM에 보내서 **점수 + 피드백 제안**을 받는다. 교사 확정 전 "제안" 단계 필수(known_constraint).

### C 비교표

| # | 이름 | 유형 | 라이선스·가격 (2026-04 기준) | 태블릿 영향 | Aura 적합성 | 한국어 |
|---|---|---|---|---|---|---|
| C1 | **Gemini 2.5 Flash** | SaaS LLM | $0.15/M in, $0.60/M out, 1M ctx | ★★★ (서버 측) | ★★★ | ★★★ |
| C2 | **GPT-4o-mini / 후속 GPT-5 mini** | SaaS LLM | $0.15/M in, $0.60/M out (4o-mini 기준, 후속 모델 교체 예정) | ★★★ | ★★★ | ★★★ |
| C3 | **Claude Haiku 4.5** | SaaS LLM | $0.25/M in, $1.25/M out (Haiku 4.5 추정, 3-Haiku는 2026-04 퇴역) | ★★★ | ★★★ | ★★ |
| C4 | **EXAONE 3.5 (LG AI Research) / EXAONE 4.0** | 오픈 웨이트 한국어 LLM | 비상업 라이선스 주의 필요, 자체 호스팅 | ★★★ | ★★ | ★★★ (한국어 특화 +4~6pp) |
| C5 | Qwen2.5-32B-Instruct | Apache-2.0 오픈 웨이트 | 자체 호스팅 (A100/A6000) | ★★★ | ★★ | ★★ |
| C6 | Sentence-Transformers (KR-SBERT, KLUE-RoBERTa) + 코사인 유사도 | Apache-2.0 | 자체 호스팅 CPU 가능 | ★★★ | ★★ | ★★★ |
| C7 | Clipo (국내 SaaS) | 상용 SaaS (수행평가 특화) | 학교 단위 라이선스 | — (외부 이동) | ★ | ★★★ |
| C8 | Gradescope AI-assisted grading | 상용 SaaS (대학 중심) | 기관 라이선스 | — | ★ | ★ (영어·수학 강, 한국어 미검증) |
| C9 | Promptfoo `llm-rubric` / AutoRubric | 오픈소스 평가 프레임워크 | MIT | N/A (자체적으로 LLM은 위 중 하나) | — (래퍼) | — |

### C 후보 상세

#### C1. Gemini 2.5 Flash (**1순위**)
- 장점(1): 가격·성능 균형 최고 — $0.15/$0.60 per M tokens, **1M 컨텍스트**로 루브릭+학급 전체 답안 일괄 투입 가능 → 답안 간 상대 일관성 확보.
- 장점(2): Vision 모달리티 무료 포함 — 축 B의 필기 이미지 OCR을 동일 콜에 통합.
- 단점(1): Google 의존 — 학교 데이터 외부 전송 동의 필요(개인정보보호법·학부모 고지).
- 단점(2): 지연(p95) 2~5초 — 제출 직후 "채점 중" 상태 UX 필요.
- Aura 적합 지점: **v1 default**. `/api/grading/suggest` 서버 액션 → Gemini 2.5 Flash 호출 → `GradeSuggestion` row 생성.

#### C2. GPT-4o-mini / GPT-5 mini
- 장점(1): Gemini와 동일 가격대($0.15/$0.60), 한국어 품질 상위권.
- 장점(2): 함수 호출·structured output 성숙.
- 단점(1): 컨텍스트 128K — Gemini 1M 대비 학급 일괄 투입 제한.
- 단점(2): OpenAI 정책 변화(교육·미성년 데이터) 추적 필요.
- Aura 적합 지점: v1 대체 백엔드(이중 벤더). 벤더 락인 방지용 **프로바이더 추상화 레이어** 설계 권장.

#### C3. Claude Haiku 4.5
- 장점(1): 루브릭 추종력 우수 (rubric-conditioned grading 논문 다수 Claude 기반).
- 장점(2): 안전·편향 제어 강함 — 학생 답안 민감 표현 처리 우위.
- 단점(1): 가격 2배(Gemini Flash 대비) — 월 수만 건 채점 시 비용 차이 누적.
- 단점(2): Claude 3 Haiku는 2026-04 퇴역 — Haiku 4.5로 이관 필수.
- Aura 적합 지점: "교사 확정 전 제안" 단계의 민감 답변(사회·도덕·문학) 전용 라우팅 옵션.

#### C4. EXAONE 3.5 / 4.0 (LG)
- 장점(1): **한국어 CSAT 도메인 적응 pretraining +4~6pp** (2026 KoCSAT 리더보드).
- 장점(2): 2.4B·7.8B 모델은 단일 GPU 자체 호스팅 가능 → 학교 데이터 국외 유출 0.
- 단점(1): EXAONE 비상업 라이선스 — Aura가 유료 Pro tier로 수익화하면 적용 여부 재확인 필요 (연구·교육 목적 조항 해석).
- 단점(2): 자체 호스팅 운영비(GPU 1대 월 $300+).
- Aura 적합 지점: 개인정보 민감 학교(사립·특수목적) 대상 **Enterprise tier** 옵션.

#### C5. Qwen2.5-32B / Qwen3
- 장점(1): Apache-2.0 — 상업 이용 자유.
- 장점(2): vLLM 최적화 성숙, A6000 2장 텐서병렬 가능.
- 단점(1): 한국어는 EXAONE 대비 약간 열위 (KLUE 벤치상 -2~3pp).
- 단점(2): 32B는 GPU 메모리 64GB 이상 요구 — 학교 단위 비현실적, 클라우드 GPU 렌트 필요.
- Aura 적합 지점: Aura 자체 채점 서버 구축 시 후보. v1엔 과도.

#### C6. Sentence-Transformers + 코사인 유사도
- 장점(1): CPU 추론 가능 — 비용 거의 0, 실시간 지연 <100ms.
- 장점(2): "모범답안 N개 vs 학생 답안" 의미 유사도는 단답형/키워드형 수행평가에 적합.
- 단점(1): 루브릭·부분점수·근거 피드백 생성 불가 — LLM 필수 병행.
- 단점(2): 3-way/5-way partial credit은 LLM이 훨씬 우수 (연구 일치).
- Aura 적합 지점: **1차 필터**(공백·무관 답안 자동 0점) + **LLM 비용 절감** 하이브리드. 단독 채점은 불가.

#### C7. Clipo (국내 SaaS)
- 장점(1): 한국 교육 현장 특화 — 서·논술 전과목 루브릭 내장, 교육청 도입 실적(스쿨앳 등재).
- 장점(2): PDF/JPG 업로드 일괄 채점 — 종이 답안 업무 흐름 기존 교사에게 익숙.
- 단점(1): **외부 SaaS로 학생 답안 이동** — Aura 수행평가 파이프라인과 분리, 성적 Aura 웹앱 자동 송신 경로 없음.
- 단점(2): 교사가 Aura·Clipo 둘 다 학습해야 함.
- Aura 적합 지점: 낮음. 벤치마크·기능 참고 대상.

#### C8. Gradescope AI-assisted grading
- 장점(1): 대학·고교 수학·과학에서 검증, rubric 변경 시 소급 적용 UX 우수.
- 장점(2): 영어 손글씨 읽기(한 줄 제약).
- 단점(1): **한국어 지원 미검증** + Turnitin 그룹 소유 → 국내 개인정보 이전 심사 부담.
- 단점(2): 초등·국어 서술형 UX와 거리 멈.
- Aura 적합 지점: 낮음.

#### C9. Promptfoo `llm-rubric` / AutoRubric 프레임워크
- 장점(1): MIT 오픈소스 — 루브릭 DSL + LLM judge 파이프라인 재사용.
- 장점(2): 평가 메트릭·consistency 체크 빌트인.
- 단점(1): 평가용 도구 → 프로덕션 채점 서비스가 아님, Aura가 래퍼를 직접 구현해야 함.
- 단점(2): LLM은 C1~C5 중 골라야 함.
- Aura 적합 지점: 내부 채점 품질 평가·회귀 테스트에 유용. 프로덕션 코어 아님.

### C 축 권고

**v1: C1 Gemini 2.5 Flash** (OCR+채점 통합) + **C6 Sentence-Transformers 전처리**(공백/무관 답안 자동 0점 필터) + **C9 Promptfoo**(내부 품질 회귀)
- 프로바이더 추상화 레이어(`/src/lib/grading/providers/{gemini,openai,claude}.ts`)로 C1·C2·C3 교체 가능하게 설계.
- **Enterprise tier v2**: C4 EXAONE 자체 호스팅으로 데이터 주권 옵션 제공.
- 모든 LLM 채점은 **"제안" 상태로만 생성**되고 교사 "확정" 버튼 이후에 `Grade.confirmedAt` 세팅 → Aura 웹앱 송신 트리거.

---

## 축 D — 성적 동기화 채널 (Aura-board → Aura 웹앱)

> **1줄 요약**: Aura-board에서 교사가 확정한 점수를 **Aura 웹앱 성적 탭**으로 실시간 전송해야 한다. `canva-publisher-receiver` PAT 패턴이 유사 선례.

### D 비교표

| # | 이름 | 유형 | 복잡도 | 실시간성 | 보안 | Aura 적합성 |
|---|---|---|---|---|---|---|
| D1 | **공통 Supabase DB + RLS 분리 + Realtime Postgres Changes** | 단일 DB 공유 | 낮음 | 실시간 (ms) | RLS로 boundary | ★★★ |
| D2 | **REST/Webhook push (PAT 패턴 재사용)** | canva-publisher-receiver 확장 | 중간 | 초 단위 | PAT + HMAC | ★★★ |
| D3 | Server-Sent Events (SSE) | aura-board → aura 웹앱 push | 중간 | 실시간 | 별도 인증 필요 | ★★ |
| D4 | PGMQ (Postgres Message Queue) | 동일 DB 내부 큐 | 중간 | 초 단위 | DB 권한 | ★★ |
| D5 | Redis Stream / Upstash | 외부 큐 | 중간 | 실시간 | 토큰 | ★★ |
| D6 | LTI 1.3 Assignment & Grade Services (AGS) | 표준 LMS 통합 | 높음 | 수초~20초 | OAuth2 Bearer | ★ |
| D7 | OneRoster 1.2 Gradebook Service (`textScore` 포함) | REST/JSON 표준 | 높음 | 배치 | OAuth2 Bearer | ★★ (공공/학교 통합 시) |

### D 후보 상세

#### D1. 공통 Supabase + RLS + Realtime Postgres Changes (**1순위**)
- 장점(1): **서비스 간 HTTP 계약 없음** — Aura-board가 `grades` 테이블에 INSERT하면 Aura 웹앱은 RLS 기반 Realtime 구독으로 즉시 수신. 채점 확정 → 성적 탭 반영 레이턴시 <500ms.
- 장점(2): Supabase RLS로 `teacherId`·`studentId`·`parentId` 각 경로 자동 필터 — parent-viewer v2와 같은 정책 엔진 재사용.
- 단점(1): 두 앱이 **같은 DB 스키마·같은 인증 풀** 공유 필수 — Aura-board와 Aura 웹앱이 **같은 Supabase 프로젝트**에 있어야 현실적.
- 단점(2): RLS 부하가 대량 ChangeFeed에서 성능 저하 → Supabase 공식 가이드: "스케일 시 별도 public 테이블+필터 또는 서버 리스트림" 권장.
- Aura 적합 지점: Aura-board와 Aura 웹앱이 **같은 Supabase 인스턴스** 전제라면 **압도적 1순위**. phase2 sketch에서 DB 공유 가정 확인 필요.

#### D2. REST/Webhook Push (PAT 패턴 재사용)
- 장점(1): **선례 존재** — `canva-publisher-receiver-roadmap.md`의 `ExternalAccessToken` + `/api/external/cards` 구조를 `/api/external/grades` 엔드포인트로 포크.
- 장점(2): 두 앱이 DB 공유 안 해도 됨 — Aura 웹앱이 독립 배포·독립 DB여도 동작.
- 단점(1): 중복 채널 유지 비용 — rate limit·PAT·3-stage migration을 또 한 번.
- 단점(2): 네트워크 단절·재시도 큐 자체 구현 필요.
- Aura 적합 지점: Aura 웹앱이 **별도 Supabase 프로젝트·별도 배포**인 경우 최적. seeds-index Seed 8 패턴 승계로 개발 공수 30~40% 절감.

#### D3. Server-Sent Events
- 장점(1): HTTP 단방향·브라우저 기본 지원.
- 장점(2): 구현 가벼움.
- 단점(1): Vercel Serverless의 SSE는 15분 세션 한도(Fluid Compute 예외) — 장시간 연결 유지 취약.
- 단점(2): 중간 HTTP 프록시·모바일 네트워크에서 끊김 이슈.
- Aura 적합 지점: D1이 안 될 때의 백업.

#### D4. PGMQ
- 장점(1): DB 내부 큐 — 외부 의존성 없음, 트랜잭션 보장.
- 장점(2): SKIP LOCKED 기반 동시성 안전.
- 단점(1): Aura 웹앱이 **같은 Postgres**에 접근해야 — D1과 전제 같음.
- 단점(2): Realtime 구독은 별도 레이어(pg_notify) 필요.
- Aura 적합 지점: D1 채택 시 백오피스 job(야간 집계·이메일 발송)에 병용. 성적 push 채널로 단독 사용은 과함.

#### D5. Redis Stream / Upstash
- 장점(1): 고성능·저비용.
- 장점(2): 다중 컨슈머 팬아웃.
- 단점(1): 외부 의존성 1개 추가.
- 단점(2): 학교 단위 메시지 볼륨 대비 오버엔지니어링.
- Aura 적합 지점: 낮음.

#### D6. LTI 1.3 AGS
- 장점(1): **글로벌 LMS 표준** — 향후 Moodle/Canvas/Blackboard 연동 시 재사용.
- 장점(2): OAuth2 Bearer · LineItem / Result 개념 성숙.
- 단점(1): Aura 웹앱은 **LMS가 아님** — Aura-board → Aura 웹앱은 양쪽 다 자사 제품. LTI 오버헤드(tool registration, platform config) 부담.
- 단점(2): 공식 Grade sync 주기 "~1시간에 1회 자동" — 수행평가 실시간 반영 요구에 미달.
- Aura 적합 지점: Aura-board가 **외부 LMS**(Moodle, Canvas)에 성적 보고할 때만 의미. 내부 Aura 웹앱 연결엔 부적합.

#### D7. OneRoster 1.2 Gradebook Service
- 장점(1): 1EdTech 공식 표준 — `textScore`로 루브릭 값 전송 가능(본 주제 핵심 요구).
- 장점(2): 공공학교·SIS 통합 시 필수 언어.
- 단점(1): Aura 내부 통신엔 과중한 표준.
- 단점(2): Category·LineItem·AssessmentResult 4-엔티티 체인 구현.
- Aura 적합 지점: **v2+ Enterprise** — 교육청·공립학교 통합 플랫폼 모드에서 재등장.

### D 축 권고

**D1 공통 Supabase + RLS + Realtime Postgres Changes** (Aura-board와 Aura 웹앱이 같은 Supabase 인스턴스라는 가정 하에)
- phase2 sketch에서 **DB 공유 여부 확인 질문 1개** 필수.
- 만약 두 앱이 분리된 Supabase라면 **D2 PAT 재사용**으로 폴백 (canva-publisher-receiver 로드맵 엔진 포크).
- **v2+**: D7 OneRoster Gradebook Service를 Enterprise tier 외부 SIS 연동 레이어로 별도 추가.

---

## 축 E — 성적 기록부 UI 패턴 (Aura 웹앱 신규 탭)

> **1줄 요약**: 교사·학생·학부모가 성적을 조회하는 UI. 표(row=학생, column=평가) 구조가 글로벌 표준. **matrix/grid 뷰는 owner + 데스크톱 전용** (메모리 방침).

### E 비교표

| # | 벤치마크 | 구조 | 강점 | 약점 | Aura 적용성 |
|---|---|---|---|---|---|
| E1 | **Google Classroom Gradebook** | 학생 행 × 과제 열, 간단 셀 편집 | 단순·초보 교사 친화 | 루브릭·부분점수 제한 | ★★★ (국내 교사 익숙) |
| E2 | **Canvas Gradebook (Instructure)** | 학생 행 × 과제 열 + rubric popover | 강력한 필터·가중치·드릴다운 | 가로 스크롤 UX 비판 다수 | ★★ (참조 강) |
| E3 | Moodle Gradebook | 학생 행 × 활동 열 + Category | 완전 커스텀 가중치 | UI 복잡·교사 학습곡선 | ★★ |
| E4 | Padlet Gradebook (Assignment Board 연계) | Padlet 자체엔 성적부 없음 (벤치마크 無) | — | — | — |
| E5 | PowerSchool SIS | SIS + Gradebook 통합 | 학부모 포털 성숙 | 미국 중심, 국내 없음 | ★ (포털 UX 참조) |
| E6 | **하이클래스 (i-Scream)** | 학생 리포트·누가기록·포인트 (성적부 아닌 생활기록) | 국내 초등 표준·학부모 앱 보급 | 공식 '성적부' 개념 약함 (법적 성적은 나이스) | ★★★ (국내 UX 기대치) |
| E7 | 클래스팅 Class123 | 학부모 앱·커뮤니케이션 중심 | 국내 점유율·학부모 친화 | 성적 기록 기능 없음 | ★★ (학부모 UX 참조) |
| E8 | 쌤기부 (teacher-logbook.kr) | AI 생기부 자동 생성 + 수행평가 채점 | 국내 교사 AI 툴 트렌드 | 기록 서사 중심, 표 UI 미약 | ★★ (서사 생성 참조) |

### E 후보 상세

#### E1. Google Classroom Gradebook
- 장점(1): 국내 교사 태반이 이미 사용 — 학습 비용 0.
- 장점(2): "학생 × 과제" 최소 단순 모델.
- 단점(1): 루브릭 셀 표현 빈약 — 수행평가 "기준1/기준2/기준3" 다차원 표현 불편.
- 단점(2): 학부모 뷰 없음 (학생 본인+교사만).
- Aura 적합 지점: owner 뷰(교사 + 데스크톱)의 **베이스 패턴**. 학생 행·과제 열·확정/미확정 상태 배지.

#### E2. Canvas Gradebook
- 장점(1): 업계 레퍼런스 — Total 열·Late/Missing 배지·루브릭 popover 성숙.
- 장점(2): 가중치·카테고리·숨김 열 등 풍부.
- 단점(1): **가로 스크롤 문제**(Instructure Community 다수 리포트) — 과제 수 많아지면 태블릿 UX 급락. `plans/tablet-performance`의 데스크톱 전용 원칙과 맞음.
- 단점(2): 교사 학습곡선 급.
- Aura 적합 지점: 참조 중 디테일 풍부. Canvas의 "열 고정·가로 스크롤" 문제는 **Aura matrix 뷰를 owner+데스크톱 전용으로 잠그는 근거**로 사용 가능.

#### E3. Moodle Gradebook
- 장점(1): Category·Weighted Mean 완전 지원.
- 장점(2): GPL — 로직 레퍼런스 합법 차용.
- 단점(1): UI 복잡도 매우 높음.
- 단점(2): 국내 교사 수용성 낮음.
- Aura 적합 지점: 로직 설계 참조. UX는 Classroom 스타일.

#### E5. PowerSchool SIS
- 장점(1): 학부모 포털의 "내 자녀 성적 타임라인" UX 완성도.
- 장점(2): 평가 항목별 드릴다운 상세 페이지.
- 단점(1): 국내 미진출·유료 엔터프라이즈.
- 단점(2): 참조만 가능.
- Aura 적합 지점: parent-viewer v2의 "자녀 단일 뷰" 확장으로 성적 타임라인 차용.

#### E6. 하이클래스
- 장점(1): **국내 초등 표준** — 학부모가 이미 앱 설치·사용. UX 기대치 기준.
- 장점(2): 누가기록·포인트 시스템 → 생활기록 병행 모델 제시.
- 단점(1): 공식 '성적'은 나이스(NEIS, 공공) 연계, 하이클래스는 비공식 요약.
- 단점(2): 채점·루브릭 UI 미약.
- Aura 적합 지점: **학부모 뷰 UX 기대치** 정렬 — 자녀 단일 프로필 + 최근 수행평가 카드 리스트.

#### E7. 클래스팅
- 장점(1): 학부모 커뮤니케이션·알림 국내 1위.
- 장점(2): 피드·학급앨범 UX.
- 단점(1): 성적 기록 기능 없음.
- Aura 적합 지점: 학부모 알림 톤·빈도만 참조.

#### E8. 쌤기부
- 장점(1): 국내 교사 AI 툴 트렌드 반영 — 수행평가 채점+생기부 자동생성 연계.
- 장점(2): 교사 업무 흐름(수행평가 → 생기부 문장) 정합.
- 단점(1): 성적부 **표** UI는 보조.
- Aura 적합 지점: "채점 확정 → 생기부 문장 초안 AI 생성" 기능을 v1.5 파킹 대상으로 포착.

### E 축 권고

**하이브리드: E1 (Google Classroom) 표 구조 + E2 (Canvas) 루브릭 popover + E6 (하이클래스) 학부모 단일 자녀 뷰**
- **owner(교사) + 데스크톱**: 학생 행 × 평가 열 표(Classroom 베이스), 각 셀 클릭 → Canvas 스타일 루브릭 popover(기준별 점수·AI 제안·피드백).
- **editor(학생) + 태블릿/PC**: 본인 행 단일 카드 뷰 (자기 점수만).
- **viewer(학부모) + 모바일 PWA**: 하이클래스 스타일 "자녀 프로필 + 최근 N개 평가 카드" 타임라인. parent-viewer v2 matrix와 합류.
- **matrix/grid 뷰**: **owner + 데스크톱 전용** (tablet-performance 방침 승계). 교사 태블릿·학부모 앱에는 미노출.

---

## 축 F — 부정행위 방지 (Test proctoring / Anti-cheating)

> **1줄 요약**: 기준 환경은 **갤럭시 탭 S6 Lite + Chrome Android + BYOD(학교 MDM 없음) + 교사 현장 감독 가능 + 초·중 학생**. 이 조합에선 OS-level 강제 락(Knox Kiosk·SEB·Locked Mode)은 **현실적 배포 불가**. 웹 표준 이벤트 감지 + 교사 대시보드 + 문항 설계의 **다층 방어**(L1~L4)가 해법. 원격 카메라 감독은 프라이버시·학부모 민원 리스크 과대로 파킹.

### F 비교표

| # | 이름 | 유형 | 라이선스·비용 | 갤탭 S6 Lite 지원 | BYOD 가능 | 초중등 적합성 | 프라이버시 리스크 |
|---|---|---|---|---|---|---|---|
| F1 | **Fullscreen API + Page Visibility API + focus/blur 이벤트** | 웹 표준 (브라우저 내장) | 무료 (MDN, W3C) | ★★★ (Chrome Android 지원) | ★★★ | ★★★ | ★ (이벤트 로그만, 영상 없음) |
| F2 | Samsung Knox Configure / Knox Manage Kiosk (ProKiosk / Single App) | Samsung MDM + OS 레벨 락 | Knox Manage 유료 (디바이스당 월 $1~3), Samsung 전용 | ★★★ (Samsung 단말 네이티브) | ✗ (**학교 소유 단말만** — BYOD 개인 계정 진입 후 MDM enroll 절차 무거움) | ★★★ (단말 배포 시) | ★ |
| F3 | Safe Exam Browser (SEB) | 오픈소스 데스크톱·iOS | 무료 (대학 컨소시엄) | ✗ (**Android 미지원**, Android는 루팅 위험 높다는 이유로 공식 보류) | N/A | N/A (Android 배포 불가) | ★ |
| F4 | Google Forms Locked Mode | SaaS (Chromebook 전용 락) | 무료 (Workspace for Education) | ✗ (**Chromebook 전용**, Android 미지원) | ✗ (학교 관리 Chromebook 필수) | ★★★ (Chromebook 보급 학교) | ★ |
| F5 | Respondus LockDown Browser + Monitor | 상용 데스크톱 SDK + 카메라 감독 | 기관 라이선스 | ✗ (Windows·Mac·ChromeOS만, **Android 태블릿 미지원 명시**) | N/A | ★ (대학·자격시험 중심) | ★★★ (카메라 녹화) |
| F6 | **커스텀 in-app 방어 조합** (copy/paste 차단, 컨텍스트메뉴 무효화, DevTools 감지, 문항풀 랜덤화, 셔플링, 시간 제한, Service Worker 네트워크 화이트리스트) | 자체 구현 (클라이언트 JS + 서버 검증) | 무료 (개발 공수) | ★★★ | ★★★ | ★★★ | ★ |
| F7 | 카메라 기반 AI 원격 감독 (Proctorio / Honorlock / Talview) | 상용 SaaS + 비디오·오디오 분석 | 응시자당 $5~15 | ★★ (웹캠 필요) | ★★ | ✗ (**초중등 프라이버시·학부모 민원**) | ★★★★ (영상·시선·음성 저장) |
| F8 | **교사 실시간 대시보드** (학생 상태 배지: 이탈·붙여넣기·무활동·완료 이벤트 집계 스트림) | 자체 구현 (Supabase Realtime 재사용, D축 연계) | 무료 (D1 인프라 재사용) | ★★★ | ★★★ | ★★★ | ★ (이벤트 메타데이터만, 화면·영상 無) |
| F9 | 키스트로크 / 타이핑 리듬 behavioral biometrics | 머신러닝 모델 | 오픈 연구 (TypingDNA 등 상용도) | ★ (S-Pen 필기 중심이므로 효용 ↓) | ★★ | ★ (초등 타이핑 표본 부족) | ★★ (지속 행동 데이터) |

### F 후보 상세

#### F1. Fullscreen API + Page Visibility API + focus/blur (**1순위 기층 — L1**)
- 장점(1): **웹 표준 무료** — Chrome Android Baseline since 2015. `document.visibilityState`로 탭 전환·앱 전환·화면 잠금 모두 감지. `onfullscreenchange`로 강제 풀스크린 해제 추적. 서버에 이벤트 스트림 전송 → 교사 대시보드·감사 로그로 활용.
- 장점(2): 프라이버시 비용 최저 — 영상·음성 0, **이벤트 메타데이터**(timestamp + 사건 유형)만 기록. 학부모 고지서 간결.
- 단점(1): **강제성 없음** — 감지만 할 뿐 학생 행동을 물리적으로 막지 못함. 이탈 시 답안 자동 제출·잠금 등 **앱 레이어 정책**으로 보완 필요.
- 단점(2): blur/focus는 포커스 전환만, visibilitychange는 "hidden" 상태만 — **"보이지만 다른 창이 위에 떠 있는"** 케이스(예: 분할화면) 감지 불가. 단, Android Chrome에서 탭 전환·앱 전환·스크린 락은 확실히 `hidden` 이벤트 발화.
- Aura 적합 지점: **L1 기본 탑재**. `AssessmentAttempt.events[]` 컬럼에 `{type: "hidden"|"focus_lost"|"fullscreen_exit", ts, duration}` 기록. 3회 이상 이탈 시 학생 화면에 "감독 교사에게 통보됨" 배너.

#### F2. Samsung Knox Configure / Knox Manage Kiosk
- 장점(1): OS 레벨 **Single App Kiosk / ProKiosk** — 학생이 Aura Board 이외 앱 실행 불가, 홈·최근앱·알림 셔터 차단. 갤탭 S6 Lite 네이티브 지원.
- 장점(2): Knox for School 교육 프로그램 — 학교 단위 대량 배포·프로파일 분리(교실 중엔 잠금, 방과후엔 해제) 등 성숙.
- 단점(1): **BYOD 불가 근접** — 학생 개인 단말을 MDM에 enroll 하려면 공장초기화·Samsung DPC 설치·학부모 동의 등 절차 무거움. 교실 시험 1회에 운영 현실성 없음.
- 단점(2): Samsung 전용 — 다른 Android(LG·샤오미 등) 미지원. 갤탭만 보유 학교에만 유효.
- Aura 적합 지점: **L4 선택지** — 학교 보유 태블릿 카트(예: 공립·학원 단체 구매) 환경엔 Knox Kiosk 토글 제공. BYOD 기준 환경에선 비활성.

#### F3. Safe Exam Browser (SEB)
- 장점(1): 대학 고부담 시험 표준, 오픈소스·무료.
- 장점(2): Moodle·ILIAS 등 LMS와 통합 성숙.
- 단점(1): **Android 공식 미지원** (SEB Consortium 공식 사유: 예산 부재 + Android 루팅 보안 리스크 높음).
- 단점(2): 데스크톱·iOS 배포 — 갤탭 S6 Lite 환경에 해당 없음.
- Aura 적합 지점: **해당 없음**. 벤치마크 참조만. (차후 데스크톱 Aura 확장 시 재등장 가능)

#### F4. Google Forms Locked Mode
- 장점(1): "탭 이탈 시 교사 이메일 통보" 등 UX 명료.
- 장점(2): Google Workspace for Education 기본 기능(무료).
- 단점(1): **Chromebook 전용** — Chrome OS 75+ + 학교 관리 Chromebook 필수. Android Chrome·태블릿 전면 미지원.
- 단점(2): Google Forms 제약 — 축 A SurveyJS 권고와 연결 불가.
- Aura 적합 지점: **참조 벤치**. "이탈 시 교사 통보" UX를 F8 교사 대시보드로 재현.

#### F5. Respondus LockDown Browser
- 장점(1): 상용 업계 표준, 1000+ 대학 도입.
- 장점(2): Respondus Monitor 결합 시 카메라 AI 감독까지.
- 단점(1): **Windows·Mac·ChromeOS 데스크톱 전용** — Android 태블릿·iPad·스마트폰 공식 미지원.
- 단점(2): 상용 라이선스 + 카메라 감독은 초중등 부적합.
- Aura 적합 지점: **해당 없음**. 데스크톱 시험실 모드 미래 확장 시만 고려.

#### F6. 커스텀 in-app 방어 조합 (**L1 심화 + L3**)
- 장점(1): **웹 원샷 배포 가능** — Aura Board 번들에 통합, 추가 앱 설치 0. `oncopy/oncut/onpaste` preventDefault, `contextmenu` 차단, `window.open` 감지·취소, DevTools 감지(`debugger` 루프·window size 비교 휴리스틱).
- 장점(2): **서버측 문항 설계 방어**(L3)가 OS 락보다 강력 — 문항풀에서 학생별 랜덤 N개, 보기 순서 셔플, 문항별 시드 기반 변형, 시간 제한, 부분 저장 락(submit 후 되돌리기 불가), Service Worker로 외부 도메인 fetch 화이트리스트.
- 단점(1): 클라이언트 JS 방어는 모두 **우회 가능** — 크롬 DevTools로 이벤트 리스너 제거 가능. 그래서 서버 검증(정답은 서버만, sanitized JSON 전달)이 필수.
- 단점(2): "일부 학생은 우회 기술 있음" 전제 — 방어는 **부정행위 비용 증가**가 목적이지 **원천 차단**이 아님. 교사 현장 감독으로 보완.
- Aura 적합 지점: **L1 + L3 핵심 스택**. 축 A의 sanitized JSON 전송 정책과 자연 결합.

#### F7. 카메라 기반 AI 원격 감독
- 장점(1): 시선 추적·입실 확인·이상 행동 AI 감지로 원격 고부담 시험 보호.
- 장점(2): 대학원·자격시험 레퍼런스 풍부.
- 단점(1): **초중등 프라이버시·학부모 민원 리스크 극대** — 카메라 녹화·얼굴 저장은 한국 개인정보보호법 민감정보 처리 동의 필요. 학부모 반발 예상.
- 단점(2): 교실 현장 감독 있는 환경에 **오버킬**. 수행평가 난이도 대비 비용·리스크 불균형.
- Aura 적합 지점: **파킹 권고**. `ideas-parking-lot.md` 이관.

#### F8. 교사 실시간 대시보드 (**2순위 — L2**)
- 장점(1): F1 이벤트 스트림을 **Supabase Realtime(D1) 채널**로 집계 → 교사 PC 대시보드에 학생별 상태 배지(녹색=정상, 주황=이탈 감지, 빨강=3회 이상). 축 D 인프라 재사용으로 추가 공수 ↓.
- 장점(2): **영상 없이 이벤트 메타데이터**만 — 프라이버시·한국 법제 호환성 최고. 학부모 고지 간단("시험 중 앱 이탈·붙여넣기 등의 행위를 감지한 시점·횟수를 기록합니다, 화면 촬영·녹화 없음").
- 단점(1): 교사가 PC 앞에 있어야 효용 최대 — 교실 이동 중엔 모바일 알림만.
- 단점(2): 이탈 감지 후 교사가 **현장 대면 확인** 필요 — 완전 자동 페널티는 오진 위험(Android Chrome 이벤트 오탐 가능, 예: 알림 배너로 일시 visibility 손실).
- Aura 적합 지점: **L2 핵심**. owner + 데스크톱 전용(매트릭스 방침과 일치). `/teacher/assessments/{id}/proctor` 라이브 탭.

#### F9. 키스트로크 / 타이핑 리듬 biometrics
- 장점(1): 영상 없이 개별 사용자 식별 가능 — 대리응시 감지 이론적으로 유용.
- 장점(2): 최근 LLM 복붙 감지 연구 활발.
- 단점(1): Aura 수행평가는 **S-Pen 필기** 중심 — 키보드 입력 표본 부족, 초등생 표본 자체가 불안정.
- 단점(2): 초중등 대상 **지속적 행동 데이터 수집**은 프라이버시 과다 — 학부모 동의 장벽.
- Aura 적합 지점: **파킹**. v3+ 고부담(중·고등 학력평가) 전환 시 재검토.

### F 축 권고 — 다층 방어 (L1~L4)

초중등 BYOD + 교사 현장 감독 + 갤탭 S6 Lite 조합에선 **OS 레벨 락 강제 불가**가 제약이다. 따라서 **부정행위 원천 차단**이 아닌 **부정행위 비용 증가 + 감지·기록 + 사후 조치**를 목표로 한 4-layer 다층 방어를 권고:

- **L1 (UX 신호 — F1 + F6 일부)**: Fullscreen API 요청 + Page Visibility API 감지 + focus/blur 로그 + paste/copy/cut 차단 + 컨텍스트메뉴 비활성 + `window.open` 차단. 모든 이벤트를 `AssessmentAttempt.events[]`에 기록.
- **L2 (교사 대시보드 — F8)**: Supabase Realtime으로 학생별 이탈·붙여넣기 감지·장시간 무활동 배지 + 이상행동 로그. 영상 감시 아님, 이벤트 메타데이터만. owner+데스크톱 전용.
- **L3 (문항 설계 — F6 서버측)**: 문항풀 랜덤화(학생별 N개 추출), 보기 순서 셔플, 지문 분할 출력, 시간 제한, 서버 전용 정답(클라이언트 sanitized JSON), 제출 후 재응시 락.
- **L4 (선택 — F2)**: Samsung Knox Kiosk를 **학교 배포 단말 모드**에서만 옵션 토글. BYOD 기본 배포엔 비활성. Aura Board 설정에 `school_managed_devices: boolean` 플래그.

**파킹**: F7(카메라 원격 감독) + F9(키스트로크 biometrics) — 초중등 프라이버시 리스크 과대, `ideas-parking-lot.md` 이관.

**결정 근거**: (1) BYOD 환경에서 Knox·SEB·LockDown·Locked Mode 모두 **배포 현실성 0~낮음**으로 확인, (2) Android Chrome은 Page Visibility·Fullscreen API가 Baseline 지원 → L1이 **무공수 기본 탑재**, (3) 교사 현장 감독 존재 → L2 대시보드가 **보조 증거 시스템**으로 충분, (4) L3 서버측 방어는 클라이언트 우회 성공해도 **점수 조작·답안 공유** 차단, (5) L4는 학교 단말 환경에만 토글 제공으로 선택권 확보.

---

## 교차축 통합 권고

### 권고 1안 — "웹 네이티브 + LLM Vision + 공통 DB + 다층 방어" (v1 default)

| 축 | 선택 | 근거 |
|---|---|---|
| A 객관식 | **SurveyJS Form Library (MIT)** | Aura React 번들 내장, iframe 0, 태블릿 TTI < 2s, 한국어 OK |
| B 입력·OCR | **tldraw/perfect-freehand 캔버스 필기 + Gemini 2.5 Flash Vision 통합 OCR** + 키보드 토글 | S-Pen 활용, OCR+채점 1 round-trip, 단절 시 큐잉 |
| C 서술형 채점 | **Gemini 2.5 Flash** (C1) + Sentence-Transformers 전처리(C6) + Promptfoo 회귀(C9) + 프로바이더 추상화 | 1M 컨텍스트로 학급 일괄, OpenAI·Claude 교체 가능 |
| D 성적 송신 | **공통 Supabase + RLS + Realtime Postgres Changes** (D1) | 두 앱 같은 Supabase 전제, ms 레벨 실시간, parent-viewer RLS 재사용 |
| E 성적부 UI | Classroom 표 구조 + Canvas 루브릭 popover + 하이클래스 학부모 뷰. **matrix는 owner+데스크톱 전용** | 국내 교사·학부모 UX 기대치 + 태블릿 격리 방침 |
| F 부정행위 방지 | **L1+L2+L3 기본 탑재** (Fullscreen/Visibility API + Supabase Realtime 교사 대시보드 + 문항풀 랜덤·셔플·시간제한) + **L4(Knox Kiosk)는 학교 단말 옵션 토글** | BYOD 기준 OS 락 불가, 갤탭 Chrome Baseline API 활용, 프라이버시 최소 침해, 카메라 감독(F7)·키스트로크(F9)는 파킹 |

**개발 공수**: 대(大) — 6~8주 (assignment-board 완료 가정) + L1·L2·L3 추가 1~1.5주.
**리스크**: Gemini 데이터 국외 전송 동의, LLM 판독 실패 UX, Supabase RLS 대량 change-feed 성능, L1 이벤트 오탐(알림 배너로 인한 visibility 일시 상실)으로 인한 교사 오판 — L2 대시보드는 감지만, 제재는 교사 대면 확인 후.

### 권고 2안 — "분리 배포 + PAT 재사용 + Clipo 외주" (Aura 웹앱이 독립 배포인 경우)

| 축 | 선택 | 근거 |
|---|---|---|
| A 객관식 | SurveyJS (동일) | 변경 없음 |
| B 입력·OCR | **CLOVA OCR(B2) + 키보드 토글** | 국내 SaaS 벤더, 한글 손글씨 공식 지원, LLM 의존도 분리 |
| C 서술형 채점 | **GPT-4o-mini(C2) 기본 + Claude Haiku 4.5(C3) 민감답 라우팅** | 벤더 이중화, 데이터 국외 이전 동의 범위 명확 |
| D 성적 송신 | **PAT 웹훅 재사용 (D2)** — `/api/external/grades` | canva-publisher-receiver 포크, 독립 배포 가능 |
| E 성적부 UI | 동일 (하이브리드) | 변경 없음 |
| F 부정행위 방지 | **L1+L3 동일** + L2는 웹훅 이벤트 스트림(별 배포)로 지연 집계 + L4 학교 단말 옵션 | D2 분리 배포 시 Realtime 구독 대신 주기적 웹훅 푸시로 대시보드 refresh |

**개발 공수**: 중 — PAT 재사용으로 보안 레이어 30% 절감, 대신 OCR·LLM 2-step 파이프라인 + L2 실시간성 저하(≈수초 지연).
**리스크**: OCR→LLM 2회 라운드트립 지연, CLOVA 가격 변동, L2 대시보드 ~5s 지연으로 교사 실시간 감독 UX 약화.

### 공통 필수 게이트 (phase2 sketch로 넘김)

1. **Aura-board ↔ Aura 웹앱 인프라 관계 질문**: 같은 Supabase 프로젝트인가, 완전 분리인가? → D1 vs D2 결정.
2. **LLM 데이터 국외 이전 정책**: Gemini/OpenAI/Claude 중 어느 벤더가 교육부·개인정보보호위 심사 통과했는가? → C1~C3 선택.
3. **학생 답안 단절 대응 큐**: 로컬 IndexedDB 큐 → 네트워크 복구 시 플러시. Aura-board 기존 SWR·idb-keyval 재사용(tablet-performance §3).
4. **"채점 제안 → 교사 확정" 상태머신**: `GradeSuggestion → Grade(confirmed)` 2-엔티티. assignment-board의 `AssignmentSlot.gradingStatus` 확장 vs 신규 엔티티. phase2에서 결정.
5. **학부모 노출 시점**: `Grade.confirmedAt` && `Grade.releasedAt` 이원화(Moodle 스타일) — 교사 확정 후 "공개" 토글 거쳐야 학부모 보임. parent-viewer v2 매트릭스에 `Grade` 행 추가.
6. **성적 보안 등급**: 최고 민감도 — Supabase RLS + DB 레벨 pgcrypto 암호화 + 접근 감사 로그.
7. **부정행위 방지 L1 이벤트 스키마**: `AssessmentAttempt.events[]` 컬럼 설계 — `{type, ts, durationMs, meta}`. 유형 enum 확정 (`visibility_hidden` · `focus_lost` · `fullscreen_exit` · `paste_blocked` · `copy_blocked` · `contextmenu_blocked` · `window_open_blocked` · `idle_over_threshold`). phase2 sketch에서 결정.
8. **L2 교사 대시보드 UX 허용 범위**: 이벤트 로그 표시만 vs 자동 알림 vs 자동 답안 락. 초등·중등 각각 정책 분리 여부. phase3 interview에서 사용자 결정.
9. **L4 Knox Kiosk 옵션 기능 노출**: 학교 계정 설정에 `school_managed_devices: boolean` 플래그 추가 여부 + 활성 시 MDM 연동 스펙(Android Enterprise Zero-Touch vs Knox Configure). v1에선 플래그만 두고 실 구현은 v2 파킹.
10. **프라이버시 고지 문구**: "시험 중 앱 이탈·붙여넣기 등의 이벤트를 기록합니다. 화면·카메라·음성 녹화는 없습니다." — 약관/개인정보처리방침 반영 + 학생·학부모 사전 안내.

---

## 참조 링크

### 축 A
- [SurveyJS Form Library GitHub (MIT)](https://github.com/surveyjs/survey-library)
- [SurveyJS Scored Quiz Documentation](https://surveyjs.io/form-library/examples/create-a-scored-quiz/documentation)
- [H5P Open Source](https://h5p.org/)
- [Moodle Quiz Alternatives 2026 — selecthub](https://www.selecthub.com/lms-software/moodle/alternatives/)
- [Quizizz vs Kahoot vs Socrative 2026](https://parental-control.flashget.com/best-kahoot-alternatives)

### 축 B
- [Google ML Kit Digital Ink Recognition Base Models (ko 지원 목록)](https://developers.google.com/ml-kit/vision/digital-ink-recognition/base-models)
- [ML Kit Digital Ink Android Integration](https://developers.google.com/ml-kit/vision/digital-ink-recognition/android)
- [Naver CLOVA OCR (Korean handwriting 공식 지원)](https://www.ncloud.com/v2/product/aiService/ocr/?language=en-US)
- [Upstage Document OCR Console](https://console.upstage.ai/docs/models/document-ocr)
- [MyScript iink SDK 4.3 Blog (2026-01)](https://www.myscript.com/blog/iink-sdk-3-0-handwriting-recognition/)
- [iinkJS GitHub](https://github.com/MyScript/iinkJS)

### 축 C
- [Gemini API Pricing 2026](https://aicostcheck.com/blog/google-gemini-pricing-guide-2026)
- [Claude Haiku 4.5 vs GPT-4o-mini vs Gemini Flash 2026 비교](https://skywork.ai/blog/claude-haiku-4-5-vs-gpt4o-mini-vs-gemini-flash-vs-mistral-small-vs-llama-comparison/)
- [Rubric-Conditioned LLM Grading 논문 (arxiv 2601.08843)](https://arxiv.org/html/2601.08843)
- [2026 Korean CSAT LLM Evaluation Leaderboard](https://www.emergentmind.com/topics/2026-korean-csat-llm-evaluation-leaderboard)
- [EXAONE 3.5 공식 Repo (LG AI Research)](https://github.com/LG-AI-EXAONE/EXAONE-3.5)
- [Sentence Transformers for Automatic Short Answer Grading (ResearchGate)](https://www.researchgate.net/publication/362965124_On_the_Application_of_Sentence_Transformers_to_Automatic_Short_Answer_Grading_in_Blended_Assessment)
- [Clipo 기능 페이지](https://clipo.ai/)
- [Promptfoo LLM Rubric Docs](https://www.promptfoo.dev/docs/configuration/expected-outputs/model-graded/llm-rubric/)
- [Gradescope AI-Assisted Grading Guide](https://guides.gradescope.com/hc/en-us/articles/24838908062093-AI-assisted-grading-and-answer-groups)

### 축 D
- [Supabase Realtime Postgres Changes](https://supabase.com/docs/guides/realtime/postgres-changes)
- [Supabase RLS Best Practices for Multi-tenant](https://makerkit.dev/blog/tutorials/supabase-rls-best-practices)
- [LTI 1.3 Assignment and Grade Services v2.0 (1EdTech)](https://www.imsglobal.org/spec/lti-ags/v2p0)
- [OneRoster 1.2 Gradebook Service (textScore 포함)](https://www.imsglobal.org/sites/default/files/spec/oneroster/v1p2/gradebook-informationmodel/OneRosterv1p2GradebookService_InfoModelv1p0.html)
- [PGMQ GitHub](https://github.com/pgmq/pgmq)

### 축 E
- [Canvas Gradebook 가로 스크롤 이슈 토론 (Instructure Community)](https://community.canvaslms.com/t5/Canvas-Question-Forum/Horizontal-scrollbar-obscuring-the-bottom-row-of-Gradebook/m-p/601162)
- [하이클래스 Google Play](https://play.google.com/store/apps/details?id=com.iscreammedia.app.hiclass.android&hl=en_US)
- [하이클래스 나무위키 (기능 요약)](https://namu.wiki/w/%ED%95%98%EC%9D%B4%ED%81%B4%EB%9E%98%EC%8A%A4(%EC%95%A0%ED%94%8C%EB%A6%AC%EC%BC%80%EC%9D%B4%EC%85%98))
- [쌤기부 (teacher-logbook.kr) — AI 수행평가·생기부](https://www.teacher-logbook.kr/)
- [교육부 수행평가 AI 활용 관리 방안 2026](https://www.moe.go.kr/boardCnts/viewRenew.do?boardID=294&boardSeq=104984&lev=0&searchType=null&statusYN=W&page=1&s=moe&m=020402&opType=N)

### 축 F
- [MDN — Page Visibility API (`visibilitychange`)](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API)
- [MDN — Fullscreen API (`requestFullscreen`, `fullscreenchange`)](https://developer.mozilla.org/en-US/docs/Web/API/Fullscreen_API)
- [Samsung Knox for School — Kiosk Mode (single/multi app)](https://www.samsungknox.com/en/solutions/knox-for-school)
- [Samsung Knox Manage Kiosk 설정 문서](https://docs.samsungknox.com/admin/knox-manage/kiosk-devices/kiosk-wizard/configure-kiosk-device-settings/)
- [Safe Exam Browser — Android 미지원 공식 입장](https://sourceforge.net/p/seb/discussion/844843/thread/b460af22/)
- [Google Forms Locked Mode — Chromebook 전용, 이탈 시 교사 이메일 통보](https://support.google.com/docs/answer/7634943?hl=en)
- [Respondus LockDown Browser — Android/iPad 미지원](https://info.navarrocollege.edu/en/knowledge/can-i-use-lockdown-browser-on-a-mobile-device-or-tablet)
- [IEEE — Online Examination with Question Bank Randomization + Tab Locking](https://ieeexplore.ieee.org/document/8912065/)
- [Synap — 15 anti-cheat methods for online exams](https://synap.ac/blog/anti-cheat-methods-for-online-exams/)
- [arXiv — Keystroke Dynamics Against Academic Dishonesty (LLM era)](https://arxiv.org/html/2406.15335v1)

### 기타 국내 현장·정책 배경
- [한국 수행평가 AI 채점 도구 비교 2026 (Coursebox)](https://www.coursebox.ai/ko/blog/coegoyi-ai-geureiding-dogu)
- [AI 디지털교과서 개발 가이드라인 (교육부)](https://webst.edunet.net/AIDT/AI%20%EB%94%94%EC%A7%80%ED%84%B8%EA%B5%90%EA%B3%BC%EC%84%9C%20%EA%B0%9C%EB%B0%9C%20%EA%B0%80%EC%9D%B4%EB%93%9C%EB%9D%BC%EC%9D%B8.pdf)

---

## 다음 단계 (phase2 sketch 입력)

1. 권고 1안을 base sketch로, 권고 2안은 **alternative sketch** 섹션에 병기.
2. 공통 필수 게이트 10개 항목을 sketch의 **"미결 질문" ≥ 6개** 섹션에 그대로 이관 (F축 4개 포함).
3. 데이터 모델 초안: `Assessment` (평가 정의) + `AssessmentQuestion` (SurveyJS JSON 래핑) + `AssessmentAttempt` (학생 응시, **`events[]` 부정행위 이벤트 로그 포함**) + `AssessmentAnswer` (문항별 답) + `GradeSuggestion` (AI 제안) + `Grade` (교사 확정, confirmedAt/releasedAt 이원화) + assignment-board `AssignmentSlot`와의 FK 관계. 학교 설정 엔티티에 `school_managed_devices: boolean` (L4 Knox 토글) 플래그.
4. 사용자 흐름: 교사 출제 → 학생 응시(태블릿, **L1 Fullscreen 진입 + 이벤트 수집**) → 객관식 즉시 채점 + 서술형 AI 제안 → **L2 교사 대시보드 실시간 배지(이탈·붙여넣기·무활동)** + 교사 리뷰 → 확정 → Aura 웹앱 성적 탭 실시간 반영 + 학부모 뷰(릴리스 게이트).
