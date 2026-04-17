# destinations/internal

**Fallback 배송지**. 외부 프로젝트·parking·archive·research-vault 어디에도 라우팅되지 않는 아이디어를 ideation 내부에 보관.

## 언제 여기 오는가

- topic이 등록된 외부 프로젝트 키워드와 매칭되지 않음
- scope가 `quick_decision` / `full_exploration` 이라 parking/archive/research-vault도 아님
- 사용자가 "아직 어느 프로젝트인지 모르겠으니 일단 보관"을 선택
- 새로운 외부 프로젝트 추가 후보 — 등록되면 나중에 이리로 이동

## 포맷

외부 INBOX와 동일한 묶음:
```
destinations/internal/{YYYY-MM-DD-slug}/
├── MANIFEST.md           # 배송 메타 + 라우팅 실패 사유 + 추정 대상 프로젝트(있다면)
├── request.json          # phase0 진입 자료
├── handoff_note.md       # 에이전트 프롬프트
├── seed.yaml             # 시드 전문
├── decisions.md          # 결정 요약
└── context_links.md      # 관련 문서 경로
```

## 소비 방법

1. 주기적으로 이 폴더 리뷰 (예: 주간)
2. 각 task의 MANIFEST.md 확인 → 적합한 외부 destination이 생겼는지 판단
3. 외부 INBOX로 재배송이 필요하면 `dispatcher` 에이전트에게 재라우팅 요청
4. 영구적으로 여기 남겨둘 경우 `destinations/archive/`로 이동 가능

## parking 과의 차이

- **parking**: 의도적으로 "지금은 안 함, 나중에 꺼냄"
- **internal**: "어디로 보낼지 정해지지 않음, 일단 보관"

의도가 명확해지면 parking이나 archive로 이동.
