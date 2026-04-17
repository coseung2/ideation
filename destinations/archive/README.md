# destinations/archive

종결·폐기·수퍼시드된 아이디어 저장소. 읽기 전용 역사.

## 포맷

```
destinations/archive/{YYYY-MM-DD-slug}/
└── summary.md
```

`summary.md` 내용:
- 아이디어 제목·요약
- 왜 아카이브됐는지 (폐기 사유·수퍼시드 관계)
- 관련 시드 ID 스냅샷
- 교훈(있다면)

## 언제 여기 오는가

1. 사용자가 명시적으로 "아카이브" 요청
2. refinement로 수퍼시드된 이전 시드 (`seeds-index.md`에서 "superseded by seed_X" 표시와 동시에)
3. 폐기 결정된 기능 (예: P1-⑦ Teamspace 동기화)

## 복구

아카이브에서 돌려내려면 새 ideation task로 재진입. 이전 시드는 parent_seed_id 체인에 남음.
