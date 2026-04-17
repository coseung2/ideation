# Phase 3 Decisions — gongmun-assistant v1

- **task_id**: `2026-04-12-gongmun-attachment-autofill`
- **session_id**: `interview_20260412_131559`
- **ambiguity_score**: `0.18` (목표 ≤ 0.20 달성)
- **진행 모드**: 에이전트 자율 답변 (사용자 부재)
- **결정 건수**: 8건 (sketch §9 미결 8개 전부 해소)

---

## Ouroboros 인터뷰 확정 (3라운드)

### D1. 실행 모드 — v1 = CLI 고정 [답변 라우팅: 에이전트]
- **결정**: v1 CLI 단일, Desktop GUI v2, 웹 v3+.
- **구현 세부**:
  - `typer` 기반 `gongmun attach ...` 명령. 인자 없이 실행 시 inquirer/typer prompt로 대화형 전환.
  - Windows용 `gongmun.bat` 래퍼 동봉 (탐색기 더블클릭 지원).
  - 코어 모듈 `gongmun.core`는 실행 모드 중립적 설계 → v2 GUI가 동일 코어 재사용.
- **근거**:
  1. 스코프 최소화 — 코어 파이프라인(Stage 1-8) 품질에 자원 집중.
  2. 개인정보 구조적 안전 — 업로드 UI 부재로 파일이 로컬 디스크 이탈 경로 자체가 없음.
  3. Python 3.11+ 설치만으로 Windows·macOS·Linux 작동, 방화벽·서명 이슈 없음.
  4. 대화형 프롬프트 + .bat 래퍼로 비개발자 교사 마찰 완화.

### D2. 주 공문 입력 방식 — 이중 경로, 단일 수렴 [답변 라우팅: 에이전트]
- **결정**: 주 경로 = 기존 hwpx 파싱(①), 보조 경로 = 유형 직접 선택(②). Stage 1에서 단일 `main_text: str`로 수렴.
- **구현 세부**:
  - `--main main.hwpx`: hwpx 파싱 → Claude context 추출 → 유형 자동/명시 선택.
  - `--text -` (stdin) 또는 `--text-file main.txt`: 텍스트 본문 경로.
  - `--prompt "..."` (main 없이): P0 3종 중 **가정통신문에만** 허용 (LLM 환각 리스크).
  - 대화형 기본값: hwpx 경로 → 비우면 유형 선택 → 간단 설명 입력 (②로 폴백).
- **근거**: 교사의 실제 업무 패턴은 "공문 기안 후 붙임 작성". 본문 맥락 일관성이 이 도구의 차별점. 입력 3종을 Stage 1에서 수렴하므로 테스트 면적 관리 가능.

### D3. 템플릿 관리 — 하이브리드 (번들 + 사용자 디렉토리 규약) [답변 라우팅: 에이전트]
- **결정**: 시스템 번들 필수 + 사용자 템플릿 "디렉토리 규약" 수준 지원. CRUD 명령은 v2.
- **구현 세부**:
  - **시스템 번들**: `templates/{type}/{school-level}-default.hwpx` (P0 3종 × 2 학교급 = 6개). `templates/catalog.json`에 매니페스트.
  - **사용자 템플릿**: `~/.gongmun/templates/user/{type}/{name}.hwpx` + 동일 경로에 `{name}.yaml` 사이드카(type, school_level, slots, schema).
  - **자동 병합**: CLI 부팅 시 사용자 디렉토리 스캔 → catalog 병합 (user-prefixed id: `user:my-consent-2026`).
  - **검증**: 로드 시 슬롯 존재·zip 무결성·mimetype 위치 3항목. 실패 시 경고 후 스킵(크래시 금지).
  - **라벨링 규약**: `docs/template-authoring.md`. 슬롯 토큰은 `{{snake_case}}`, 필수는 `*` 접두. 슬롯 이름 집합은 시스템 결정 (sketch §4).
- **v2로 미룸**: `gongmun template add/remove/validate`, 업로드 GUI, 템플릿 공유.

---

## 에이전트 자율 확정 (인터뷰 진입 질문 중 축약 재확정, 5건)

sketch §9의 나머지 5개 질문은 Ouroboros 인터뷰가 ambiguity 0.18에 3라운드 만에 도달해 자동 종료했으므로, 에이전트 라우팅 원칙(기존 결정 준수·일관 정책 적용)에 따라 에이전트 자율 답변으로 확정. 모두 "기존 패턴·보안 정책·sketch 권고" 범주로 사용자 확정 불필요.

### D4. LLM 프로바이더 — Claude API 기본 + Ollama 옵션 [답변 라우팅: 에이전트, 보안 정책 준수]
- **결정**: 기본 Claude API (Sonnet, Haiku 폴백). 설정에서 Ollama 로컬 모델로 전환 가능.
- **구현 세부**:
  - `~/.gongmun/config.yaml`의 `llm.provider: anthropic|ollama`. 기본 `anthropic`.
  - `llm/client.py`에 provider 추상 인터페이스 (analyze_context, suggest_attachment_type, fill_narrative). 두 구현체 동일 시그니처.
  - Ollama 사용 시 기본 모델 `qwen2.5:14b-instruct-q4`, 사용자 커스텀 허용.
  - 학교망 방화벽·API 키 비용 이슈 교사용 대체 경로 확보.
- **근거**: 본문은 비식별 정보만 전달되지만, 민감 기관 정책상 외부 API 금지 교사가 존재. v1부터 구조적으로 로컬 대안 확보해야 배포 경로 막히지 않음.

### D5. 개인정보 처리 정책 — 로컬 고정·마스킹 로그·자동 삭제 없음 [답변 라우팅: 에이전트, 기존 보안 정책 준수]
- **결정**:
  - 본문 생성: Claude/Ollama로 일반 맥락만 전달, `llm/privacy.py`가 호출 직전 한국 주민번호·전화번호·이메일·학번 패턴 스캔 → 매치 시 예외(exit code 5).
  - 명단·개인데이터 주입: 100% 로컬 결정론, Anthropic 클라이언트 호출 0회(`test_table_fill.py` mock 단언).
  - LLM 로그: 기본 off. `--debug` 시에만 로컬 파일 저장, 저장 전 프롬프트·응답에 동일 마스킹 파이프라인 적용.
  - 명단 CSV 자동 삭제: **하지 않음**. 교사가 동일 데이터로 재실행·검증하는 업무 패턴이 일반적이므로 파일 생명주기는 교사 결정. `.tmp/` 중간 산출물만 성공 시 자동 정리.
- **근거**: sketch §6 프라이버시 경계 준수. 교육청 "학생 개인정보 처리 지침" 조항 중 "외부 전송 금지·최소 수집·접근 로그" 3항목은 설계로 보장됨. "자동 삭제"는 지침 요건 아님(교사 재량).

### D6. 결재라인·기관 정보 범위 — `~/.gongmun/config.yaml` 로컬 고정 [답변 라우팅: 에이전트, 기존 결정 준수]
- **결정**: `~/.gongmun/config.yaml`에 개인 설정 저장. CLI 첫 실행 시 셋업 마법사.
- **필드 범위**:
  ```yaml
  school:
    name: "○○초등학교"
    level: "elementary"  # elementary|middle|high
    department: "3학년부"
  officer:
    name: "홍길동"
    role: "담당교사"
    contact: "내선 1234"
  approval_chain:
    - {role: "담당", name: "홍길동"}
    - {role: "검토", name: "김부장"}
    - {role: "결재", name: "이교감"}
  llm:
    provider: "anthropic"
    model: "claude-sonnet-4-5"
  ```
- **LLM 전달 정책**: 기본 **로컬 치환 only**. LLM에 결재자 실명 전달 금지. 예외적 전달은 opt-in 플래그 `--share-officer-to-llm` (문서에 리스크 명시).
- **근거**: sketch §6 R8 완화. 결재자 실명은 LLM 프롬프트 개입 불필요 — 템플릿의 결재란은 로컬 치환으로 충분.

### D7. 배포 모드 — 개인 설치 (pip/pipx) [답변 라우팅: 에이전트, 스코프 정책 준수]
- **결정**: v1은 `pip install gongmun-assistant` 또는 `pipx install gongmun-assistant`. 학교 공유 서버는 v3+.
- **구현 세부**:
  - PyPI 배포, Python 3.11+ 요구.
  - 첫 실행 시 `~/.gongmun/` 초기화 및 config 셋업 마법사.
  - Windows 교사용 설치 가이드: `pipx` 권장 (전역 오염 없음) + `gongmun.bat` 자동 PATH 등록.
- **명시적 비포함**: 다중 교사 격리, 학교 공유 서버 인증, 파일·프로필 분리. R1(python-hwpx NC 라이선스) 배포 블로커 회피도 개인 용도 명시로 해결.

### D8. 포지셔닝·차별화 — "붙임파일 특화 + LLM 맥락 파악 + 로컬 실행" [답변 라우팅: 에이전트, 스코프 명시]
- **결정**: 나라에듀·한컴어시스턴트와는 **보완** 관계. 공문 본문 생성은 범용 AI 도우미·한컴어시스턴트 영역이므로 v1 제외.
- **차별점 3축**:
  1. **붙임파일 특화** — 본문 기안 후 "붙임 N종을 일관되게 생성"하는 구간은 어느 도구도 커버 안 함.
  2. **LLM 맥락 파악** — 본문 파싱 → 붙임 자동 분류·슬롯 채움. 한컴어시스턴트의 서식 변환과는 계층이 다름.
  3. **로컬 실행·개인정보 분리** — 명단·점수·학부모 연락처는 외부 API에 도달하지 않는 구조 보장.
- **교사 워크플로우 진입점**: 공문이 나라에듀에서 기안·결재 완료된 직후 "붙임 첨부" 직전 단계. 도구가 생성한 hwpx를 교사가 육안 검수 후 나라에듀에 업로드.

---

## 실패 처리 (하드 룰 재확정)

- hwpx 생성 실패 시: XML·zip 오류 원인을 `validate_report.txt`에 기록, 원본 템플릿은 불변(atomic save).
- 중간 산출물 `.tmp/`: 실패 시 유지(디버깅용), 성공 시 자동 정리.
- 종료 코드: 0=성공, 2=hwpx 검증 실패, 3=LLM 실패, 4=템플릿 없음, 5=개인정보 패턴 감지.

---

## 새로 드러난 분기 (현재 세션 편입 금지 — v2+ 파킹)

- **P1**: P1 3종(평가표·품의서 내역서·참석자 명부) — 표 구조 고정되어 있고 slot 자유도 중간. v2 주 범위.
- **P2**: P2 2종(개별 결과통지서·보고서/회의록) — 서술 자유도 높아 LLM 환각 리스크 관리 필요.
- **GUI 고려**: Tauri + Python sidecar 또는 PyWebView. 코어 `gongmun.core`가 실행 모드 중립이므로 UI 레이어만 추가.
- **학교 서버 배포**: v3+. 다중 교사 격리, SSO(EDU-ID), 템플릿 마켓플레이스.
- **템플릿 CRUD 명령**: v2. `gongmun template add/remove/validate`.
- **슬롯 자동 탐지**: v2. 현재 사용자가 YAML에 수기 선언.
- **한컴어시스턴트 연동**: 추후 검토. 본문 생성 영역 중복 방지 필요.

---

## Phase 4 진입 체크리스트

- [x] ambiguity ≤ 0.2 달성 (0.18)
- [x] D1-D8 모든 결정에 근거 기록
- [x] 에이전트 자율 답변 범위가 라우팅 원칙 준수
- [x] 새로 드러난 분기는 파킹, 현재 세션 미편입
- [x] session_id.txt 저장
- [x] Ouroboros 리턴: "Ready for Seed generation"

Phase 4 seed generation 진입 준비 완료.
