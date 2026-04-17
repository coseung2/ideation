# Phase 2 Sketch: 공문 붙임파일 자동 생성 도구 (gongmun-assistant v1)

- **task_id**: `2026-04-12-gongmun-attachment-autofill`
- **기반 문서**: phase0/request.json, phase1/exploration.md §5 권고
- **기반 스킬**: `gongmun-assistant/skills/hwpx-master.SKILL.md`
- **작성일**: 2026-04-12

---

## 0. 전제 결정 (phase1 권고 고정)

| 축 | 결정 | 비고 |
|---|------|-----|
| hwpx 편집 | **python-hwpx v2.9+** (lxml 기반) | hwpx-master 스킬 규칙과 1:1 매칭 |
| LLM — 본문 분석·붙임 본문 생성 | **Claude API (Sonnet 기본, Haiku 경로 옵션)** | 본문은 비식별 정보만 포함 |
| LLM — 표 채움(명단·점수·연락처 등 개인정보 포함) | **로컬 결정론 Python (LLM 미경유)** | 개인정보 외부 전송 0 |
| 대상 포맷 | **hwpx 단일** (hwp·docx 제외) | v1 스코프 고정 |
| 실행 OS | Windows 우선, macOS/Linux 호환 | 한컴오피스 미설치에서도 파일 생성 가능해야 |
| 프로젝트 성격 | 신규 프로젝트 — 재사용할 DB 스키마 없음 | Prisma 엔티티 대신 **CLI/파이프라인 스케치**로 대체 |

**명시적 비-결정**: 실행 모드(CLI vs 웹 vs Desktop GUI), 배포 경로, LLM 프로바이더 고정 — phase3 인터뷰에서 확정.

---

## 1. 실행 모드 옵션 (미결 — phase3 결정)

신규 프로젝트이므로 "유저가 무엇을 띄우고 어떻게 조작하는가"가 모든 구조 결정의 선행 변수다. 세 옵션을 병렬 설계 후 공통 코어를 뽑는다.

### Option A — CLI (v1 1순위 추천)
- **명령**: `gongmun attach --main main.hwpx --type roster --data students.csv --out ./out/`
- **장점**: ① 코어 파이프라인만 있으면 즉시 실행 ② 태블릿/Windows 노트북 어디서든 Python 설치만으로 작동 ③ 개인정보 로컬 고정이 구조적으로 보장(업로드 UI 자체가 없음) ④ 교사 1인 개인 업무 스코프에 최소 복잡도
- **단점**: ① 비-개발자 교사에게 허들 ② 다중 붙임 배치 처리는 스크립트 작성 필요
- **v1 적합**: ○ — 코어 파이프라인 = CLI로 선작성, 후속 UI는 이 코어 위에 얹음

### Option B — 로컬 웹 앱 (Next.js + Python API)
- **구성**: Next.js(UI) + FastAPI/Flask(hwpx 처리) / 또는 Electron로 감싸기
- **장점**: ① 교사 친화적 드래그·드롭 업로드 UI ② 붙임 유형 선택 UX 시각화 ③ 공유 학교 서버 배포 확장 가능
- **단점**: ① 초기 인프라 비용(Python 서버 + Next.js 두 런타임) ② 업로드 → 서버로 파일 전송 과정에서 "로컬 고정" 설계가 깨질 위험 ③ 포트·방화벽·인증 고민 추가
- **v1 적합**: △ — **v2 후보**. v1에는 과잉

### Option C — Desktop GUI (Tauri/PyWebView)
- **구성**: Python 코어 + 경량 웹뷰 (Tauri + Python sidecar, 또는 PyWebView)
- **장점**: ① 로컬 고정이 구조적으로 자연 ② 설치 파일 1개 배포 ③ 교사 친화
- **단점**: ① 빌드·서명·업데이트 채널 운영 필요 ② 크로스 플랫폼 빌드 복잡도
- **v1 적합**: △ — **v2 후보**

### 권고
**v1 = CLI + 얇은 스크립트 + JSON 설정 파일**. 코어 파이프라인을 CLI로 결정 짓고, UI는 phase3 인터뷰 이후 결정.
- 코어 모듈(`gongmun.core`)은 실행 모드 중립적 — 나중에 어떤 GUI로도 얹을 수 있게 설계
- CLI는 `typer` 권장 (자동 help·타입 검증)

---

## 2. 입력·출력 계약

### 2-1. 입력
| 필드 | 형식 | 필수 | 설명 |
|-----|------|----|------|
| `main_document` | hwpx 파일 경로 OR .txt 파일 경로 OR stdin 문자열 | 필수 | 주 공문 본문. hwpx이면 자동 text 추출 |
| `attachment_type` | enum(8종) OR `auto` | 필수 | `auto` 지정 시 LLM이 분류 |
| `data` | CSV/JSON/없음 | 유형별 조건부 | 명단·점수 등 구조화 데이터 |
| `template_id` | str | 옵션 | 명시하지 않으면 유형·학교급에서 기본 선택 |
| `output_dir` | dir path | 옵션(기본 `./out/`) | 결과 hwpx 저장 |
| `profile` | yaml path | 옵션 | 기관명·결재라인·담당자 기본값 |

### 2-2. 출력
- **주 산출**: 붙임 hwpx 파일 N개 (1건 공문 → 1개 이상 붙임)
- **보조 산출**:
  - `manifest.json` — 생성된 붙임 목록·유형·템플릿·LLM 토큰 사용량
  - `validate_report.txt` — zip 무결성·mimetype ZIP_STORED·XML 유효성 체크 결과
- **종료 코드**: 0(성공) / 2(검증 실패 — hwpx 깨짐) / 3(LLM 실패) / 4(템플릿 없음) / 5(데이터 스키마 불일치)

---

## 3. 파이프라인 (의사코드)

```
def generate_attachment(main_doc, att_type, data, template_id, profile) -> Path:
    # ── Stage 1. 주 공문 파싱 (로컬, LLM 미경유)
    if main_doc.suffix == ".hwpx":
        main_text = hwpx_extract_text(main_doc)   # python-hwpx, header·section0 전체 텍스트
    else:
        main_text = main_doc.read_text(encoding="utf-8")

    # ── Stage 2. 맥락 분석 (Claude API, 본문=비식별)
    context = claude.analyze_main_context(main_text)
    # → {"title","purpose","legal_basis","target","deadline","tone":"공문 관용체"}

    # ── Stage 3. 붙임 유형 결정
    if att_type == "auto":
        att_type = claude.suggest_attachment_type(context)  # 유형 8종 중 선택
    # (교사가 직접 지정한 경우 이 단계 스킵)

    # ── Stage 4. 템플릿 선택 (로컬)
    template_path = catalog.resolve(att_type, profile.school_level, template_id)
    # templates/roster/elementary-default.hwpx 같은 구조

    # ── Stage 5. 표 구조 추출 + 데이터 채움 (로컬 결정론)
    tpl = hwpx.open(template_path)                # python-hwpx Low-level
    if data is not None:
        schema = extract_table_schema(tpl)        # 헤더 라벨 자동 탐지
        validate_data_against_schema(data, schema)
        fill_table_by_label(tpl, data)            # fill_by_path API
        # ※ 이 경로에 LLM 개입 금지 — 개인정보 외부 유출 0

    # ── Stage 6. 본문 텍스트 영역 생성 (Claude API, 비식별 컨텍스트만)
    narrative_slots = find_narrative_placeholders(tpl)  # {{본문}}, {{목적}} 같은 토큰
    filled = claude.fill_narrative(context, slots=narrative_slots, tone="공문")
    apply_narrative(tpl, filled)                  # 결정론 치환 (LLM 결과는 텍스트로만 들어감)

    # ── Stage 7. linesegarray 제거 + 재패키징 (hwpx-master 스킬 규칙)
    strip_linesegarray(tpl.section0)              # <linesegarray> 전부 삭제
    out_path = hwpx.save(tpl, output_dir, atomic=True)
    # save 내부에서 mimetype 첫 엔트리·ZIP_STORED 보장 (python-hwpx atomic save)

    # ── Stage 8. 검증
    report = validate_hwpx(out_path)              # zip 무결성, XML well-formedness, mimetype 위치
    if not report.ok:
        raise HwpxValidationError(report)

    return out_path
```

**경계 규칙 (하드 룰)**
1. Stage 5에 LLM 금지 — 테스트에서 호출 mocking으로 검증
2. Stage 2·3·6에만 Claude API 허용
3. 모든 hwpx 편집은 `Contents/section0.xml`에 한정 (스킬 §3 준수)
4. 문자열 조작 금지 — `lxml.etree` API만

---

## 4. 붙임 유형 8종 데이터·필드 스펙 (v1 구현 우선순위 표시)

| # | 유형 | v1 우선 | 필수 데이터 | 템플릿 필드(텍스트 슬롯) | 채움 경로 |
|---|-----|---------|-----------|------------------------|----------|
| 1 | 명단표 | **P0** | CSV(번호·이름·학년반·연락처) | `{{사업명}}`,`{{기준일}}` | 표=로컬, 슬롯=LLM |
| 2 | 개별 결과통지서 | P2 | CSV(수신자) + 결과 | `{{수신}}`,`{{결과}}`,`{{안내}}` | 표=로컬, 본문=LLM |
| 3 | 평가표/심의안 | P1 | 평가항목·가중치·점수 | `{{평가대상}}`,`{{심의일}}`,`{{결론}}` | 표=로컬, 결론=LLM |
| 4 | 동의서/신청서 | **P0** | 사업 메타(기간·범위·수집항목) | `{{목적}}`,`{{수집항목}}`,`{{보유기간}}`,`{{서명란}}` | 전부 LLM + 개인정보 문구 관용구 검증 |
| 5 | 보고서/회의록 | P2 | 일시·장소·참석자·안건 | `{{안건}}`,`{{논의}}`,`{{결정사항}}` | 표=로컬, 서술=LLM |
| 6 | 품의서 붙임(내역서) | P1 | 항목·단가·수량 | `{{사업명}}`,`{{합계}}` | 표+합계=로컬 계산, 사업명=LLM |
| 7 | 가정통신문 | **P0** | (거의 없음 — 공문 본문에서 파생) | `{{머리글}}`,`{{본문}}`,`{{유의사항}}`,`{{회신란}}` | 전부 LLM (개인정보 부재) |
| 8 | 참석자 명부 | P1 | 날짜·명단 | `{{행사명}}`,`{{기간}}` | 표=로컬, 슬롯=LLM |

**v1 범위 = P0 3종** (명단표·동의서·가정통신문) — 표 구조 고정·서술 자유도 낮음 → LLM 환각·스킬 위반 리스크 최소.

---

## 5. 파일 구조 초안

```
gongmun-assistant/
├── README.md
├── pyproject.toml                      # poetry 권장
├── skills/
│   └── hwpx-master.SKILL.md            # 기존
├── src/gongmun/
│   ├── __init__.py
│   ├── cli.py                          # typer entry — `gongmun attach ...`
│   ├── config.py                       # profile.yaml 로더
│   ├── core/
│   │   ├── pipeline.py                 # Stage 1-8 오케스트레이션
│   │   ├── hwpx_io.py                  # python-hwpx 래퍼 + linesegarray 제거
│   │   ├── template_catalog.py         # 유형·학교급 → template 경로 resolver
│   │   ├── table_fill.py               # 결정론 표 채움 (LLM 금지 경계)
│   │   ├── narrative_fill.py           # LLM 기반 슬롯 채움
│   │   └── validate.py                 # zip·mimetype·xml 검증
│   ├── llm/
│   │   ├── client.py                   # Anthropic SDK 래퍼 (재시도·타임아웃)
│   │   ├── prompts/
│   │   │   ├── classify_attachment.md
│   │   │   ├── analyze_context.md
│   │   │   └── fill_narrative.md
│   │   └── privacy.py                  # 입력 텍스트 개인정보 스캐너(가드)
│   └── types/                          # pydantic 모델(Context, Profile, Schema)
├── templates/
│   ├── roster/                         # 유형 1
│   │   ├── elementary-default.hwpx
│   │   └── middle-default.hwpx
│   ├── consent/                        # 유형 4
│   ├── parent-letter/                  # 유형 7
│   └── catalog.json                    # 유형 → 파일 매핑
├── tests/
│   ├── fixtures/*.hwpx
│   ├── test_hwpx_io.py
│   ├── test_table_fill.py              # LLM mock — 호출 0회 보증
│   ├── test_pipeline_e2e.py
│   └── test_validate.py
└── scripts/
    └── validate_template.py            # 템플릿 추가 시 수동 실행
```

---

## 6. 프라이버시·보안 경계

| 데이터 | 허용 경로 | 금지 경로 |
|-------|--------|---------|
| 주 공문 본문(비식별) | Claude API 전송 | — |
| 학생·학부모 명단 CSV | 로컬 메모리·파일 시스템만 | Claude API 전송 **금지** |
| 결재자·담당자 이름(profile.yaml) | 본문 치환 시 필요 시 LLM에 전달 | 기본은 로컬 치환 |
| LLM 요청 로그 | 기본 off, `--debug`에서만 로컬 파일 | 외부 수집 **금지** |

**가드 레일**:
- `llm/privacy.py` — LLM 호출 직전에 프롬프트에 한국 주민번호/전화번호/이메일 패턴이 섞이면 예외 발생 (에러 코드 5)
- 단위 테스트 `test_table_fill.py` — Anthropic 클라이언트를 mock해 "호출 0회" 단언

---

## 7. 리스크 표

| # | 리스크 | 영향 | 확률 | 완화 |
|---|-------|-----|-----|------|
| R1 | python-hwpx Non-Commercial 라이선스 — 교육청 대량 배포 블로커 | 중 | 중 | (a) 교사 개인 용도만 v1 범위 명시 (b) 배포 전 airmang 저자 문의 (c) 라이선스 막히면 phase1 보조 경로(lxml 직접 구현)로 전환 |
| R2 | `<linesegarray>` 누락 시 한컴오피스 "보안설정 오류" | 높음 | 높음 | Stage 7 자동 제거 + Stage 8 검증. 테스트에 한컴오피스 실제 오픈 케이스 추가(수동 QA) |
| R3 | Claude API 장애 / 비용 폭증 | 중 | 중 | 재시도+백오프, 토큰 상한 설정, Haiku 폴백, 캐시(같은 본문은 해시 키로 메모이제이션) |
| R4 | LLM이 본문에 없는 사실 환각 | 높음 | 높음 | 프롬프트에 "본문 외 사실 추가 금지" 명시, Stage 6 결과를 원문 키워드 커버리지로 검사, v1은 P0 3종(자유서술 최소)으로 제한 |
| R5 | 개인정보 실수 유출 — 교사가 main_doc에 학생 이름 포함 | 높음 | 중 | privacy.py 패턴 스캐너, CLI 시작 시 경고 prompt, 동의 플래그 `--i-understand-no-pii` 강제 |
| R6 | 템플릿 catalog 관리 비용 | 중 | 높음 | catalog.json 스키마 고정, 템플릿 추가 시 `scripts/validate_template.py` 필수, v1은 3유형×2학교급=6개로 제한 |
| R7 | hwpx 재패키징 시 mimetype 순서·압축 실수 | 높음 | 중 | python-hwpx `atomic save` 사용, 직접 zip 다룰 시 스킬 §4 체크리스트 단위 테스트 |
| R8 | LLM 프롬프트에 결재자 실명 유입 | 중 | 중 | profile.yaml은 로컬 치환 기본, LLM 전달은 opt-in 설정 |
| R9 | 교사가 Python·CLI 사용 불가 | 중 | 높음 | v1 CLI 고정하되 설치 가이드·Windows용 .bat 래퍼 제공. v2에서 GUI 경로 |

---

## 8. 검증 체크리스트 (Done 기준)

- [ ] P0 3유형(명단·동의서·가정통신문) 각 1개 템플릿이 카탈로그에 등록
- [ ] Stage 1-8 파이프라인이 동일 main_doc로 재실행 시 bit-identical 결과(LLM temperature=0 기준)
- [ ] `test_table_fill.py`가 Anthropic 클라이언트 호출 0회 단언 통과
- [ ] 생성된 hwpx가 한컴오피스에서 "보안설정" 경고 없이 열림 (수동 QA)
- [ ] `validate_hwpx`가 zip 무결성·mimetype ZIP_STORED·첫 엔트리 위치·XML well-formedness 4항목 모두 OK
- [ ] `<linesegarray>` 요소가 section0.xml에 0개
- [ ] privacy.py가 주민번호 패턴 포함 프롬프트를 차단

---

## 9. 미결 질문 (phase3 인터뷰 재료, 8개)

1. **실행 모드** — v1을 CLI로 확정하는 데 동의하나? (GUI는 v2) 교사가 직접 터미널 사용 가능한가, 아니면 .bat 더블클릭 수준 래퍼가 필요한가?
2. **주 공문 입력 방식** — 파일 경로만 받을 것인가, 텍스트 붙여넣기(stdin/파일)도 받을 것인가? 두 경로를 동시 지원 시 검증 부담이 있음
3. **템플릿 관리 권한** — 교사가 자신만의 템플릿을 `templates/`에 추가할 수 있어야 하는가(사용자 경로), 시스템 제공 6개로 고정인가(v1 단순화)? 템플릿 라벨링 규약 누가 정하나?
4. **LLM 프로바이더 고정** — Claude API 전제가 맞나? 학교망 방화벽·개인 API 키 비용이 이슈면 Ollama 로컬 모델 경로를 v1부터 넣어야 하는가?
5. **개인정보 처리 정책** — LLM 호출 로그를 로컬 저장할 것인가? 명단 CSV는 처리 후 자동 삭제? 교육청의 학생 개인정보 처리 지침 구체 조항을 어디까지 맞춰야 하나?
6. **결재라인·기관 정보** — `profile.yaml`로 분리할 항목 범위(학교명·결재자·담당자·연락처)? LLM에 전달 가능 범위?
7. **배포 모드** — 개인 설치(pip install) vs 학교 공유 서버? 공유 서버라면 다중 교사 격리(파일·프로필)가 필수
8. **포지셔닝·차별화** — 나라에듀·한컴어시스턴트·기존 매크로와의 관계는 "대체"인가 "보완"인가? 교사가 이미 쓰는 양식·워크플로우 중 어느 지점에 이 도구가 끼어드나?

---

## 10. Phase 3 인터뷰 진입 전제

- sketch만으로 확정 가능한 항목은 이미 §0·§2·§3·§5에 고정
- 인터뷰는 §9 8개 질문 집중, 목표는 v1 스펙 freeze
- 인터뷰 이후 phase4(시드)에서 카탈로그·프롬프트 파일을 확정하고 phase5에서 PoC 구현
