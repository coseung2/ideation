# gongmun-assistant v1 로드맵 (공문 붙임파일 자동 생성 CLI)

> 작성일: 2026-04-12
> Seed: `seed_3f953e443fa1` (task `2026-04-12-gongmun-attachment-autofill`)
> Interview: `interview_20260412_131559` (ambiguity 0.18, 3라운드 자동 종료)
> **Destination**: `gongmun-assistant/` — **ideation 외부 신규 프로젝트** (Aura-board와 무관)
> 기반 스킬: `gongmun-assistant/skills/hwpx-master.SKILL.md`
> 전제 문서:
> - `ideation/tasks/2026-04-12-gongmun-attachment-autofill/phase2/sketch.md` (8 Stage 파이프라인·파일 구조)
> - `ideation/tasks/2026-04-12-gongmun-attachment-autofill/phase3/decisions.md` (D1~D8)
> - `ideation/tasks/2026-04-12-gongmun-attachment-autofill/phase4/seed.yaml` (GongmunAttachmentSpec)

---

## 0. 핵심 명제

> **한국 교사가 공문(hwpx) 1건을 기안한 직후, CLI 1회 실행으로 붙임 3종(명단표·동의서·가정통신문)을 본문 맥락 일관되게 자동 생성한다. 본문 맥락만 Claude API에 전달하고, 명단 CSV·결재자 실명은 절대 LLM 경로로 흐르지 않는다. hwpx 출력은 한컴오피스에서 보안경고 없이 열린다.**

Aura-board(padlet 기반 교실 보드)와는 **완전 독립 프로젝트**. 교사 개인 설치(`pipx install gongmun-assistant`)로 동작하며, DB·웹서버·계정 없음. 코어(`gongmun.core`)는 실행 모드 중립으로 설계해 v2 Desktop GUI·v3 학교 공용 서버 재사용을 열어둔다.

---

## 1. 모듈 구조 (CLI 단일 런타임, 코어 중립)

```
gongmun-assistant/
├── pyproject.toml                 # poetry, entry_points → gongmun.cli:app
├── gongmun.bat                    # Windows 더블클릭 래퍼 (D1)
├── skills/
│   └── hwpx-master.SKILL.md       # 하드 룰 참조 — linesegarray 제거·mimetype 순서·section0 한정
├── src/gongmun/
│   ├── __init__.py
│   ├── cli/                       # [CLI] 실행 모드 계층 (교체 가능)
│   │   ├── app.py                 # typer entry — `gongmun attach ...`
│   │   ├── interactive.py         # 인자 없이 실행 시 대화형(inquirer)
│   │   └── bootstrap.py           # 첫 실행 시 ~/.gongmun 초기화 마법사 (D6)
│   ├── core/                      # [CORE] 실행 모드 중립 파이프라인
│   │   ├── pipeline.py            # Stage 1-8 오케스트레이션 (pipeline_stage 추적)
│   │   ├── hwpx_io.py             # python-hwpx 래퍼 + linesegarray 제거 + atomic save
│   │   ├── table_fill.py          # 결정론 표 채움 — LLM 금지 경계 (D5)
│   │   ├── narrative_fill.py      # 슬롯 치환 (LLM 결과는 텍스트로만 주입)
│   │   ├── validate.py            # zip·mimetype·xml 검증
│   │   └── errors.py              # exit code 0/2/3/4/5 매핑
│   ├── templates/                 # [TEMPLATES] 카탈로그 레지스트리
│   │   ├── catalog.py             # resolve(type, school_level, template_id?) → path + manifest
│   │   ├── loader.py              # 시스템 번들 + 사용자(~/.gongmun/templates/user/) 병합
│   │   └── sidecar.py             # {name}.yaml 사이드카 파서 (type·school_level·slots·schema)
│   ├── llm/                       # [LLM] provider 추상 (D4)
│   │   ├── client.py              # ABC: analyze_context / suggest_attachment_type / fill_narrative
│   │   ├── anthropic_provider.py  # Sonnet 기본, Haiku 폴백
│   │   ├── ollama_provider.py     # qwen2.5:14b-instruct-q4 기본
│   │   ├── privacy.py             # 주민번호·전화·이메일·학번 패턴 스캐너 (exit 5)
│   │   └── prompts/
│   │       ├── classify_attachment.md
│   │       ├── analyze_context.md
│   │       └── fill_narrative.md
│   ├── hwpx_io/                   # [HWPX-IO] python-hwpx 의존 격리 (R1 전환점)
│   │   ├── reader.py              # extract_text, find_slots, schema_from_table
│   │   ├── writer.py              # fill_by_path, atomic_save, strip_linesegarray
│   │   └── zip_rules.py           # mimetype ZIP_STORED, 첫 엔트리 보장
│   ├── config/                    # [CONFIG] ~/.gongmun/config.yaml (D6)
│   │   ├── schema.py              # pydantic: school·officer·approval_chain·llm
│   │   └── loader.py              # 첫 실행 셋업 마법사 진입점
│   └── types/                     # GongmunAttachmentSpec 구현 (pydantic)
├── templates/                     # 시스템 번들 (P0 6개 = 3 type × 2 school_level)
│   ├── roster/
│   │   ├── elementary-default.hwpx
│   │   └── middle-high-default.hwpx
│   ├── consent/
│   │   ├── elementary-default.hwpx
│   │   └── middle-high-default.hwpx
│   ├── parent-letter/
│   │   ├── elementary-default.hwpx
│   │   └── middle-high-default.hwpx
│   └── catalog.json               # 매니페스트 (type · school_level · slots · schema)
├── scripts/
│   └── validate_template.py       # 번들 추가 시 CI 수동 실행
├── tests/
│   ├── fixtures/*.hwpx
│   ├── test_privacy_boundary.py   # mock Anthropic — CSV 경로 호출 0회 단언
│   ├── test_table_fill.py
│   ├── test_hwpx_io.py
│   ├── test_catalog_resolve.py
│   ├── test_pipeline_e2e.py
│   └── test_validate.py
└── docs/
    ├── template-authoring.md      # {{snake_case}} · * 접두 필수 슬롯 규약 (D3)
    └── install-windows.md         # pipx + .bat 설치 가이드
```

**레이어 경계 (하드 룰)**:
- `cli/` → `core/` 단방향만 허용. `core/` → `cli/` 금지 (v2 GUI 재사용 보장).
- `core/` → `llm/` 호출은 ABC만 의존. Anthropic/Ollama 구현체는 config에서 주입.
- `table_fill.py`는 `llm/` 임포트 자체 금지 (정적 검사 `test_privacy_boundary.py`).
- hwpx 편집은 **오직 `hwpx_io/`** 통해서만 (python-hwpx NC 라이선스 블로킹 시 R1 전환 지점 1개로 격리).

---

## 2. 8단계 파이프라인 (의사코드)

```python
# gongmun.core.pipeline.generate_attachment
def generate_attachment(
    main_input: MainInput,        # hwpx | txt file | stdin | --prompt
    att_type: AttachmentType,     # roster | consent | parent-letter | auto
    csv_data: Optional[Path],
    template_id: Optional[str],
    config: Config,
) -> AttachmentResult:

    # ── Stage 1. 입력 수렴 (D2) — 모든 경로가 main_text: str 로 단일화
    main_text = converge_to_text(main_input)
    # hwpx → hwpx_io.reader.extract_text()  (section0 + header)
    # .txt → read_text(encoding="utf-8")
    # stdin/--prompt → 직접 사용 (parent-letter 한정, 다른 type은 거부)

    # ── Stage 2. 맥락 분석 (LLM, 비식별)
    privacy.assert_no_pii(main_text)      # 주민번호·전화·이메일·학번 패턴 → exit 5
    context = llm.analyze_context(main_text)
    # → DocumentContext{title, purpose, legal_basis, period, target, tone}

    # ── Stage 3. 붙임 유형 결정
    if att_type == "auto":
        att_type = llm.suggest_attachment_type(context)   # P0 3종 중
    # 명시 지정 시 LLM 미호출

    # ── Stage 4. 템플릿 해소 (로컬)
    template_path, slot_manifest, schema = templates.catalog.resolve(
        att_type, config.school.level, template_id
    )
    # 사용자 템플릿 로드 실패 → 경고 후 스킵, 시스템 번들 폴백 (크래시 금지)

    # ── Stage 5. 표 채움 (결정론 — LLM 금지 경계)
    doc = hwpx_io.reader.open(template_path)
    if csv_data is not None:
        rows = parse_csv(csv_data)
        validate_schema(rows, schema)
        hwpx_io.writer.fill_table_by_label(doc, rows)
    # ※ test_privacy_boundary.py: 이 스테이지에서 llm.* 호출 0회 단언

    # ── Stage 6. 서술 슬롯 채움 (LLM, 비식별 컨텍스트만)
    slots = hwpx_io.reader.find_slots(doc)  # {{snake_case}} · * 접두 필수
    narrative = llm.fill_narrative(
        context=context,
        slots=slots,
        officer=config.officer if config.share_officer_to_llm else None,  # 기본 False (D6)
    )
    hwpx_io.writer.apply_slots(doc, narrative)
    # ※ 결재자 실명은 로컬 치환 기본 — narrative 주입 이후 apply_approval_chain()

    # ── Stage 7. linesegarray 제거 + 원자적 저장 (hwpx-master 스킬 §3)
    hwpx_io.writer.strip_linesegarray(doc.section0)
    out_path = hwpx_io.writer.atomic_save(doc, output_dir)
    # mimetype ZIP_STORED + 첫 엔트리 보장 (zip_rules)

    # ── Stage 8. 검증
    report = validate.run(out_path)       # zip·mimetype·xml·linesegarray=0
    if not report.ok:
        raise HwpxValidationError(report)    # exit 2

    return AttachmentResult(
        output_hwpx_path=out_path,
        template_meta=slot_manifest,
        pipeline_stage=8,
    )
```

**경계 규칙**:
1. Stage 2·3·6만 LLM 허용. Stage 5는 정적 검사로 격리.
2. Stage 7 이전에 반드시 `strip_linesegarray` (한컴오피스 보안 경고 방지 — R2).
3. 실패 시 `.tmp/` 중간 산출물 유지(디버깅용), 성공 시 자동 정리.
4. 종료 코드: `0`=성공 · `2`=hwpx 검증 실패 · `3`=LLM 실패 · `4`=템플릿 없음 · `5`=개인정보 패턴 감지.

---

## 3. P0 범위 (v1 고정) + v2+ 파킹

### 3.1 P0 붙임 3종 × 2 학교급 = 시스템 템플릿 6개

| 유형 | school_level | 표 | 서술 슬롯 | 주 입력 경로 |
|---|---|---|---|---|
| **명단표** (roster) | elementary / middle-high | CSV 행(번호·이름·반·연락처) → 표 결정론 채움 | `{{*사업명}}`·`{{*기준일}}` | `--main` 필수 |
| **동의서** (consent) | elementary / middle-high | 없음 | `{{*목적}}`·`{{*수집항목}}`·`{{*보유기간}}`·`{{서명란}}` | `--main` 필수 |
| **가정통신문** (parent-letter) | elementary / middle-high | 없음 | `{{*머리글}}`·`{{*본문}}`·`{{유의사항}}`·`{{회신란}}` | `--main` OR `--prompt` 허용 |

**슬롯 규약** (D3):
- 토큰 포맷: `{{snake_case}}`
- 필수 슬롯: `*` 접두 (`{{*사업명}}`)
- 사이드카 YAML: `{name}.yaml` — `{type, school_level, slots: [...], schema: {...}}`

### 3.2 v2+ 파킹 (현재 세션 편입 금지)

| 항목 | 이관 시점 | 비고 |
|---|---|---|
| P1 붙임 3종 (평가표·품의서 내역서·참석자 명부) | v2 주 범위 | 표 구조 고정, 슬롯 자유도 중간 |
| P2 붙임 2종 (개별 결과통지서·보고서/회의록) | v3 | 서술 자유도 높음 → LLM 환각 관리 필요 |
| Desktop GUI (Tauri + Python sidecar / PyWebView) | v2 | `gongmun.core`가 실행 모드 중립 설계라 UI만 추가 |
| 학교 공용 서버 (다중 교사 격리·SSO EDU-ID) | v3+ | python-hwpx NC 라이선스 블로커 해결 선행 |
| `gongmun template add/remove/validate` CRUD | v2 | v1은 디렉토리 규약만 |
| 슬롯 자동 탐지 | v2 | v1은 사이드카 YAML 수기 선언 |
| 한컴어시스턴트 연동 | 추후 검토 | 본문 생성 영역 중복 방지 필요 |
| 즉시 revoke (<1s, Redis 블랙리스트) | — | 해당 없음 (계정 없음) |
| 템플릿 마켓플레이스·공유 | v3+ | |

---

## 4. 개인정보 경계 정책

### 4.1 데이터 분류

| 데이터 | 허용 경로 | 금지 경로 | 가드 |
|---|---|---|---|
| 주 공문 본문 (비식별) | Claude/Ollama 전송 | — | `privacy.py` 패턴 스캔 → 매치 시 exit 5 |
| 학생·학부모 명단 CSV | 로컬 메모리·디스크만 | LLM 전송 **금지** | `table_fill.py`에서 `llm/` 임포트 금지, `test_privacy_boundary.py`가 mock Anthropic 호출 0회 단언 |
| 결재자·담당자 실명 (`~/.gongmun/config.yaml`) | 로컬 치환 기본 | LLM 전달 opt-in (`--share-officer-to-llm`) | 문서에 리스크 명시 |
| LLM 요청 로그 | 기본 off. `--debug` 시 로컬 파일만 | 외부 수집 **금지** | 저장 전 동일 마스킹 파이프라인 재적용 |
| 주민번호·전화·이메일·학번 패턴 | — | LLM 프롬프트 **금지** | 호출 직전 스캐너, 매치 시 예외 |

### 4.2 감사·정리

- 명단 CSV 자동 삭제 **하지 않음** — 교사 재실행·검증 업무 패턴 (D5).
- `.tmp/` 중간 산출물만 성공 시 자동 정리. 실패 시 디버깅 유지.
- LLM provider 전환(`llm.provider: anthropic|ollama`)으로 민감 기관 정책 대응 (학교망 방화벽·외부 API 금지).

### 4.3 정적 검사

- `test_privacy_boundary.py`:
  1. `core.table_fill` 모듈 임포트 그래프에 `llm.*` 미포함 단언.
  2. E2E 파이프라인 실행 중 `anthropic.Client` mock 호출 수집 → 명단 CSV 행 문자열이 payload에 0회 단언.
  3. `privacy.assert_no_pii` 단위 테스트 — 주민번호 13자리·전화 010-xxxx-xxxx·이메일·학번 5~8자리 매칭.

---

## 5. 배포·운영

### 5.1 설치 (D7)

- **PyPI 패키지**: `pip install gongmun-assistant` 또는 `pipx install gongmun-assistant` (권장 — 전역 오염 없음).
- **Python 요구**: >= 3.11.
- **첫 실행**: `~/.gongmun/` 초기화 + config 셋업 마법사.
- **Windows**: `gongmun.bat` 자동 PATH 등록 → 탐색기 더블클릭 시 터미널 진입.
- **명시적 비포함**: 다중 교사 격리·학교 공유 서버·SSO.

### 5.2 실행 예시

```bash
# 주 경로 — 기존 공문에서 붙임 자동 분류
gongmun attach --main ./2026-공문.hwpx --data ./students.csv

# 유형 명시
gongmun attach --main ./공문.hwpx --type consent

# 가정통신문 — 본문 없이 프롬프트만 (parent-letter 한정)
gongmun attach --type parent-letter --prompt "수학여행 안전 안내"

# 대화형 (인자 없이)
gongmun
```

### 5.3 포지셔닝 (D8)

- 나라에듀·한컴어시스턴트와는 **보완** 관계.
- 차별 3축: ① 붙임 특화 ② LLM 맥락 파악(본문 → 붙임 자동 분류·슬롯) ③ 로컬 실행·개인정보 분리.
- 진입점: 공문이 나라에듀에서 결재 완료된 직후, 붙임 첨부 직전.

---

## 6. 작업 분할 (GM-1 ~ GM-10)

| ID | 제목 | 선행 | 핵심 산출 |
|---|---|---|---|
| **GM-1** | 프로젝트 스캐폴드 + pyproject + `gongmun.core` 모듈 경계 | — | `src/gongmun/{cli,core,templates,llm,hwpx_io,config,types}/` 빈 패키지 + 레이어 임포트 정적 검사 CI |
| **GM-2** | `hwpx_io/` — reader·writer·zip_rules (python-hwpx 래퍼) | GM-1 | extract_text·find_slots·fill_by_path·strip_linesegarray·atomic_save. hwpx-master 스킬 §3·§4 체크리스트 단위 테스트 |
| **GM-3** | `llm/` provider ABC + Anthropic + Ollama 구현체 + `privacy.py` | GM-1 | `LLMProvider` ABC(analyze_context·suggest_attachment_type·fill_narrative) + 두 구현체 시그니처 동일 + 주민번호·전화·이메일·학번 스캐너 |
| **GM-4** | `templates/catalog.py` + 시스템 번들 6개 + 사이드카 YAML | GM-2 | `catalog.resolve(type, school_level, template_id?)` 단위 테스트 · 사용자 템플릿 경고 스킵 경로 · `scripts/validate_template.py` |
| **GM-5** | `core/table_fill.py` — 결정론 표 채움 + 정적 격리 | GM-2 | `fill_table_by_label(doc, rows)` + schema 검증 + `test_privacy_boundary.py` (llm 임포트 금지 + mock 호출 0회) |
| **GM-6** | `core/narrative_fill.py` + `core/pipeline.py` Stage 1-8 오케스트레이션 | GM-3·GM-4·GM-5 | `generate_attachment()` + exit code 0/2/3/4/5 + `.tmp/` 수명 관리 |
| **GM-7** | `config/` 스키마 + 첫 실행 셋업 마법사 + `~/.gongmun/config.yaml` | GM-1 | school·officer·approval_chain·llm.provider 필드 · opt-in `--share-officer-to-llm` |
| **GM-8** | `cli/app.py` typer 커맨드 + 대화형 폴백 + `gongmun.bat` | GM-6·GM-7 | `gongmun attach` · 인자 없이 실행 → inquirer 프롬프트 · Windows 더블클릭 래퍼 |
| **GM-9** | `core/validate.py` + E2E 테스트 (P0 3종 × 2 학교급) | GM-6 | zip 무결성·mimetype ZIP_STORED·첫 엔트리·XML well-formedness·linesegarray=0. 한컴오피스 수동 QA 케이스 문서화 |
| **GM-10** | PyPI 배포 파이프라인 + 설치 가이드 + README | GM-8·GM-9 | `pipx install gongmun-assistant` 동작 · `docs/install-windows.md` · `docs/template-authoring.md` |

**수용 기준 매트릭스**: `seed.yaml#acceptance_criteria` 10항목을 GM-1~GM-10 각 작업의 DoD로 분해 — phase9 QA에서 역추적.

---

## 7. 기반 스킬 참조

- **hwpx-master** — `gongmun-assistant/skills/hwpx-master.SKILL.md`
  - §3 `<linesegarray>` 제거 규칙 (Stage 7)
  - §4 mimetype ZIP_STORED + 첫 엔트리 (zip_rules.py)
  - 문자열 조작 금지 · `lxml.etree` API 전용
  - `Contents/section0.xml`에 한정 편집
- phase1 exploration.md — python-hwpx v2.9+ 라이선스(NC) 리스크 및 lxml 직접 구현 폴백 경로
- phase2 sketch.md §6 프라이버시 경계 · §7 리스크 표 R1~R9

---

## 8. 리스크 (요약)

| # | 리스크 | 완화 |
|---|---|---|
| R1 | python-hwpx NC 라이선스 — 학교 공유/상용 배포 블로커 | v1은 "교사 개인 용도" 명시 + `hwpx_io/`로 의존 격리(교체 지점 1개) |
| R2 | `<linesegarray>` 누락 → 한컴오피스 보안경고 | Stage 7 자동 제거 + Stage 8 검증 + 수동 QA |
| R3 | Claude API 장애·비용 | Sonnet→Haiku 폴백 · Ollama 전환 · 본문 해시 캐시 |
| R4 | LLM 환각 (본문에 없는 사실) | 프롬프트 "본문 외 사실 금지" + 원문 키워드 커버리지 검사 + v1 P0 3종 한정 |
| R5 | 교사가 main_doc에 학생 이름 포함 | `privacy.py` 패턴 스캐너 + CLI 시작 경고 |
| R6 | 템플릿 catalog 관리 비용 | catalog.json 스키마 고정 · 추가 시 `validate_template.py` 필수 · v1 6개 제한 |
| R7 | hwpx 재패키징 순서·압축 실수 | python-hwpx atomic save · zip_rules 단위 테스트 |
| R8 | 결재자 실명 LLM 유입 | 기본 로컬 치환 · opt-in `--share-officer-to-llm` |
| R9 | 교사 CLI 미숙 | `gongmun.bat` 더블클릭 + 대화형 프롬프트 기본 |

---

## 9. 변경 로그

| 날짜 | 시드 | 변경 |
|---|---|---|
| 2026-04-12 | `seed_3f953e443fa1` (ambiguity 0.18) | 신규 생성. D1~D8 확정 사항 반영 (CLI 단일·이중 경로 단일 수렴·하이브리드 템플릿·Claude 기본 Ollama 옵션·로컬 고정 개인정보·config 로컬 고정·개인 설치·붙임 특화 포지셔닝). P0 3종 × 2 학교급 = 6 템플릿. GM-1~GM-10 작업 분할. |

---

## 10. 외부 프로젝트 경계

- **이 로드맵은 Aura-board(ideation / padlet)와 무관** — 코드·DB·인증·배포 채널 전부 분리.
- `destinations/_registry.md` "gongmun-assistant" 항목 기준으로 `ideation/` 외부 `gongmun-assistant/INBOX/`에 산출물 전달.
- seeds-index 의존성 다이어그램에서도 **완전 독립 노드**로 표기 (Seed 1~8 어느 것과도 화살표 없음).
