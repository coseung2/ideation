# Agent: html-builder

**Phase**: 5 — render
**역할**: 승인 draft + 확정 이미지 → `template.html` 주입 → 완성 HTML.

## 입력

- `phase3/notebook_draft.approved.md` + `phase2/notebook.json` (구조화 원본)
- `phase4/image_map.json`, `phase4/images/`
- `../../template.html` (스켈레톤)

## 산출

- `phase5/notebook.html` — 상대경로 이미지
- `phase5/notebook.standalone.html` — 이미지 base64 임베드 (Chromium file:// 이슈 회피용, phase6에서 사용)

## Primary Tool

**Jinja2 + mistune** — Python 네이티브, 한글 안전, 루프 자연스러움.

```bash
pip install jinja2 mistune
```

## 핵심 코드

```python
from jinja2 import Environment, FileSystemLoader
import json, base64, mimetypes, pathlib

note = json.load(open("phase2/notebook.json", encoding="utf-8"))
img_map = json.load(open("phase4/image_map.json", encoding="utf-8"))

# 이미지 base64 임베드 (Chromium file:// 한글경로 이슈 회피)
for block_id, img in img_map.items():
    m = mimetypes.guess_type(img["file"])[0] or "image/jpeg"
    data = base64.b64encode(open(img["file"], "rb").read()).decode()
    img["data_uri"] = f"data:{m};base64,{data}"

env = Environment(loader=FileSystemLoader("templates"), autoescape=True,
                  trim_blocks=True, lstrip_blocks=True)
html = env.get_template("notebook.html.j2").render(note=note, img_map=img_map)
pathlib.Path("phase5/notebook.standalone.html").write_text(html, encoding="utf-8")
```

## 템플릿 예시 (`notebook.html.j2`)

```jinja
{% extends "base.html.j2" %}
{% block pages %}
{% for page in note.pages %}
<section class="page">
  <div class="top-bar">
    <div>
      <div class="unit-tag">{{ note.subject_grade }} ▸ {{ note.unit_name }}</div>
      <div class="chapter-title">{{ page.chapter_title }}</div>
      <div class="date-line">{{ page.lesson_num }}차시 ─ {{ page.lesson_question }}</div>
    </div>
    <div class="page-ref">교과서 p.{{ page.page_from }}~{{ page.page_to }}</div>
  </div>

  <div class="goals"><strong>◎ 학습 목표</strong> ─ {{ page.goal }}</div>

  <div class="grid">
    <div class="block">
      <h3>{{ page.section_a.title }}</h3>
      {% for b in page.section_a.blocks if b.checked %}
        {% if b.type == "keyword_def" %}
          <div class="kv"><span class="k">{{ b.keyword }}</span>
            <span class="v">: {{ b.explanation }}</span></div>
        {% elif b.type == "callout" %}
          <div class="callout">{{ b.text }}</div>
        {% endif %}
      {% endfor %}
    </div>
    <div class="block">
      {% set img = img_map.get(page.id ~ ".section_b") %}
      {% if img %}
      <div class="image-slot">
        <img src="{{ img.data_uri }}" alt="{{ img.alt_text }}">
        <div class="caption">{{ img.source_detail }}</div>
        <div class="attribution">{{ img.attribution or img.license }}</div>
      </div>
      {% endif %}
    </div>
  </div>

  {% for b in page.bubbles if b.checked %}
    <div class="bubble">💬 {{ b.speaker }}: "{{ b.text }}"</div>
  {% endfor %}

  {% if page.fill_blank %}
    <div class="fill-blank">
      {{ page.fill_blank.sentence | replace("___", '<span class="blank">' ~ page.fill_blank.answer ~ '</span>') | safe }}
    </div>
  {% endif %}

  <div class="footer">✎ p.{{ loop.index }}</div>
</section>
{% endfor %}
{% endblock %}
```

## CSS 주의사항

- template.html의 CSS는 수정 금지 (스타일 표준 유지)
- 이미지 블록은 `.image-slot` 클래스만 사용
- **SVG·인라인 도안 직접 생성 금지** (v1 이슈 재발 방지)
- 한글 줄바꿈: `word-break: keep-all; overflow-wrap: break-word;`
- 페이지 나눔: `.block { break-inside: avoid; }` + 5~6블록/페이지

## HTML 검증

```bash
pip install html5validator
html5validator --root phase5/ --match "*.html"
```

## Fallback

- **Handlebars/Eta** (Node 스택 합칠 때)
- **Claude 직접 HTML 생성** (phase2 확장) — 단 수정·재렌더 비용↑ 비권장

## 허용 도구

`Read`, `Write`, `Edit`, `Bash`(Python + jinja2)

## 검증 게이트

- 모든 이미지 `src`가 data_uri 또는 실재 파일
- HTML 페이지 수 == draft 페이지 수
- `html5validator` 심각 오류 0
- 🗑 섹션·📝 교사 지시 섹션의 내용은 HTML에 포함되지 않음

## 실패 처리

- 이미지 경로 해석 실패 → 회색 `.image-placeholder` 박스 대체 + `_questions.md` 기록
- template 문법 오류 → 직전 커밋된 template 버전으로 롤백

## 참고

- https://jinja.palletsprojects.com/
- https://mistune.lepture.com/
