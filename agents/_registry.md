# Agent Registry

이 폴더의 각 `.md`는 **전문 에이전트 계약서(system prompt)**다. 오케스트레이터(메인 세션)가 `Agent` 도구로 호출할 때 `prompt` 파라미터에 해당 계약서 전문 + 런타임 입력을 결합해서 전달한다.

## 오케스트레이터 역할 (이 대화 세션)

- 파이프라인 phase 순서대로 진행
- 각 phase 시작 시 해당 에이전트 호출 (Agent tool, subagent_type="general-purpose")
- 에이전트 산출물을 받아 다음 phase 에이전트에 전달
- 검증 게이트 판정(파일 존재·필드 완비·ambiguity ≤ 0.2 등)
- 사용자와의 커뮤니케이션 (진행 보고·확정 필요 질문)
- 에스컬레이션 처리 (에이전트가 판단 유보한 항목)

**오케스트레이터는 직접 WebSearch·Read·Write로 에이전트 일을 대신 하지 않는다.** 예외: 에이전트 호출 전후의 메타 작업(검증·파일 이동·task 폴더 생성).

## 에이전트 카탈로그

| 파이프라인 | Phase | 에이전트 | 계약서 |
|---|---|---|---|
| ideation | 0 capture | capture-analyst | `capture-analyst.md` |
| ideation | 1 explore | explorer | `explorer.md` |
| ideation | 2 sketch | sketch-architect | `sketch-architect.md` |
| ideation | 3 interview | interview-facilitator | `interview-facilitator.md` |
| ideation | 4 seed | seed-generator | `seed-generator.md` |
| ideation | 5 integrate | integrator | `integrator.md` |
| ideation | 6 handoff | handoff-writer | `handoff-writer.md` |
| ideation | 7 dispatch | dispatcher | `dispatcher.md` |
| refinement | 0 target | target-identifier | `target-identifier.md` |
| refinement | 1 audit | auditor | `auditor.md` |
| refinement | 2 delta | delta-analyst | `delta-analyst.md` |
| refinement | 3~6 | (ideation의 동일 에이전트 재사용) | — |

## 공통 규칙 (모든 에이전트에 적용)

1. **padlet 폴더는 읽기 전용** — Read/Glob/Grep만, Edit/Write 금지
2. **ideation 폴더는 자유** — 단 `plans/`·`research/`·`data/` 갱신은 integrator 에이전트만
3. **메모리 참조** — `/home/coseung2/.claude/projects/.../memory/MEMORY.md`의 방침 준수 (태블릿 기준·자율 진행·품질 우선 등)
4. **산출물 형식** — phase 파일 명세에 정의된 경로·포맷 엄수
5. **에스컬레이션** — 에이전트 계약서의 "Escalation" 조건 충족 시 작업 중단하고 오케스트레이터에게 원인과 차단 정보 반환
6. **보고서 간결** — 에이전트 최종 메시지는 ≤ 200자 (상세는 산출 파일에)

## 호출 포맷 (오케스트레이터가 Agent tool에 넘기는 것)

```
Agent({
  description: "Phase {N} — {에이전트 이름} 실행",
  subagent_type: "general-purpose",
  prompt: """
  {계약서 파일 전문}

  ---
  런타임 입력:
  - task_id: {task_id}
  - 입력 파일들: {경로 목록}
  - 추가 맥락: {이번 실행 특화 정보}
  """
})
```

## 에이전트 간 소통 금지

에이전트끼리 직접 통신 불가. **모든 데이터 교환은 파일(`tasks/{task_id}/phase{N}/`)을 매개**로 이루어진다. 오케스트레이터가 다음 phase에 파일 경로를 넘긴다.

## 실패 처리

- 에이전트가 실패하면(에스컬레이션·타임아웃·품질 미달) 오케스트레이터가 **같은 에이전트 재호출** (최대 3회)
- 3회 실패 시 사용자에게 원인 보고 후 개입 요청
