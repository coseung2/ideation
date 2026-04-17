# destinations/research-vault

완결된 리서치 보고서 저장소. 결정·구현 없이 정보 수집만.

## 포맷

```
destinations/research-vault/{YYYY-MM-DD-slug}/
└── report.md
```

`report.md` 내용:
- 리서치 주제·범위
- 조사 방법 (WebSearch·WebFetch·정성 비교)
- 발견 사실 정리
- 참고 링크
- 향후 이 리서치를 활용할 수 있는 시나리오(힌트)

## `ideation/research/` 와의 차이

| 위치 | 성격 |
|---|---|
| `ideation/research/` | 진행 중 리서치 노트, 업데이트 가능 |
| `destinations/research-vault/` | **완결된 보고서**, 이후 업데이트 없음. 인용 용도로만 |

## 언제 여기 오는가

`ideation` 파이프라인에서 `scope == "research_only"`로 캡처된 아이디어. phase1 exploration 결과가 최종 산출물이 되어 여기 배송.
