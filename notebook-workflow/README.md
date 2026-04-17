# 교과서 → 학생 공책정리 PDF 생성 워크플로우

**목적**: 초등 교과서 PDF를 입력받아 "학생이 직접 필기한 것 같은" 스타일의 공책정리본 PDF를 자동·반자동 생성.

**아키텍처**: 오케스트레이터(메인 Claude) + 7개 전문 서브에이전트 + 교사 검증 게이트 1개.
**상태**: v1 동작 확인 (사회 5-1 1단원 샘플). v2(LLM 자동화) 설계 진행중.
**환경 설치**: [`SETUP.md`](SETUP.md) 참조 (Python/시스템 의존성·폰트·환경변수).

---

## 파일 구조

```
notebook-workflow/
├── README.md                 # 이 파일 — 진입점·이식 가이드
├── SETUP.md                  # 환경 설치 (Python·apt·폰트·환경변수)
├── template.html             # 재사용 HTML/CSS 스켈레톤 (이미지 슬롯 포함)
│
├── agents/                   # 각 phase 전문 에이전트 계약서
│   ├── _registry.md          # 카탈로그 + phase 매핑
│   ├── intake-analyst.md     # phase 0
│   ├── textbook-extractor.md # phase 1
│   ├── content-curator.md    # phase 2
│   ├── image-scout.md        # phase 4
│   ├── html-builder.md       # phase 5
│   ├── pdf-renderer.md       # phase 6
│   └── dispatcher.md         # phase 7
│
├── prompts/                  # phase 실행 사양
│   ├── _index.md
│   └── phase{0..7}_*.md
│
├── tasks/                    # 실행 이력 (감사용)
│   └── {YYYY-MM-DD-slug}/
│       ├── phase0/…
│       └── …
│
└── examples/
    └── social-5-1-unit1.html  # 참고 예시 (v1 수동 작성본)
```

## 파이프라인

```
[0] intake ──→ [1] extract ──→ [2] draft ──→ ⏸ [3] validate (교사) ──→
    [4] image-source ──→ [5] render ──→ [6] pdf ──→ [7] deliver
```

| Phase | 담당 | 주력 도구 | 핵심 산출 |
|---|---|---|---|
| 0 intake | intake-analyst | **PyMuPDF** | `request.json` (메타·TOC·sha256) |
| 1 extract | textbook-extractor | **PyMuPDF + Claude Vision** | 텍스트·이미지·bbox `extract.json` |
| 2 draft | content-curator | **Claude Tool Use (JSON schema)** | 체크박스형 `notebook_draft.md` |
| **3 validate** | **교사 수동** | **Obsidian + Tasks 플러그인** | `approved.md` (status: approved) |
| 4 image-source | image-scout | **Pillow + Wikimedia API + exiftool** | CC/공공 이미지 + `attribution.md` |
| 5 render | html-builder | **Jinja2 + mistune** | 완성 HTML (이미지 base64 임베드) |
| 6 pdf | pdf-renderer | **Playwright (Chromium)** | A4 PDF + 검증 |
| 7 deliver | dispatcher | **PowerShell interop + sha256sum** | OneDrive 배송 + 알림 |

상세는 [`prompts/_index.md`](prompts/_index.md) 참조.

## 오케스트레이터 프로토콜

메인 세션은 **직접 편집·서치·렌더를 하지 않는다**. 각 phase를 전담 에이전트에 `Agent` 도구로 위임한다.

```
Agent({
  description: "Phase N — {agent-name}",
  subagent_type: "general-purpose",
  prompt: "<agents/{agent-name}.md 전문>\n\n---\n런타임 입력:\n- task_id: ...\n- 입력 파일: ...\n- 추가 맥락: ..."
})
```

### 오케스트레이터가 직접 하는 일
- `tasks/{task_id}/phase{N}/` 폴더 생성
- 검증 게이트 판정
- phase 간 파일 경로 전달
- 재시도 결정 (최대 3회)
- 사용자 커뮤니케이션 (draft.md 검증 요청, 이미지 선택 에스컬레이션)

### 오케스트레이터가 하지 않는 일
- 직접 pdftotext·pdfimages·chromium 실행
- 직접 HTML 편집
- 직접 이미지 서치
- 에이전트 산출물 수정 (재호출로 해결)

## 이미지 정책

**허용 (우선순위순)**
1. 교과서 삽화 (학교 교육 목적 이용, 저작권법 제25조)
2. 정부·공공기관 공개 자료 (국토지리정보원 · 외교부 독도 · 문화재청 등)
3. Wikimedia Commons CC
4. Unsplash / Pexels

**금지**
- Google 이미지 검색 직접 다운 (저작권 불명)
- 타 교사 학습지·블로그 이미지 무단 사용
- 생성형 AI 이미지 (사회과 지리·역사 정확성 불안정)
- **AI 손그림 SVG / 인라인 도안** (교사 피드백: 부적절)

## 산출물 경로 관례

| 단계 | 위치 |
|---|---|
| 실행 이력 | `notebook-workflow/tasks/{task_id}/` |
| Chromium 렌더 임시 | `/mnt/c/temp_claude/` (WSL snap 제약 회피) |
| 최종 PDF 배송 | `OneDrive - 남선초등학교\학습자료\{과목}\공책정리\` |

## 이식 체크리스트

- [ ] `notebook-workflow/` 전체 복사
- [ ] Chromium 또는 Edge headless 설치
- [ ] WSL 환경: snap chromium은 `/tmp` 미지원 → `/mnt/c/` 쓰기
- [ ] Google Fonts CDN 접근 가능 (없으면 `fonts/` 폴더에 TTF + `@font-face`)
- [ ] OneDrive 동기화 실행
- [ ] 배송 경로의 과목·학년 폴더 존재 확인
- [ ] 각 에이전트의 `허용 도구` 목록이 실행 환경에서 가능한지 확인

## 스타일 가이드 (template.html 준수)

**공책다움의 4대 요소**
1. **배경**: 모눈·세로 마진선·가로 라인
2. **폰트**: 손글씨체 (Gaegu, Nanum Pen Script)
3. **레이아웃**: 비대칭·콜라주
4. **장식**: 마스킹테이프·스탬프·말풍선·하이라이트

**피해야 할 패턴**
- 교과서 원문 복붙 (학생 공책은 선택적·축약적)
- 단색 배경 + 기본 산세리프 폰트
- 모든 블록 같은 크기/정렬
- **AI 손그림 삽화** (실제 이미지로 대체)

## 로그

- 2026-04-15: v1 완성. 사회 5-1 1단원 샘플 생성. 교사 승인.
- 2026-04-15: 교사 피드백 반영 — AI 손그림 SVG 부적절, 실제 이미지 소싱 정책 확립.
- 2026-04-15: 에이전트 기반 파이프라인 설계 (7 에이전트 + 1 검증 게이트).
- (로드맵: `../plans/textbook-notebook-roadmap.md`)
- (리서치: `../research/notebook-generation-from-textbook-research.md`)
