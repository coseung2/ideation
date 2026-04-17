# Phase 1 Exploration: 공문 붙임파일 자동 생성 도구 (gongmun-attachment-autofill)

- **task_id**: `2026-04-12-gongmun-attachment-autofill`
- **topic**: 공문 본문 맥락을 해석해 붙임용 hwpx 파일을 자동 생성하는 gongmun-assistant v1
- **작성일**: 2026-04-12
- **기반 스킬**: `gongmun-assistant/skills/hwpx-master.SKILL.md`

---

## 1. 조사 범위

1. 한국 공문/기안문 자동화 기존 도구 (한컴·교육 현장·민간 스타트업)
2. hwpx 포맷 처리용 Python 라이브러리 및 XML 베스트 프랙티스
3. LLM 배포 옵션 (로컬 vs 클라우드) — 개인정보 포함 가능성 고려
4. 교사 업무 공문 붙임 유형 카탈로그 (≥ 6종)

---

## 2. 유사/관련 도구 비교 (후보 ≥ 3)

| # | 도구 | 라이선스 | 가격 | 배포 형태 | 태블릿·로컬 친화 | gongmun-assistant 적합성 |
|---|------|---------|------|----------|----------------|-------------------------|
| A | **python-hwpx** (airmang) | Custom Non-Commercial | 무료(비상업) | 파이썬 라이브러리, lxml ≥ 4.9 | 로컬 친화 O (한컴 불필요, cross-platform) | **매우 높음** — section0.xml 편집, 표 자동 채우기(fill_by_path), Low-level OWPML 접근 제공. hwpx-master 스킬 규칙(lxml만 사용)과 정합 |
| B | **pyhwpx** (martiniifun) | MIT (wrapper) / 한컴 API 종속 | 무료(한컴 필요) | pywin32 자동화 래퍼 | 윈도+한컴오피스 필수 → 교사 로컬 OK, 서버 배포 불가 | 중간 — 한컴오피스 GUI 자동화로 안정적이지만, v1 목표(파일 단독 생성)와 상충. 한컴 미설치 환경 불가 |
| C | **한컴어시스턴트** (Hancom) | 상용 SaaS | 구독형 | 클라우드 + 한컴오피스 플러그인 | 윈도 전용, 클라우드 의존 | 낮음 — 윤문·요약 중심, 붙임 파일 타입 자동 분기·양식 채움 기능 미제공. 개인정보 클라우드 전송 이슈 |
| D | **Inline AI** (inline-ai.com) | 상용 SaaS | 구독형 | 한글 문서 위 AI 비서 | 윈도 전용 | 낮음 — 본문 윤문용, 붙임 타입별 양식 생성 X |
| E | **hwpx-mcp-server** (airmang) | MIT 계열(GH 공개) | 무료 | MCP 서버 (Claude Desktop 연동) | 로컬 실행 가능 | 높음(부차) — python-hwpx 위에 MCP 얹은 형태. LLM 연동 레퍼런스로 활용 가능 |
| F | **K-에듀파인 표준서식** | 공공 무료 | 무료 | 웹 포털 수동 다운로드 | 교사 필수 사용 | 소스 데이터 — 붙임 양식 seed로 확보해야 함 |

### 후보별 장단점

**A. python-hwpx**
- 장점: ① hwpx-master 스킬이 전제하는 lxml·section0.xml 편집 모델과 정확히 일치 ② 표 라벨 기반 자동 채움 API(`fill_by_path`, `find_cell_by_label`) 제공 ③ 한컴오피스 미설치 환경에서도 재패키징 가능 (mimetype ZIP_STORED 처리 포함)
- 단점: ① Non-Commercial 라이선스 — 교사 개인 사용은 가능하나 교육청 배포 시 재확인 필요 ② 도형·이미지 삽입은 미완성 ③ 문서 복구(validate.py) 후처리 필수
- gongmun 적합: `<linesegarray>` 제거·mimetype 비압축 규칙을 내장 헬퍼로 우회 가능 → v1 PoC 백본으로 직결

**B. pyhwpx**
- 장점: ① 한컴 공식 API 래핑 → 렌더링 정확도 최고 ② 표·개체 조작 API 풍부
- 단점: ① 윈도+한컴오피스 종속 → 서버화 불가 ② GUI 자동화라 배치 처리 느림 ③ 스킬 규칙(XML 직접 편집)과 접근법 상충

**C. 한컴어시스턴트 / D. Inline AI**
- 장점: 본문 윤문은 강력
- 단점: ① 붙임 타입 분기·양식 자동 채움 기능 없음 ② 개인정보 클라우드 전송 ③ 비용·종속성

**E. hwpx-mcp-server**
- 장점: LLM↔hwpx 편집의 MCP 레퍼런스 아키텍처 — 2단계(본문 이해→붙임 채움) 워크플로우 설계 시 참고
- 단점: Claude Desktop 종속, 교사 범용 배포는 어려움

---

## 3. 공문 붙임 유형 카탈로그 (교사 업무 기준, ≥ 6종)

| 유형 | 표준 구성 요소 | 자동화 난이도 | 데이터 입력원 |
|------|-------------|-------------|-------------|
| 1. **명단표** (강사·학생·학부모) | 표 헤더(번호/이름/소속/연락처), 반복 행, 합계 | 낮음 (표 반복 생성) | CSV·나이스 추출 |
| 2. **개별 결과통지서** | 수신자별 1부, 제목·수신·본문·결과·안내·발송일 | 중간 (mail-merge 패턴) | 명단 + 결과 테이블 |
| 3. **평가표 / 심의안** | 평가 항목 행, 가중치·점수 열, 서명란 | 중간 (표 구조 고정, 값만 채움) | 평가 루브릭 |
| 4. **동의서 / 신청서** | 개인정보 수집 고지, 서명·날짜, 체크박스 | 중간 (개인정보 동의문구 관용표현 필수) | 사업 메타데이터 |
| 5. **보고서 / 회의록** | 일시·장소·참석자·안건·논의·결정사항 | 높음 (자유 서술 비중↑) | 회의 메모·원본 |
| 6. **결재 품의서 붙임** (내역서·예산서) | 사업명·예산 항목·단가·수량·합계 | 중간 (표 + 합계 계산) | 품의 본문 |
| 7. **안내문 / 가정통신문** | 머리글·본문·유의사항·회신란 | 낮음 (템플릿 치환) | 본문 요약 |
| 8. **참석자 명부 / 출석부** | 날짜 × 이름 매트릭스 | 낮음 | 명단 |

→ **v1 우선순위**: 1(명단표) → 4(동의서) → 7(가정통신문). 이유: 표 구조 고정·서술 자유도 낮음 → LLM 환각 리스크↓, hwpx-master 스킬 규칙으로 안전 편집 가능.

---

## 4. LLM 배포 옵션 비교

| 축 | 로컬 LLM (Gemma 3, Llama 3.1-8B, Qwen 2.5) | 클라우드 LLM (Claude/GPT/Gemini API) |
|---|---|---|
| 개인정보 | 유출 0 (로컬 RAM) | 제3자 서버 전송 — 교육청 정보보호 지침 저촉 가능 |
| 한국어 공문 품질 | Qwen 2.5·EXAONE은 우수 / 7-8B급 공문 관용표현은 수준↓ | Claude Sonnet/Opus 경어체·관용표현 우수 |
| 비용 | 0원 (하드웨어 감가상각만) | 월 수천~수만 원 (건당 과금) |
| 인프라 | Ollama/LM Studio 12-16GB RAM 필요 | API 키만 있으면 즉시 |
| 교사 배포 | 설치 허들 높음 | 즉시 사용 |
| 민감도 매칭 | 학생 이름·연락처 포함 시 필수 | 비식별 본문만 처리해야 |
| v1 적합 | △ (품질 검증 필요) | ○ (PoC 속도) |

### 권고 하이브리드
- **본문 분석(붙임 타입 분류·필드 추출)**: 클라우드 LLM (Claude Sonnet) — 본문에는 개인정보 없음
- **개인정보 포함 데이터 주입(명단 CSV → 표 셀)**: 로컬 결정론 Python 로직 (LLM 미경유)
- → "민감도로 분리하는 2단 파이프라인"이 교사 배포 현실해

---

## 5. 1순위 권고 + 근거

### 권고: **python-hwpx + Claude API(본문 분석 전용) + 결정론적 표 채움**

**근거**
1. **스킬 정합성**: hwpx-master 스킬이 요구하는 lxml 전용·section0.xml만 수정·`<linesegarray>` 삭제 규칙은 python-hwpx의 Low-level OWPML API와 1:1 매칭. 스킬 규칙 위반 리스크 최소
2. **교사 로컬 현실**: Windows + 한컴오피스 환경에서 한컴 GUI 없이도 파일 생성 가능 (pyhwpx 대비 장점). PoC를 서버·CLI·IDE 어디서도 재현 가능
3. **개인정보 분리 설계**: LLM은 공문 본문(비식별)만 처리해 붙임 타입·필드 스키마 산출 → 명단·결과 데이터는 로컬 CSV에서 결정론적 주입. 교육청 보안 지침(학생 개인정보 외부 전송 금지) 준수
4. **재패키징 안정성**: python-hwpx의 atomic save·mimetype ZIP_STORED 내장 처리가 한컴오피스 보안설정 오류(알려진 이슈)를 회피
5. **유형 커버리지**: 표 `fill_by_path` API로 v1 우선 3종(명단·동의서·가정통신문) 전부 커버
6. **라이선스 주의점**: python-hwpx의 Non-Commercial 조항 — 교사 개인 업무 사용은 허용 범위, 교육청 대량 배포 전 상업 라이선스 또는 airmang 저자 문의 필요(**이슈 이관 필수**)

### 보조 선택지 (리스크 회피용)
- Non-Commercial 조항이 배포 블로커가 되면: 직접 lxml로 hwpx-master 스킬 예제만 사용하여 경량 구현 (의존 최소, 라이선스 자유) → Phase 2 sketch에서 fallback 경로로 설계

---

## 6. 참조 링크 (≥ 2)

- [airmang/python-hwpx — GitHub](https://github.com/airmang/python-hwpx) (v2.9.0, 2026-04-02) — 1순위 라이브러리, lxml 기반 pure-Python hwpx 편집
- [한컴테크: Python을 통한 HWPX 포맷 파싱 (1)](https://tech.hancom.com/python-hwpx-parsing-1/) — 공식 OWPML 파싱 가이드 (스킬 규칙 근거)
- [GitHub: hancom-io/hwpx-owpml-model](https://github.com/hancom-io/hwpx-owpml-model) — OWPML 필터 모델 공식 레포
- [airmang/hwpx-mcp-server](https://github.com/airmang/hwpx-mcp-server) — LLM↔hwpx MCP 연동 레퍼런스
- [martiniifun/pyhwpx](https://github.com/martiniifun/pyhwpx) — 대안 후보 (한컴 GUI 자동화)
- [경기교육 공문서 작성법 (한 곳에 정리한)](https://www.goe.go.kr/resource/old/BBSMSTR_000000000028/BBS_202410150153084250.pdf) — 공문 관용표현·붙임 형식 공식 가이드
- [한컴 개발자 포럼: Hwpx section0.xml 수정 시 보안설정 오류](https://forum.developer.hancom.com/t/hwpx-section0-xml/2414) — `<linesegarray>` 제거 근거

---

## 7. Phase 2 sketch 제안

- **Sketch 1**: python-hwpx + Claude API — 민감도 2단 파이프라인 (1순위)
- **Sketch 2**: lxml 직접 구현 + 로컬 LLM(Ollama Qwen 2.5-7B) — 라이선스·오프라인 강경 모드
- **Sketch 3**: pyhwpx + 한컴오피스 GUI — 교사 로컬 전용, 렌더링 정확도 최우선 (fallback)

공통 결정 필요 항목: ①붙임 타입 분류기 프롬프트 설계 ②양식 템플릿 저장소 구조(유형×학교급) ③ `<linesegarray>` 제거 자동화 위치(라이브러리 vs 도구 계층).
