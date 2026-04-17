# Agent: content-curator

**Phase**: 2 — draft
**역할**: 교과서 원문 + 삽화 카탈로그 → 교사 검토용 `notebook_draft.md` 생성.

## 입력

- `phase0/request.json`
- `phase1/extract.json` 또는 `phase1/text_by_page/*.txt`
- `phase1/image_manifest.json`

## 산출

- `phase2/notebook_draft.md` (프론트매터 `status: draft`)
- `phase2/notebook.json` (구조화 원본, 검증·재생성용)
- `phase2/image_candidates.json`

## Primary Tool

**Anthropic Tool Use (JSON schema 강제)** — 스키마 위반 시 모델이 자체 재시도, 단일 의존성.

```bash
pip install anthropic jsonschema
```

## 핵심 코드 — Tool Use 스키마

```python
import anthropic, json, jsonschema

TOOL = {
  "name": "emit_notebook",
  "description": "교과서 차시를 학생 공책정리용 JSON으로 변환",
  "input_schema": {
    "type": "object",
    "required": ["lesson_title", "page_range", "learning_goals", "blocks"],
    "properties": {
      "lesson_title": {"type": "string", "maxLength": 40},
      "page_range": {"type": "string"},
      "learning_goals": {"type": "array", "items": {"type": "string"},
                         "minItems": 1, "maxItems": 3},
      "blocks": {
        "type": "array", "minItems": 3, "maxItems": 6,
        "items": {
          "type": "object",
          "required": ["type", "checkbox"],
          "properties": {
            "type": {"enum": ["keyword_def", "list", "callout", "bubble", "fill_blank"]},
            "keyword": {"type": "string", "maxLength": 20},
            "explanation": {"type": "string", "maxLength": 120},
            "items": {"type": "array", "items": {"type": "string"}},
            "text": {"type": "string"},
            "speaker": {"type": "string"},
            "sentence": {"type": "string"},
            "answer": {"type": "string"},
            "image_candidate_ids": {"type": "array", "items": {"type": "string"}},
            "image_search_query": {"type": "string"},
            "checkbox": {"type": "boolean", "const": true}
          }
        }
      },
      "excluded_candidates": {
        "type": "array",
        "items": {"type": "object", "required": ["keyword", "reason"],
                  "properties": {"keyword": {"type": "string"},
                                 "reason": {"type": "string"}}}
      }
    }
  }
}

SYSTEM = """너는 초등 공책정리 도우미다.
원칙:
- 키워드는 교과서 본문에서 추출한 어휘만 사용. AI 창작 금지.
- 페이지당 블록 3~6개. 과도 밀도 지양.
- 설명은 한 문장, 초등 어휘만.
- "fill_blank"의 sentence는 교과서의 '한 문장 정리' 원문 우선.
- 이미지 후보는 phase1 image_manifest.images[].id 로 지정.
  적합한 교과서 삽화가 없으면 image_search_query 필드에 영어 학술명 쿼리 제공.
- 교사가 뺀 것도 볼 수 있게 excluded_candidates에 이유와 함께 기록."""

client = anthropic.Anthropic()
resp = client.messages.create(
    model="claude-sonnet-4-5", max_tokens=4000,
    tools=[TOOL], tool_choice={"type": "tool", "name": "emit_notebook"},
    system=SYSTEM,
    messages=[{"role": "user", "content": user_prompt_with_raw_text}]
)
note = resp.content[0].input
jsonschema.validate(note, TOOL["input_schema"])
```

## 마크다운 변환

```python
def to_md(note):
    md = f"---\ntask_id: {task_id}\nstatus: draft\n---\n\n"
    md += f"## 📄 {note['lesson_title']} (교과서 {note['page_range']})\n\n"
    md += "### ◎ 학습 목표\n"
    for g in note["learning_goals"]:
        md += f"- [x] {g}\n"
    md += "\n### 🔑 키워드:설명\n"
    for b in note["blocks"]:
        if b["type"] == "keyword_def":
            md += f"- [x] **{b['keyword']}** : {b['explanation']}\n"
    md += "\n### 📌 Callout\n"
    for b in note["blocks"]:
        if b["type"] == "callout":
            md += f"- [x] {b['text']}\n"
    md += "\n### 💬 말풍선\n"
    for b in note["blocks"]:
        if b["type"] == "bubble":
            md += f"- [x] {b.get('speaker','')}: \"{b['text']}\"\n"
    md += "\n### 🖼 이미지 후보\n"
    for b in note["blocks"]:
        if b.get("image_candidate_ids"):
            for iid in b["image_candidate_ids"]:
                md += f"- [ ] `{iid}` — 교과서 삽화\n"
        if b.get("image_search_query"):
            md += f"- [ ] `external:search:{b['image_search_query']}` — 외부 CC 서치\n"
    # fill_blank
    for b in note["blocks"]:
        if b["type"] == "fill_blank":
            md += f"\n### 📝 한 문장 정리\n- {b['sentence']} (정답: {b['answer']})\n"
    md += "\n---\n\n## 🗑 LLM이 뺐지만 교사가 복구 가능한 후보\n"
    for ex in note.get("excluded_candidates", []):
        md += f"- **{ex['keyword']}** — (이유: {ex['reason']})\n"
    md += "\n## 📝 교사 추가 지시\n> 여기에 쓰세요.\n"
    return md
```

## Fallback

- **Instructor + Pydantic**: OpenAI·Gemini 병행 시. 단일 Claude 환경엔 과투자
- **Guided JSON (llama.cpp)**: 로컬 추론. 한글 품질 낮아 교과서 맥락엔 비추
- 블록 수 미달 시 차시 원문 ±1 차시 병합 후 재호출

## 선별 원칙

1. 키워드는 교과서 본문 어휘만
2. 페이지당 블록 3~6개
3. fill_blank는 교과서 원문 복제
4. 이미지 후보는 교과서 삽화 우선, 없을 때만 `image_search_query`
5. 🗑 섹션에 뺀 이유 기록 — 투명성

## 허용 도구

`Read`, `Write`, `Bash`(Python + anthropic SDK), `Grep`

## 검증 게이트

- `jsonschema.validate(note, TOOL.input_schema)` 통과
- 모든 차시 블록 ≥ 3
- 이미지 후보 or 검색 쿼리 ≥ 1 (차시당)
- 🗑 섹션·교사 지시 섹션 포함
- 프론트매터 `status: draft`

## 실패 처리

- 키워드 중복 (`set` 길이 불일치) → 재생성
- 학습목표 동어반복 → 시스템 프롬프트에 "직전 생성과 다른 표현" 추가 후 재시도
- 원문 너무 짧음 → `_questions.md` 기록 + 차시 범위 재검토 요청

## 참고

- https://docs.anthropic.com/en/docs/build-with-claude/tool-use
- https://json-schema.org/learn/miscellaneous-examples
