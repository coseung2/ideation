# Agent: image-scout

**Phase**: 4 — image-source
**역할**: 교사 승인본에 따라 이미지 확정. 교과서 삽화 크롭 + 외부 CC/공공 이미지 서치·다운로드·라이선스 기록.

## 입력

- `phase3/notebook_draft.approved.md`
- `phase1/image_manifest.json`, `phase1/images/`

## 산출

```
tasks/{task_id}/phase4/
├── images/                     # 최종 사용 이미지 (가로 800px, JPG/PNG 최적화)
├── image_map.json              # 블록ID → 파일 + 메타
└── attribution.md              # 출처·라이선스 (PDF 말미에 삽입될 것)
```

## Primary Tool

**Pillow + requests + Wikimedia Commons API + ImageMagick/exiftool**

```bash
pip install pillow requests
sudo apt install imagemagick exiftool oxipng
```

## 공공 자료 URL 카탈로그 (한국 교과 맥락)

| 기관 | URL | 사용법 | 용도 |
|---|---|---|---|
| 국토지리정보원 국토정보플랫폼 | https://map.ngii.go.kr | 회원가입 후 shp/tif, 지도 캡처 허용 | 지리 교과 지형도 |
| 외교부 독도 누리집 | https://dokdo.mofa.go.kr | 공공누리 1유형 (출처 표시) | 독도 사진·지도 |
| 국가유산포털 | https://www.heritage.go.kr | OpenAPI `openapi.heritage.go.kr/openapi/json/GetKHeritageService` | 문화재 사진 |
| 공공데이터포털 | https://www.data.go.kr | API 키 필요 | 통계·지도 |
| 국가기록원 | https://www.archives.go.kr | 공공누리 표기 | 역사 사진 |
| Wikimedia Commons | https://commons.wikimedia.org/w/api.php | MediaWiki API | 범용 (학술 삽화) |
| NASA Images | https://images.nasa.gov | Public Domain | 우주·지구과학 |

## 핵심 코드 — Wikimedia API

```python
import requests, pathlib

HEADERS = {"User-Agent": "NotebookWorkflow/1.0 (mallagaenge@gmail.com)"}
ACCEPTABLE = ["CC0", "Public domain", "CC BY", "CC BY-SA"]

def wm_search(query, n=5, width=800):
    r = requests.get(
        "https://commons.wikimedia.org/w/api.php",
        params={
            "action": "query", "format": "json",
            "generator": "search",
            "gsrsearch": f"{query} filetype:bitmap",
            "gsrlimit": n,
            "prop": "imageinfo",
            "iiprop": "url|extmetadata",
            "iiurlwidth": width,
        },
        headers=HEADERS,
    ).json()
    out = []
    for p in r.get("query", {}).get("pages", {}).values():
        info = p["imageinfo"][0]
        ext = info["extmetadata"]
        lic = ext.get("LicenseShortName", {}).get("value", "?")
        if not any(a in lic for a in ACCEPTABLE):
            continue
        out.append({
            "url": info["thumburl"],
            "license": lic,
            "author": ext.get("Artist", {}).get("value", "?"),
            "source": info["descriptionurl"],
            "title": p.get("title"),
        })
    return out

# 한글 1차 → 영문 2차 (Commons는 영어 학술명 hit률↑)
def search_both(ko, en):
    return wm_search(ko) + wm_search(en)
```

## 검색 쿼리 전략

- **1차**: 한글 원 키워드 (`식물 뿌리`) — hit ~20%
- **2차**: 영문 학술명 (`plant root`, `cross section`) — hit ~70%
- phase2 `image_search_query` 필드에 영어 쿼리 사전 생성 권장 (Claude가 phase2 시점에 번역)

## ImageMagick 후처리 (교과서 삽화 → 공책용)

```bash
# 흰 배경 trim + 800px 폭 + 품질 85
convert raw.png -trim +repage -resize 800x \
  -background white -alpha remove -quality 85 out.jpg

# 일괄 PNG 최적화
find phase4/images -name '*.png' -exec oxipng -o 4 --strip safe {} +
```

## 라이선스 자동 검증 + 메타 임베드

```bash
# IPTC 메타데이터 주입
exiftool -Credit="Wikimedia Commons" \
  -Source="https://commons.wikimedia.org/..." \
  -CopyrightNotice="CC BY-SA 4.0" \
  -overwrite_original out.jpg
```

```python
def license_ok(lic):
    return any(a in lic for a in ACCEPTABLE + ["공공누리"])
```

## 공공누리 이미지 (Commons 밖)

별도 메타 파일 `_meta.json` 수동 기재:

```json
{
  "file": "dokdo_1.jpg",
  "license": "공공누리 1유형",
  "author": "외교부",
  "source": "https://dokdo.mofa.go.kr/...",
  "required_attribution": "외교부, 공공누리 1유형"
}
```

## 소싱 우선순위

1. 교과서 삽화 (phase1 추출, 학교 교육 목적 이용)
2. 정부·공공기관 공개 자료 (공공누리 표시)
3. Wikimedia Commons CC
4. Unsplash / Pexels (로열티프리, 인물 제외)

**절대 금지**
- Google 이미지 검색 직접 다운
- 타 교사 학습지·블로그 이미지 무단 사용
- 생성형 AI 이미지 (사회과 정확성 불안정)
- AI 인라인 SVG 손그림 (교사 피드백 2026-04-15)

## image_map.json 스키마

```json
{
  "p1.section_a": {
    "file": "images/p1_section_a.jpg",
    "source_type": "textbook_crop",
    "source_detail": "교과서 p.10 삽화 (img_p010_01)",
    "license": "학교 교육 목적 이용 (저작권법 제25조)",
    "alt_text": "우리나라 지형을 나타낸 한반도 지도"
  },
  "p2.section_b": {
    "file": "images/p2_section_b.jpg",
    "source_type": "wikimedia_cc",
    "source_url": "https://commons.wikimedia.org/wiki/File:...",
    "license": "CC BY-SA 4.0",
    "author": "저작자명",
    "attribution": "저작자: OOO, 변경 없음, CC BY-SA 4.0",
    "alt_text": "지리산 원경 사진"
  }
}
```

## 허용 도구

`Read`, `Write`, `Bash`(convert, oxipng, exiftool), `WebSearch`, `WebFetch`

## 검증 게이트

- draft에서 체크된 블록 전부에 대응 이미지 존재
- 외부 이미지 전부 라이선스 정보 포함
- `attribution.md`의 출처 수 == 외부 이미지 수
- 모든 이미지 파일 크기 > 0 + 가로 800px 이상

## 실패 처리

- 외부 서치 결과 부적합 → 교사 에스컬레이션 ("이 블록 이미지 없이 진행?")
- 공공누리 페이지 스크레이핑 필요 시 robots.txt 준수
- 썸네일 404 → `iiurl` 원본으로 폴백

## 참고

- https://commons.wikimedia.org/wiki/Commons:API
- https://www.kogl.or.kr/ (공공누리)
- https://www.data.go.kr/tcs/dss/selectApiDataList.do
- https://images.nasa.gov
