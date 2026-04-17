# Handoff — gongmun-assistant v1 초기 모듈 구현 진입

- **task_id**: `2026-04-12-gongmun-attachment-autofill`
- **seed_id**: `seed_3f953e443fa1` (ambiguity 0.18)
- **destination**: `gongmun-assistant/` (신규 프로젝트, 자체 파이프라인 부재 → 본 request는 **초기 모듈 구현 진입용**)
- **request 포맷**: padlet phase0 스펙 준수 (파일명만 `project_phase0_request.json`)

## 배경

한국 교사의 공문 업무는 "본문 기안·결재 → 붙임 N종 작성" 2단계로 구성되지만, 후자 단계에서 본문 맥락과 붙임(명단·동의서·가정통신문)의 정합성을 자동 보장하는 도구가 없다. v1은 CLI로 이 구간만 특화하며, LLM은 공개 맥락 분석에만 쓰고 개인정보(명단 CSV)는 로컬 결정론 Python으로 분리 주입한다. 코어는 실행 모드 중립적으로 설계해 v2 GUI가 동일 코어를 재사용할 수 있게 한다.

## 참조 문서 필수 독해 순서

1. `ideation/plans/seeds-index.md` — 전체 seed 포트폴리오 내 본 task의 위치 확인.
2. `ideation/plans/gongmun-assistant-v1-roadmap.md` — v1 범위·Stage 1-8 파이프라인·P0 3종 스코프.
3. `ideation/tasks/2026-04-12-gongmun-attachment-autofill/phase4/seed.yaml` — 제약·수용 기준·ontology_schema·evaluation_principles의 단일 진실 원천.
4. `ideation/tasks/2026-04-12-gongmun-attachment-autofill/phase3/decisions.md` — D1-D8 결정과 근거(CLI 고정·이중 경로 수렴·하이브리드 템플릿·프라이버시 정책·배포 모드).
5. `gongmun-assistant/skills/hwpx-master.SKILL.md` — HWPX XML 조작 규칙(**linesegarray 삭제·문자열 조작 금지**, python-hwpx v2.9+ lxml 기반 API만 사용).
6. `gongmun-assistant/README.md` — 프로젝트 현재 상태·디렉토리 규약.
7. `ideation/tasks/2026-04-12-gongmun-attachment-autofill/phase5/updated_docs.md`, `new_docs.md` — 문서 동기화 델타.

## 기준 단말 · 제약

- Python >= 3.11, Windows 우선, macOS/Linux 호환.
- hwpx 포맷 전용, python-hwpx v2.9+ (lxml) 의존 — GPL/NC 라이선스 격리를 위해 개인 설치(pipx) 경로만 v1 지원.
- LLM: Claude Sonnet 기본, Haiku 폴백, Ollama opt-in. LLM 호출 직전 `llm/privacy.py`가 주민번호·전화·이메일·학번 패턴 스캔(매치 시 exit code 5).
- CLI 단일 실행 모드, 대화형 프롬프트 + `gongmun.bat` 더블클릭 런처.

## 이번 작업 (seed.goal)

> Build gongmun-assistant v1 — a CLI tool for Korean teachers that automatically generates hwpx attachment files (명단표·동의서·가정통신문) from official document (공문) context using Claude API and local deterministic Python.

## 수용 기준 체크리스트 (seed.acceptance_criteria 1:1 매핑)

- [ ] AC1: `gongmun attach --main main.hwpx` 가 hwpx를 파싱하고 Claude API로 context를 추출한다.
- [ ] AC2: 모든 입력(hwpx / 텍스트 파일 / stdin / `--prompt`)이 Stage 1에서 단일 `main_text: str` 로 수렴한다.
- [ ] AC3: 붙임 유형이 `--type auto` 로 자동 추론되거나 `--type roster|consent|parent-letter` 로 명시 지정된다.
- [ ] AC4: 보조 경로 `--type parent-letter --prompt "..."` 가 주 문서 없이 동작한다(가정통신문 전용).
- [ ] AC5: 시스템 번들이 6개 템플릿(P0 3종 × 초/중고 2학교급)을 포함하고 `scripts/validate_template.py` 로 검증된다.
- [ ] AC6: 사용자 템플릿을 `~/.gongmun/templates/user/{type}/{name}.hwpx` + YAML 사이드카에서 로드한다.
- [ ] AC7: 템플릿 로드 실패 시 경고를 출력하고 크래시 없이 스킵한다.
- [ ] AC8: `catalog.resolve(type, school_level, template_id?)` 가 template_path + slot_manifest를 반환한다.
- [ ] AC9: Windows `gongmun.bat` 래퍼로 더블클릭 실행이 가능하다.
- [ ] AC10: 인자 없이 실행 시 대화형 프롬프트 모드가 활성화된다.

## 주의사항

- **XML 조작 규칙 엄수**: `<linesegarray>` 요소 삭제·수동 생성·문자열 치환 금지. python-hwpx lxml 트리 API로만 슬롯 치환. 한컴오피스가 재계산할 수 있도록 원본 구조 보존.
- **Claude API 프라이버시 경계**: 명단 CSV·주민번호·전화·이메일·학번·결재자 실명은 API 페이로드에 진입 불가. `test_table_fill.py` 는 Anthropic 클라이언트 호출 0회 mock 단언을 포함해야 한다. opt-in 플래그 `--share-officer-to-llm` 외에는 결재자 정보 로컬 치환 only.
- **표 채움은 로컬 결정론**: 테이블 row 생성·슬롯 대입은 순수 Python으로 처리. LLM은 서술형 문단(`fill_narrative`)만 담당.
- **실패 처리 하드 룰**: exit code 0=성공, 2=hwpx 검증 실패, 3=LLM 실패, 4=템플릿 없음, 5=개인정보 패턴 감지. 원본 템플릿은 atomic save로 불변 유지.
- **스코프 경계**: 템플릿 CRUD 명령·웹/GUI·학교 서버 배포·P1/P2 붙임 유형은 v2+ 파킹. 현재 세션 편입 금지.
- **임의 결정 금지**: seed.acceptance_criteria와 phase3 decisions.md D1-D8 를 이탈하는 스펙 변경은 재인터뷰 필요.
