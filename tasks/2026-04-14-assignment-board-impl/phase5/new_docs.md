# Phase 5 — New Docs

- **task_id**: `2026-04-14-assignment-board-impl`
- **seed**: `seed_38c34e91bf28`
- **실행일**: 2026-04-14

## 신규 생성 plan 문서

| # | 파일 | 목적 / 구조 |
|---|---|---|
| 1 | `plans/assignment-board-roadmap.md` | 과제 배부 보드 v1 살아있는 설계 문서. 12개 절: (0) 핵심 명제 — 북극성 5 primitive / (1) Board 확장 스키마 / (2) AssignmentSlot 엔티티 스키마 / (3) 상태 머신(전이도 + 재제출 매트릭스 + 히스토리 정책) / (4) UI 엔트리포인트 표 / (5) 학급 N 정책 / (6) 역할별 흐름(owner·editor·viewer) / (7) 태블릿 성능 체크리스트 / (8) 관련 로드맵 관계 (event-signup·tablet-performance·parent-viewer·implementation-roadmap/Canva) / (9) 작업 분할 AB-1 ~ AB-10 / (10) 수용 기준 13종 / (11) v2+ 파킹 / (12) 리스크 7종 + 변경 로그. |

## 신규 건수: **1**

## 신규 `data/*.json` 시드 데이터

없음. Seed 11은 엔티티 스키마 확장만 필요하며, 시스템 템플릿·사전 데이터 없음. (참고: Seed 4 식물 카탈로그나 Seed 10 mallang-ranch처럼 시드 데이터가 필요한 도메인 아님.)

## 신규 `research/*` 보고서

없음. exploration.md에서 수집한 6개 공개 레퍼런스(Classroom·Teams·Seesaw·Canvas·Moodle·Padlet + 트라이디스)는 task 산출물 내부에서 참조되며, plan 레벨 별도 리서치 보고서로 승격 불필요.

## `ideas-parking-lot.md` 추가 항목

없음. 본 시드의 v2+ 파킹 항목들(SubmissionHistory 승격 · 5×8 격자 · 갤러리 모드 · 풀 코멘트 · Roster 자동 동기화 · matrix 뷰)은 `plans/assignment-board-roadmap.md §11`에 집약되어 있어, `ideas-parking-lot.md` 상향 이관 불필요.
