# 아이디어 파킹 로트

> 지금은 안 하지만 나중에 꺼내볼 수 있게 보관.

---

## 보드 미디어 2-Tier 저장 (board-media-tiering)

> 출처: `tasks/2026-04-16-board-media-tiering/`
> 결정: **파킹**. 결정해야 할 하위 항목이 많아 피로. 실제 용량 한계에 가까워질 때 재개.

학생·교사가 보드에 이미지·영상을 더 많이 올릴 수 있게 **핫(Supabase 썸네일) / 콜드(R2·B2 원본) 2-Tier 분리**. 보드 렌더링 시 썸네일만 로드, 클릭 시 원본 스트리밍. 동기는 **C(Supabase 저장 비용)**, 보너스로 **B(보드 열 때 렌더링 렉)** 도 해결.

원래 영감은 AWS S3 HDD 병렬 저장(Erasure Coding 5+4, Power of Two Choices)이었으나, 노드 3~5개 규모에선 청크 분할 이득 없고 Google/OneDrive ToS 위반 소지 + 태블릿 재조립 부하로 **청크 분할 방향은 기각**. S3에서 차용할 부분은 **티어링 사상**뿐.

### 합의된 방향
- 핫: Supabase에 썸네일(이미지 ~30KB WebP, 비디오 포스터 + 10초 프리뷰 ~500KB) + 메타데이터
- 콜드: R2($15/TB) 또는 B2($6/TB)에 원본 통짜. 이그레스 무료 경로 확보
- 클라이언트 리사이즈 후 업로드, 비디오는 HLS 변환 고려
- 인덱스 DB: `{thumb_url, original_url, bytes, uploaded_at}`

### 꺼낼 시점 트리거
- Supabase 저장 한도 80% 이상 도달
- 또는 학생 업로드 한도 때문에 수업 중 실제 불편 리포트 발생
- 또는 보드 열 때 태블릿 렌더링 렉이 썸네일 없이는 풀 수 없는 수준까지 악화

### 재개 시 결정할 7가지
1. 콜드 스토리지 공급자 (R2 / B2 / Tigris / 기타)
2. 썸네일 파이프라인 위치 (클라이언트 / Supabase Edge Function / 별도 서비스)
3. 비디오 트랜스코딩 위치 (태블릿 / 클라우드 함수 / ffmpeg 서버)
4. 접근 제어 (signed URL TTL, viewer 권한 검증)
5. 마이그레이션 (기존 Supabase 원본 → 콜드 이관 + 롤백)
6. Free/Pro 정책 정합성 (학생별 한도)
7. 실패 모드 (콜드 장애 시 graceful degradation)

### 참고
- https://bigdata.2minutestreaming.com/p/how-aws-s3-scales-with-tens-of-millions-of-hard-drives
- https://www.youtube.com/watch?v=JpPbeO_TwWg ("HDD에 초당 1petabyte 저장하기")
- `tasks/2026-04-16-board-media-tiering/phase0/conversation_notes.md` (상세 논의 기록)

---

## 게임 제작 보드 (마인크래프트 느낌)

"학생이 마인크래프트 같은 게임을 직접 만들 수 있는 공간" 탐색.

### 해석 4갈래

| 해석 | 경험 | 도구 | 태블릿 | 개발 부담 |
|---|---|---|---|---|
| A. 월드 빌딩 | 블록 쌓아 건축 | voxel.js, Luanti(Minetest) | 🔴 | 매우 높음 |
| B. 게임 로직 제작 | 블록 코딩으로 규칙·캐릭터 | MakeCode Arcade | 🟢 | 낮음 |
| C. 복셀 모델링 | 캐릭터/아이템 복셀 제작 | Blockbench | 🟡 | 중간 |
| D. 모드 제작 | 기존 게임 확장 | Minetest Lua | 🔴 | 초과 |

### 추천 진입 (꺼낼 때)

1. **V1 — MakeCode Arcade 임베드 카드**: 공식 iframe embed 제공. Canva oEmbed 수준 개발 난이도. 태블릿 OK. "우리 반 아케이드" 보드에 학생 게임 모아 플레이.
2. **V2 — Blockbench 복셀 갤러리**: 학생 복셀 모델 전시. gltf 뷰어.
3. **V3 — voxel.js 자체 월드**: 야심차지만 태블릿 안 맞고 개발 헤비. 외부 Minecraft Education 쓰라고 권하는 편이 현실적.

### 파생 아이디어
- 게임 잼 보드 (주간 주제)
- 플레이 리뷰 보드 (영상·감상평)
- 캐릭터 설정집 보드
- 게임 기획서 보드 (실제 제작 전 기획 단계)

### 참고 링크
- https://arcade.makecode.com/share
- http://voxel.github.io/voxeljs-site/
- https://www.luanti.org/en/

---

## mallang-ranch-p2e — Apple App Store iOS 네이티브 빌드 (v2)

> 출처: `tasks/2026-04-13-mallang-ranch-p2e/phase3/decisions.md` N2
> 결정: **v2 파킹**. Season 0 MVP는 PWA 웹앱 전용.

토큰·NFT 제거 F2P 빌드를 별도 분기로 만들면 Apple App Store에 등재 가능. 단 1인 9개월 타겟에 2~3개월 추가 공수가 들어가 과도. Season 0 출시 후 DAU·매출 안정화 시점에 v2 검토.

**꺼낼 시점 트리거**:
- Season 0 launch 후 6개월 + 안정적 DAU 확보
- 또는 Apple App Store P2E 정책 명확화

**예상 공수**: 2~3개월 (F2P 빌드 분리 + Apple 심사 대응)

**참고**:
- Sunflower Land·Pixels 모두 PWA 우선 + 모바일 앱은 후순위
- 위메이드 이미르 한국 차단 + 글로벌 PWA 모델 전례

---

## mallang-ranch-p2e — Immutable zkEVM Passport 심사 결과 대응 (followup)

> 출처: `tasks/2026-04-13-mallang-ranch-p2e/phase3/decisions.md` N3
> 결정: **research followup investigation**. 기본 채택은 Base + Coinbase Smart Wallet, Immutable Passport는 심사 통과 시 2순위 조건부.

Immutable zkEVM Passport는 패스키 기반 월렛 추상화 + 게임 친화 인프라. 단 게임 심사 통과가 전제. 심사 통과 시 Base 체인에서도 Passport 월렛 활용 가능한지 (크로스체인 가능성) 추가 리서치 필요.

**꺼낼 시점 트리거**:
- Immutable Games 심사 결과 수령
- 또는 Coinbase Smart Wallet UX 한계 발견 시

**리서치 항목**:
- Passport 월렛이 Base 컨트랙트와 호환되는가 (sponsored tx 포함)
- Passport 인증 → Base 트랜잭션 서명 우회 경로 존재 여부
- Immutable Games Hub 리스팅 조건 (배포 전제·로열티·수수료)

**결과 후 행동**:
- 심사 통과 + 호환 확인: Passport를 Coinbase Smart Wallet과 병행 옵션으로 추가
- 심사 실패 또는 비호환: Base + Coinbase Smart Wallet 단일 유지

---

## parent-viewer v2 refinement — 학급 코드 + 셀프매칭 + 교사 승인 (v2+ / P2 파킹)

> 출처: `tasks/2026-04-13-parent-class-invite-refine/phase3/decisions.md` (seed_6d7077aac472, parent_seed_id=seed_37b35654542f)
> 본 시드는 parent-viewer v1(학생별 코드)를 대체하는 v2로 발행됨. 아래는 **v2 본배포 이후로 이연**된 세부 항목.

### 교사 자유 메시지 입력 (거부 이메일 커스텀 문구) — v2

- v1은 `wrong_child`/`not_parent`/`other` 사유 3종 드롭다운 고정 문구만 제공 (D-28·D-29).
- 교사가 거부 이메일에 맥락을 설명할 수 있도록 자유 텍스트 입력 허용 여부 검토.
- **차단 요인**: 모더레이션 리스크 (교사 개인정보 유출·부적절 문구 가능성) + 1인 개발자 운영 부담.
- **꺼낼 트리거**: 교사 1,000명 이상 운영 단계 + CS 프로세스 확보 + 모더레이션 자동화(LLM 필터) 가능 시점.

### 에스컬레이션 경로 (SLA 초과 시 학급장/관리자 알림) — v2

- v1은 권고 SLA 24h / 자동 만료 7일만 제공. hard SLA·에스컬레이션 없음 (D-21·D-23).
- 72시간 초과 pending에 대해 학교 관리자·학년부장 등 상위 권한자에 배지/이메일 발송 검토.
- **차단 요인**: 1인 개발자 운영, 에스컬레이션 수신자 RBAC 부재.
- **꺼낼 트리거**: 학교 단위 Enterprise tier 도입 시점 + 학교 관리자 역할 엔티티 추가.

### 사칭 감지 SOP 고도화 — P2

- v1은 "거부율 임계 초과 시 교사 배지" + 교사 수동 회전 버튼만 제공 (D-14·D-48).
- 패턴 감지 (IP·기기·이메일 도메인·연속 거부율) 알고리즘화, 자동 경고·자동 회전 트리거 도입 검토.
- **꺼낼 트리거**: 실제 사칭 시도 통계 축적(300건+) + 통계 기반 임계 튜닝 가능 시점.

### Kakao/Google OAuth · 비밀번호 로그인 — v2+

- v1 매직 링크(이메일 OTP) 전용 유지 (D-38 유지, parent-viewer v1부터 동일).
- 학부모 재인증 마찰 감소용 OAuth 도입 검토.
- **꺼낼 트리거**: Kakao Developers 사업자 인증 완료 + 국내 학부모 70%+ 카톡 계정 기반 확보.

### 부/모 합산 권한 통합 — v2+

- v1은 동일 자녀에 부·모 각각 **별도 승인** (`@@unique([parentId, studentId])`만 적용, D-08).
- 가정 단위 집계·공동 수신 이메일 옵션 검토.
- **차단 요인**: "가정" 엔티티 부재, 이혼·별거 가정 프라이버시 고려.
- **꺼낼 트리거**: Parent 앱 네이티브 도입 시점.

### 학부모 앱 네이티브 (Flutter/React Native) — v3+

- v1 웹 모바일 PWA 전용 유지.
- 푸시 알림·오프라인 캐시·생체 인증 도입을 위한 네이티브 검토.
- **꺼낼 트리거**: PWA MAU 10만+ 도달 + 푸시 수요 확인.

---

## 수행평가 자동채점 파이프라인 (assessment-autograde, Seed 12) — v1.5/v2/v3+ 파킹

> 출처: `tasks/2026-04-16-performance-assessment-autograde/phase3/decisions.md` §2 + `phase4/seed.yaml` exit_conditions(scope_overflow)
> 본 시드(`seed_0badf1e571bc`, 2026-04-16)는 MCQ + SHORT 2종·L1 자동 화면이탈 잠금·Pro 전용(런칭 플래그 off 출시)로 확정. 아래는 **v1에서 의도적으로 제외**한 항목.

### ESSAY(논술·장문 서술) UI 활성 — v1.5 베타 플래그

- schema는 5종(`MCQ·OX·NUMERIC·SHORT·ESSAY`) 유지하되 v1 create API는 MCQ·SHORT만 Zod gate 통과.
- **차단 요인**: OCR 판독 불가·Gemini LLM 오채점 리스크·이의제기 플로우 파일럿 필요·LLM 비용 상한 관리 불확실.
- **꺼낼 트리거**: Promptfoo SHORT 회귀 기준선 확립 + MAU 안정 + `/admin/llm-cost` 대시보드에서 SHORT 실비 확보 이후 ESSAY 파일럿 반 선정.

### 자동 잠금·자동 제출 (부정행위 정책 A·C) — v1.5 요청 시

- U3 인터뷰에서 사용자 확정: **"알림 + 자동 화면이탈 잠금"** 하이브리드 채택. 그러나 **정책 A(즉시 자동 제출)** 및 **정책 C(임계값 복합 전이)** 는 v1에서 배제.
- **차단 요인**: 오탐(false positive) 리스크·학부모 민원 감내 범위 미확정·BYOD 기본 환경.
- **꺼낼 트리거**: 실제 운영 데이터(ProctorEvent 로그 6개월+) 축적 + 교사 요청 누적 시 임계값 복합 정책 C 검토.

### 부정행위 임계값 교사 조정 UI — v1.5

- v1은 🟢 0~1 / 🟠 2 / 🔴 3+ **고정 임계값**.
- **꺼낼 트리거**: 교사 요청 누적 + 학급별 맥락 차이 통계 확보.

### Knox Kiosk 실제 MDM 연동 — v2+ Enterprise

- v1은 UI 플래그(`Classroom.schoolManagedDevices && AssessmentTemplate.kioskMode`) + 가이드 문서만. **실 MDM 연동 코드 0**.
- **차단 요인**: BYOD 기본 환경·학교 단말 카트 보급 전제.
- **꺼낼 트리거**: Enterprise tier 학교 단위 판매 개시 + Knox Manage 파트너십 체결.

### Capacitor "Aura Board Tablet" 네이티브 셸 + ML Kit Digital Ink — v2

- v1은 순수 웹(Chrome Android + S-Pen) 유지 방침.
- **꺼낼 트리거**: PWA 성능 상한 도달 + 네이티브 Digital Ink 필요 수요 확인.

### OneRoster 1.2 Gradebook Service (D7) · LTI 1.3 AGS (D6) — v2+ 외부 연동

- v1은 Aura-board ↔ Aura 웹앱 내부 통신만 대상. 외부 SIS/LMS 연동 오버킬 판정.
- **꺼낼 트리거**: 학교 단위 외부 LMS 도입 요청 (NEIS·Canvas·Moodle).

### 쌤기부-style 생기부 문장 AI 생성 — v1.5 별 task

- 수행평가 확정 → LLM 문장 초안 생성 → 교사 편집 → NEIS export 후속 파이프라인.
- **분리 사유**: 현 assessment-autograde 스코프 이탈. 별 task로 Ouroboros 인터뷰 후 진행.
- **꺼낼 트리거**: assessment-autograde v1 배포 후 최소 1학기 운영 경험 + 교사 수동 생기부 업무 패턴 확인.

### 카메라 기반 AI 원격 감독 (F7) — 파킹(우선순위 낮음)

- 학생 탭 카메라로 얼굴·표정·시선 분석.
- **차단 요인**: 초중등 프라이버시·학부모 민원·개인정보보호법 동의 리스크.
- **꺼낼 트리거**: 중·고등 고부담 평가 + 학부모 명시 동의 프로세스 구축 시 검토(그때도 opt-in만).

### 키스트로크 biometrics (F9) — v3+ 중·고등 고부담 평가

- 타이핑 패턴으로 본인 확인.
- **차단 요인**: 초등 S-Pen 중심이라 키보드 표본 부족·초등에는 과잉 사찰 인상.
- **꺼낼 트리거**: 중·고등 수능 모의고사·교육청 시험 연동 시.

### 교사 현장 감독용 모바일 대시보드 — 별 task

- 매트릭스 뷰는 owner + 데스크톱 전용 원칙을 유지하면서, 교사가 교실을 돌아다니며 태블릿·모바일로 proctor 이벤트 확인하고 싶은 요구.
- **차단 요인**: 현 owner+데스크톱 원칙과 충돌. 별도 설계 task 필요.
- **꺼낼 트리거**: 교사 사용자 조사에서 수요 30%+ 확인 시.

### 이메일·문자 학부모 알림 — v1.5

- v1은 Aura 웹앱 PWA 푸시(OneSignal 또는 Supabase Edge + FCM)만.
- **꺼낼 트리거**: parent-viewer v2 주간 이메일 인프라와 통합 시점.

### Upstage Document OCR / CLOVA OCR (B2·B3) — LLM 비용 폭주 시 fallback

- v1은 Gemini Vision 단일 경로(SHORT inkImageUrl → Vision 1-round-trip).
- **꺼낼 트리거**: Gemini Vision 월 비용이 Pro ARR 대비 임계(예: 20%) 초과 시 전처리 단계 fallback 검토.

---

## (앞으로 다른 파킹 항목은 이 아래에 섹션으로 추가)
