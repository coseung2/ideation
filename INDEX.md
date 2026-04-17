# ideation — 산출물 인덱스

상위 **Aura-board**(`padlet/`, 교실 학습 플랫폼)에 대한 Canva Developer 통합 + 태블릿 성능 + 식물관찰일지 + 그림보드+라이브러리 + 행사 신청 + 수익 tier 등의 기획 저장소.

**파이프라인 하네스**가 구축되어 있어 아이디어 → 인터뷰 → 시드 → 인수인계까지 자동화. 실제 구현은 `padlet/` feature 파이프라인에 phase0 request로 진입.

## 📁 구조

```
ideation/
├── INDEX.md                      # 이 파일
├── research/
│   ├── canva-developer-aura-research.md     # Canva Developer 전수 조사 + 갭 분석
│   └── notebook-generation-from-textbook-research.md  # 교과서→공책정리 자동생성 리서치 (2026-04-15)
├── plans/
│   ├── implementation-roadmap.md             # Canva 통합 6항목 (P0-①/② → P2-⑥)
│   ├── tablet-performance-roadmap.md         # 태블릿 성능 + T0-①~④ + 수용 게이트
│   ├── plant-catalog.md                      # 식물 10종 × 단계 × 관찰포인트
│   ├── plant-journal-roadmap.md              # 식물관찰일지 기능 로드맵 (PJ-1~7)
│   ├── textbook-notebook-roadmap.md          # 교과서→공책정리 PDF 파이프라인 (v1 동작중, 2026-04-15)
│   └── phase0-requests.md                    # padlet feature 파이프라인 진입 템플릿 모음
├── data/
│   └── plant-species-seed.json               # Prisma seed 용 10종 구조화 데이터
├── notebook-workflow/                        # 공책정리 워크플로우 (이식 가능 단위)
│   ├── README.md                             # 사용법·이식 가이드·명령어
│   ├── template.html                         # 재사용 CSS+구조 스켈레톤
│   └── examples/social-5-1-unit1.html        # 사회 5-1 1단원 완성 예시
└── canva-assignment-pdf-merge/               # 기존 동작 중인 Canva 스킬
    └── SKILL.md                              # 과제 완료본 PDF 병합 스킬
```

## 🧭 읽는 순서

처음 진입 시:
1. **`CLAUDE.md`** — 하네스 오케스트레이션 원칙
2. **`plans/seeds-index.md`** — 전체 시드·결정 인덱스
3. **`research/canva-developer-aura-research.md`** — Canva 통합 리서치
4. **`plans/tablet-performance-roadmap.md`** — 모든 기능의 상위 제약

새 아이디어 진입 시:
- **`prompts/ideation/_index.md`** — 파이프라인 실행 순서

재검토 진입 시:
- **`prompts/refinement/_index.md`** — 기존 결정 재평가 절차

구현 착수(padlet) 시:
- **`plans/phase0-requests.md`** — 작업 블록 복사
- **`tasks/{task_id}/phase6/handoff_note.md`** — 에이전트에게 줄 프롬프트

## 🔗 상호 참조 규칙

각 플랜 문서는 서로를 인용함:
- 태블릿 로드맵 → Canva 로드맵 각 항목의 태블릿 영향 평가
- 식물관찰일지 로드맵 → 태블릿 로드맵(성능 예산) + Canva 로드맵(P0-① oEmbed, P0-② Publisher)
- phase0 요청들 → 위 모든 문서의 앵커 링크

## 📌 주요 결정 (고정)

| 항목 | 결정 |
|---|---|
| 태블릿 성능 | 1차 필터. iPad 9th 기준 TTI < 3s, 드래그 60fps, iframe 동시 ≤ 3 |
| 매트릭스 뷰 접근 | owner + 데스크톱 전용 (editor·viewer·태블릿 전부 제외) |
| 식물 선택 권한 | 교사 제안형(허용 리스트) |
| 학생 식물 수 | 1인 1식물 |
| 단계 판정 | 학생 자기 선언 |
| 단계 건너뛰기 | 허용 |
| 사진 첨부 | 권장 (미첨부 시 사유 입력) |
| 예상 기간 | 범위(min~max) |
| 참고 이미지 | 초기 공백, 교사가 Canva URL로 채움 |
| P1-⑦ Teamspace 동기화 | **폐기** (태블릿 성능 위험) |

## 🚫 건드리지 않는 것

- 상위 폴더 `padlet/` (Aura-board) — **읽기 전용**. 실구현은 padlet의 feature 파이프라인에서만.
- 인증(OAuth/Canva)·저장소(Blob)·실시간 엔진 등 미결 사항은 각 로드맵에 명시.

## 📅 작성 이력

- 2026-04-12: 초판 (리서치·Canva 로드맵·태블릿 로드맵·식물 카탈로그·식물관찰일지 로드맵·seed·phase0 템플릿)
