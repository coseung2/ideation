# Phase 0 capture notes — performance-assessment-autograde

## scope 판단 근거: `full_exploration`
본 주제는 독립적 하위 결정축이 최소 5개 이상 얽혀 있어 parking/research_only/quick_decision 어느 것도 부적합.

1. **채점 엔진 선택축** — 객관식(결정론) vs 서술형(LLM/키워드 루브릭/임베딩 유사도). 서술형 AI는 최소 3개 비교(OpenAI·Claude·한국어 특화 KoBERT/HyperCLOVA) 필요 → phase1 explorer 필수.
2. **응시 UX 입력축** — OMR 마킹 컴포넌트 + S-Pen 필기 + OCR(Google ML Kit / Tesseract / 네이버 CLOVA OCR) 비교. 갤탭 S6 Lite 성능 예산 필수 검증.
3. **성적 송신 채널축** — Aura-board↔Aura 웹앱 통신. (a) 공통 Postgres 직접 쿼리, (b) 신규 REST API, (c) canva-publisher-receiver(seed_26af361e92b7) PAT 패턴 승계 중 택1.
4. **권한·보안축** — 성적은 parent-viewer v2(seed_6d7077aac472) 매트릭스에서 "Quiz 점수 v1 비노출"로 명시되어 있어 충돌 해소 설계 필요. 교사↔학생↔학부모 RBAC 재정의 수준.
5. **Aura 웹앱 성적 탭축** — 신규 라우트·뷰·필터(학급별·과목별·기간별)·내보내기(엑셀/NEIS 호환?) 스펙 탐색 필요.

결정축 간 종속성이 강하고(채점 엔진 결정 → 송신 페이로드 구조 → Aura 웹앱 뷰 설계), 단일 quick_decision으로 답할 수 없음 → `full_exploration`.

## 중복·관련 시드 판정
- **assignment-board (seed_38c34e91bf28, 2026-04-14)**: layout="assignment"로 제출물 수거까지만 다룸. **확장 여지** 있으나 본 주제는 채점+성적 송신 레이어 신규 추가 → 중복 아님, phase1~2에서 Board.layout="assessment" 신규 vs layout="assignment" 확장 여부 결정 필요.
- **parent-viewer v2 (seed_6d7077aac472)**: 자녀 범위 매트릭스에서 "Quiz 점수 v1 비노출" 결정 존재. 본 주제 성적 탭과 정책 충돌 해소 필수 (refinement 트리거 가능성).
- **canva-publisher-receiver (seed_26af361e92b7)**: PAT/Rate Limit/Blob 저장 패턴이 "Aura-board → 외부 수신" 계약 템플릿. 본 주제의 "Aura-board → Aura 웹앱" 송신 채널 설계 참조 가능.
- **implementation-roadmap P1-④ Quiz 파싱 (seed_408901a564e8)**: Canva Quiz 파싱 3단 하이브리드가 존재. 출제 소스로 Canva Quiz 임포트 옵션 고려 여지 (phase1 확장 탐색).
- **ideas-parking-lot.md**: "수행평가/자동채점/성적/OMR" 키워드 매칭 0건 — 신규 주제 확정.

## target_user 유추 근거
사용자 프롬프트 "학생들 수행평가를 배부하고, 학생들은 태블릿 보드앱에서 OMR 마킹하듯 답을 적거나 서술형 답안을 적으면" — **학생**이 주 응시자, **교사**가 주 출제·확정자. 학부모는 프롬프트에 명시 없으나 Aura 웹앱 성적 탭 소비자로서 parent-viewer 매트릭스 연동 필요 여부를 phase1~3에서 결정하기 위해 부차 사용자로 포함.

## 송신 목적지(dispatcher 힌트, phase7 참조용)
- **padlet INBOX**: Aura-board(교실 앱) 측 출제·응시·채점 UI·스키마 확장
- **aura INBOX**: Aura 웹앱 측 신규 '성적' 탭 + 송신 수신 API
- 두 INBOX 모두 잠재 수신자 — phase6 handoff에서 산출물 분할 필요 가능성 높음.
