# Destinations Registry (v2)

정교화된 아이디어의 **최종 배송지 레지스트리**. 오케스트레이터가 `dispatcher` 에이전트를 통해 이곳에 기재된 경로로 배송한다.

## 구조 변경 (v2, 2026-04-12)

- **외부 프로젝트 destination**은 실제 프로젝트 폴더 안의 `INBOX/`에 직접 배송
- **ideation 내부 destination**(parking·archive·research-vault)만 `ideation/destinations/` 아래 유지

## 외부 프로젝트 Destinations

| 목적지 | 경로 | 프로젝트 성격 | 라우팅 트리거 (키워드·topic) |
|---|---|---|---|
| **padlet** | `../padlet/INBOX/` | Aura-board 교실 학습 플랫폼 | "Aura-board", "그림보드", "식물관찰", "행사 신청", "라이브러리", "Canva 통합", "학생·교사·보드" |
| **aura** | `../aura/INBOX/` | AURA 대시보드 (Next.js, editorial 디자인) | "AURA 대시보드", "AuthKey", "editorial 카드", "aura-mobile", "aura-video" |
| **class** | `../class/INBOX/` | 교사 수업 기획·슬라이드·비디오 콘텐츠 | "수업 기획", "slides_request", "video_request", "lesson", "커리큘럼" |
| **library** | `../library/INBOX/` | Reading Creation Platform (독서·창작) | "독서 플랫폼", "reading creation", "governance contracts", "독서록·창작" |
| **trading-journal** | `../trading-journal/INBOX/` | 암호화폐 매매일지 웹앱 | "매매일지", "trading journal", "전략 승률", "거래 차트" |
| **퀀트기획서** | `../퀀트기획서/INBOX/` | 바이비트 자동매매 봇 + 대시보드 | "bybit 봇", "자동매매", "bot1/bot2", "전략 파라미터", "EC2 봇" |
| **free** | `../free/INBOX/` | 교육공공데이터 AI활용대회 출품작 묶음 | "공모전", "afterschool", "child-select", "safe-route", "culture-pass", "meal-safe" 등 서브 프로젝트명 |
| **gongmun-assistant** | `../gongmun-assistant/INBOX/` | 한국 공공기관·학교 공문서 작성 자동화 (hwpx-master 기반) | "공문", "기안문", "붙임", "hwpx", "공문서", "공공기관 문서", "행정 업무 문서" |

**외부 프로젝트 INBOX 특성**:
- 각 INBOX에 `README.md`가 있어 "어디서 왔는지·삭제해도 되는지" 자립 설명
- 각 프로젝트 팀(에이전트·사용자)이 소비 후 자유롭게 삭제 가능
- 외부 INBOX 쓰기는 **dispatcher 에이전트만** 수행 (다른 에이전트는 접근 금지)

## ideation 내부 Destinations

| 목적지 | 경로 | 용도 | 라우팅 트리거 |
|---|---|---|---|
| **parking** | `destinations/parking/` | 지금은 안 하지만 나중에 꺼낼 | `scope == "parking"` |
| **archive** | `destinations/archive/` | 종결·폐기·수퍼시드 | 사용자 명시 "아카이브" or refinement로 수퍼시드된 이전 시드 |
| **research-vault** | `destinations/research-vault/` | 완결된 리서치 보고서 | `scope == "research_only"` |
| **internal** ⭐ | `destinations/internal/` | **Fallback** — 외부 매칭 실패 시 로컬 보관 | 다른 모든 라우팅 실패 |

## 라우팅 로직 (dispatcher 판정)

### 1차 — scope 기준
| scope | 기본 destination |
|---|---|
| `parking` | parking |
| `research_only` | research-vault |
| `quick_decision` / `full_exploration` | → 2차로 |

### 2차 — topic 키워드 매칭
- topic에서 위 표의 키워드 매칭 → 해당 외부 destination
- 여러 destination 매칭 시 가장 구체적인 것 우선 (예: "Aura-board 식물관찰" → padlet)
- 매칭 실패 → 3차로

### 3차 — 사용자 확정
- AskUserQuestion으로 destination 리스트 제시 + "로컬 보관" + "다른 곳" 옵션
- "다른 곳" 선택 시 새 destination 등록 필요 (에스컬레이션)

### 4차 (Fallback) — internal 로컬 보관
- 사용자 응답 없음 또는 "로컬 보관" 선택
- topic이 기존 어느 외부 프로젝트와도 맞지 않음
- → `destinations/internal/{task_id}/` 에 외부 INBOX와 동일한 묶음 저장
- MANIFEST.md에 라우팅 실패 사유와 추정 대상 프로젝트 힌트 기재

### 특수 라우팅
- free 프로젝트는 **복수 서브 프로젝트**를 포함하므로 MANIFEST에 `target_subproject` 필드 필수 기재
- refinement로 수퍼시드된 이전 시드는 본 destination과 별개로 `archive`에도 동시 배송

## 배송 산출물 표준

### 외부 프로젝트 INBOX
```
{project}/INBOX/{YYYY-MM-DD-slug}/
├── MANIFEST.md            # 배송 메타 (topic·scope·seed_id·routing_reason·delivered_at·target_subproject?)
├── request.json           # 대상 프로젝트의 phase0 request 포맷
├── handoff_note.md        # 에이전트 프롬프트
├── seed.yaml              # 시드 YAML 전문
├── decisions.md           # phase3 결정 요약
└── context_links.md       # ideation 내 관련 문서 경로 (상대 경로)
```

### parking (경량)
```
destinations/parking/{YYYY-MM-DD-slug}.md   # 단일 마크다운
```

### archive (경량)
```
destinations/archive/{YYYY-MM-DD-slug}/
└── summary.md
```

### research-vault
```
destinations/research-vault/{YYYY-MM-DD-slug}/
└── report.md
```

## 확장 방법

새 외부 프로젝트를 destination으로 등록하려면:
1. `{project-folder}/INBOX/` 생성
2. `INBOX/README.md` 작성 (출처·구조·삭제 가능 명시 — 기존 README 템플릿 참조)
3. 본 레지스트리 "외부 프로젝트 Destinations" 표에 행 추가 + 라우팅 트리거 키워드 지정
4. `dispatcher.md` 라우팅 로직은 본 레지스트리를 읽으므로 에이전트 측 별도 변경 불필요

## 비활성화 방법

외부 프로젝트가 더 이상 배송 받기를 원하지 않으면:
1. 본 레지스트리의 해당 행에서 "라우팅 트리거" 칼럼을 비워둠
2. 해당 INBOX 폴더는 유지 (기존 배송은 소비 후 삭제)
3. dispatcher가 해당 destination을 비활성 처리

## 변경 이력

- **2026-04-12**: v2 — 외부 destination을 ideation 내부에서 외부 프로젝트 실제 폴더(`../{project}/INBOX/`)로 이전. 아우라·class·library·trading-journal·퀀트기획서·free 추가 등록.
