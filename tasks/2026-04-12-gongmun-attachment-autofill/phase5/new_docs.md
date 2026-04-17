# Phase 5 — Newly Created Plan Documents

- **task_id**: `2026-04-12-gongmun-attachment-autofill`
- **seed_id**: `seed_3f953e443fa1`
- **created_at**: 2026-04-12

## 신규 생성

### 1. `ideation/plans/gongmun-assistant-v1-roadmap.md`

gongmun-assistant v1 전담 로드맵. Aura-board와 **완전 독립**한 외부 신규 프로젝트(`gongmun-assistant/`)의 설계 문서만 ideation에 보관.

**섹션 구성**:
- §0 핵심 명제
- §1 모듈 구조 (CLI 단일 런타임, 코어 중립) — 7개 레이어(cli·core·templates·llm·hwpx_io·config·types) + 레이어 경계 하드 룰
- §2 8단계 파이프라인 의사코드 (Stage 1 입력 수렴 → Stage 8 검증)
- §3 P0 범위 (명단표·동의서·가정통신문 × 2 학교급 = 6 템플릿) + v2+ 파킹 (P1·P2·GUI·학교 서버·CRUD 등)
- §4 개인정보 경계 정책 (3중 방어 — 패턴 스캐너·결정론 표 채움·LLM 로그 마스킹)
- §5 배포·운영 (pipx install + gongmun.bat + 대화형 + 포지셔닝 3축)
- §6 작업 분할 GM-1 ~ GM-10 (10개 feature 블록)
- §7 기반 스킬 참조 (hwpx-master)
- §8 리스크 R1~R9
- §9 변경 로그
- §10 외부 프로젝트 경계 (완전 독립 명시)

**핵심 특징**:
- 스키마(Prisma)·DB 없음 — CLI 툴이라 **모듈 구조**로 대체
- `seed.yaml`의 GongmunAttachmentSpec 9개 필드가 파이프라인 의사코드에 1:1 반영
- D1~D8 결정 전부 로드맵 확정 결정 표에 흡수
- 기반 스킬 `hwpx-master.SKILL.md` 명시 참조 (§1·§7)
