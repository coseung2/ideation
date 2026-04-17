# Phase 2 — Sketch

데이터 모델·흐름·리스크 초벌 스케치. 미결 항목 목록 추출.

## 입력

- `tasks/{task_id}/phase0/request.json`
- `tasks/{task_id}/phase1/exploration.md`

## 출력

`tasks/{task_id}/phase2/sketch.md`

```markdown
# Sketch — {topic}

## 전제 결정 (phase1 결과 반영)

- 도구 1순위: Drawpile (GPL-3.0, 자체 호스팅)
- 기존 인프라 재사용: OAuth·Export·Folder
- 기준 단말: 갤럭시 탭 S6 Lite

## 데이터 모델 초안

기존 스키마(`padlet/prisma/schema.prisma`) 참조하여 추가·수정 포인트 명시.

```prisma
model StudentAsset { ... }
model AssetAttachment { ... }
```

## 사용자 흐름

### 학생 창작 흐름
1. 그림보드 진입
2. ...

### 학생 재사용 흐름
...

### 교사 관리 흐름
...

## 태블릿 성능 체크

`tablet-performance-roadmap.md` 성능 예산 준수 여부:
- [ ] iframe ≤ 3
- [ ] 섹션 스코프 쿼리
- [ ] 델타 페이로드
- ...

## Canva/Aura 기존 기능 시너지

- [ ] P0-① oEmbed 카드와의 연결점
- [ ] tier 한도 고려 (Free/Pro)

## 알려진 리스크

| 리스크 | 영향 | 완화 |
|---|---|---|
| ... | ... | ... |

## 미결 질문 (phase3 인터뷰 재료)

1. ?
2. ?
3. ?
```

## 절차

1. phase1 권고안을 전제로 고정
2. padlet 기존 스키마·패턴을 **읽기 전용으로 참조**해 재사용 가능한 엔티티 식별
3. 신규·변경 엔티티 Prisma 초안 작성 (정식 스펙 아님, 방향 제시)
4. 사용자 흐름을 역할별(학생·교사·학부모)로 분리
5. 태블릿 성능 예산 체크리스트 적용
6. Canva 통합·tier·다른 보드 기능과의 시너지 점검
7. 리스크 표 작성
8. **미결 질문 목록** — phase3 인터뷰에서 답할 질문들 ≥ 1개

## 자율 진행 지침

- 이미 `plans/`에 관련 문서가 있으면 인용·차이점만 기술
- 미결 질문은 너무 많이 만들지 말고(3~7개 적정), 인터뷰 시간 늘리지 않도록 핵심만
- "에이전트가 바로 결정할 만한 사소한 것"은 sketch에서 바로 결정 — 인터뷰로 미루지 말 것

## 검증 게이트 (스케치 검증)

- 데이터 모델 초안 존재
- 사용자 흐름 ≥ 1개 (역할별 구분)
- 태블릿 성능 체크리스트 명시
- 미결 질문 ≥ 1개 (단, 0개면 phase3 스킵 허용)

미달 시 phase2 재실행.

## 스킵

- `미결 질문 == 0`: phase3 스킵, phase4에서 기존 결정만으로 시드 생성 (ambiguity 자동 계산 조건부)
- `scope == "parking"`이면 phase2 자체를 실행하지 않음

## 핸드오프

`sketch.md` 한 파일 phase3에 전달.
