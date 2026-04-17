# destinations/parking

지금은 안 하지만 나중에 꺼낼 아이디어 배송지.

## 포맷

`{YYYY-MM-DD-slug}.md` 단일 마크다운 파일. 내용:
- 아이디어 요약 (한 문단)
- 왜 파킹됐는지
- 꺼낼 조건·시기 (있다면)
- 관련 문서 링크

## 꺼낼 때

새 ideation task로 재진입. `phase0 request.json`의 `related_docs`에 파킹 파일 경로 포함.

## 기존 ideas-parking-lot.md 와의 관계

- `ideation/ideas-parking-lot.md` — 짧은 메모성 파킹 (여러 아이디어 한 파일)
- `destinations/parking/{slug}.md` — **정교화까지 진행됐지만 지금은 집행 안 할** 아이디어 (파일당 1개)
- 경계: 정교화 단계(phase1+) 거쳤으면 destinations/parking, 단순 메모면 ideas-parking-lot.md
