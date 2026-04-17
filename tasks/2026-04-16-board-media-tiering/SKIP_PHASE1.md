# SKIP — phase1 이후 전체

**날짜**: 2026-04-16
**사유**: scope = parking

phase0까지 진행 후 사용자가 "어렵다, 일단 보류" 판단. 결정할 하위 항목 7개가 동시에 튀어나와 피로감 발생.

## 현재 상태
- phase0/request.json + conversation_notes.md 완비
- 아키텍처 방향(2-Tier 티어링)만 합의, 세부 선택지 미정
- 원본 청크 분할(드라이브 분산)은 기각 확정

## 재개 조건 (트리거)
- Supabase 저장 한도 80% 이상 도달
- 학생 업로드 한도 불편 리포트 발생
- 보드 렌더링 렉이 썸네일 없이는 해결 불가능한 수준

## 재개 시 진입점
1. `phase0/conversation_notes.md` 읽기
2. 사용자에게 "미정 7개 중 우선순위" 묻기
3. 필요 시 phase1(explorer) 에이전트로 콜드 스토리지 공급자 비교 리서치부터

## 파킹 로트 등록
`ideas-parking-lot.md`에 "보드 미디어 2-Tier 저장" 섹션 추가 완료.
