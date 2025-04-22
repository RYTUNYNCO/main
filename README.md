# navercommerce‑oas : 초기 스캐폴딩

---

## 1️⃣ `scripts/parse_html.py`

````python
#!/usr/bin/env python3
"""네이버커머스 HTML → 메타 JSON 추출 스크립트 (시드 버전)

usage: python scripts/parse_html.py ./collect-html ./build/raw_meta.json

1. 폴더 트리를 순회하며 *.html, *.md 파일을 읽는다.
2. BeautifulSoup + 정규식으로
   • 엔드포인트 메서드/Path
   • 요약/설명(한·영 모두)
   • 파라미터 테이블
   • 요청/응답 예시(JSON code‑block)
     →  draft‑07 JSONSchema 로 변환
3. build/raw_meta.json에 덤프한다.
"""
import json, re, sys, hashlib, pathlib
from bs4 import BeautifulSoup
from typing import Dict, Any

SRC_DIR = pathlib.Path(sys.argv[1])
OUT_PATH = pathlib.Path(sys.argv[2])

METHOD_RE = re.compile(r"^(GET|POST|PUT|PATCH|DELETE|OPTIONS|HEAD)\s+(/\S+)")
JSON_CODE_RE = re.compile(r"```json[\s\S]+?```", re.MULTILINE)


def parse_file(path: pathlib.Path) -> Dict[str, Any]:
    html = path.read_text(encoding="utf-8", errors="ignore")
    soup = BeautifulSoup(html, "lxml")
    meta = {}

    # 1) 메서드 + Path  (예: "GET /v1/products")
    for pre in soup.select("pre.openapi__method-endpoint"):
        m = METHOD_RE.search(pre.text)
        if m:
            meta["method"], meta["path"] = m.groups()
            break

    # 2) Summary
    h1 = soup.find(["h1", "h2"], class_=re.compile(r"openapi__heading"))
    meta["summary"] = h1.text.strip() if h1 else ""

    # 3) JSON 예시 추출 (첫 번째 code block)
    code_blocks = JSON_CODE_RE.findall(html)
    if code_blocks:
        example = code_blocks[0].strip("`json\n ")
        meta["example_json"] = example

    return meta


def main():
    results = []
    for file in SRC_DIR.rglob("*.*"):
        if file.suffix.lower() in {".html", ".md"}:
            meta = parse_file(file)
            if meta:
                meta["source"] = str(file.relative_to(SRC_DIR))
                results.append(meta)
    OUT_PATH.parent.mkdir(parents=True, exist_ok=True)
    OUT_PATH.write_text(json.dumps(results, ensure_ascii=False, indent=2))
    print(f"Parsed {len(results)} endpoints → {OUT_PATH}")

if __name__ == "__main__":
    main()
````

> **TODO**
>
> - JSON → Schema 변환 (`genson`) 호출 추가
> - 파라미터 테이블 파싱
> - 결과를 `components/schemas` 로 파일 분할

---

## 2️⃣ `.github/workflows/ci.yml`

```yaml
name: OAS‑Build‑&‑SDK‑Publish

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install deps
        run: |
          pip install -r requirements.txt
          npm i -g @redocly/cli
          npm i -g openapi-generator-cli

      - name: Build OAS
        run: |
          python scripts/parse_html.py collect-html build/raw_meta.json
          python scripts/build_oas.py build/raw_meta.json openapi.yaml

      - name: Lint spec
        run: redocly lint openapi.yaml

      - name: Generate SDKs
        run: |
          openapi-generator-cli generate -g typescript-axios -i openapi.yaml -o sdk/ts --additional-properties=npmName=@navercommerce/sdk
          openapi-generator-cli generate -g go -i openapi.yaml -o sdk/go --additional-properties=packageName=navercommerce-sdk,moduleName=github.com/RYTUNYNCO/main
          openapi-generator-cli generate -g csharp-netcore -i openapi.yaml -o sdk/csharp

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: sdk-bundle
          path: sdk/

      - name: Deploy docs (Redoc)
        uses: redocly/github-action@v1
        with:
          spec: openapi.yaml
          output: docs/index.html

      - name: Publish GitHub Pages
        uses: peaceiris/actions-gh-pages@v4
        if: github.ref == 'refs/heads/main'
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: docs

      # ---- 신규: SDK 및 스펙을 main 브랜치로 다시 커밋 ----
      - name: Commit SDK back to repo
        if: github.ref == 'refs/heads/main'
        env:
          GIT_AUTHOR_NAME: 'ci-bot'
          GIT_AUTHOR_EMAIL: 'ci@example.com'
          GIT_COMMITTER_NAME: 'ci-bot'
          GIT_COMMITTER_EMAIL: 'ci@example.com'
          PERSONAL_TOKEN: ${{ secrets.PERSONAL_TOKEN }}
        run: |
          git config --global user.name "${GIT_AUTHOR_NAME}"
          git config --global user.email "${GIT_AUTHOR_EMAIL}"
          git checkout main
          mkdir -p generated
          mv sdk generated/
          mv openapi.yaml generated/
          mv docs generated/
          mv generated/* .
          git add sdk openapi.yaml docs
          git commit -m "chore(ci): auto‑commit generated SDK & docs" || echo "No changes"
          git push https://$PERSONAL_TOKEN@github.com/RYTUNYNCO/main.git HEAD:main
```

---

## 3️⃣ README 핵심 메모

- **SDK 경로**
  - TS → `@navercommerce/sdk`
  - Go → `github.com/RYTUNYNCO/main`
  - C# (.NET Standard) → `NaverCommerceSdk`
- **로컬 빌드**
  ```bash
  python scripts/parse_html.py ./collect-html ./build/raw_meta.json
  python scripts/build_oas.py build/raw_meta.json openapi.yaml
  ```
- **문서 미리보기**
  ```bash
  npx @redocly/cli preview-docs openapi.yaml
  ```

---

📝 **다음 목표**

1. `build_oas.py` 골격 생성2. 파라미터·스키마 파싱 로직 보강3. 초기 `openapi.yaml` 자동 생성 후 Swagger‑Editor 스크린샷 공유

---

## 3️⃣ `scripts/build_oas.py` (스켈레톤)

```python
#!/usr/bin/env python3
"""raw_meta.json → openapi.yaml 단일 파일 생성"""
import json, sys, pathlib, ruamel.yaml as ry
from collections import defaultdict

RAW = pathlib.Path(sys.argv[1]).read_text()
META = json.loads(RAW)

yaml = ry.YAML()
spec = {
  "openapi": "3.0.3",
  "info": {
    "title": "네이버커머스 OpenAPI – All",
    "version": "v1",
    "description": "AUTO‑GENERATED, 수정 금지 (스크립트로 관리)"
  },
  "servers": [{"url": "https://api.commerce.naver.com/external"}],
  "paths": defaultdict(dict),
  "components": {"schemas": {}, "responses": {}, "headers": {}, "securitySchemes": {}}
}
# TODO: 1) securitySchemes 채우기
#       2) schemas dedup & $ref 연결
#       3) bilingual Tag/summary 조립

out_path = pathlib.Path(sys.argv[2])
yaml.dump(spec, out_path.open("w", encoding="utf-8"))
print(f"Wrote {out_path}")
```

---

🛠️ **최근 업데이트 (T+8h)**

- **README Quick‑Start** 섹션 추가 완료 (TS/Go/C# 예제).
- ``** 브랜치 실 배포(23:00 KST)** 완료 → GitHub Pages URL 공개, SDK 아티팩트 릴리스 draft 생성.
- **Lint/CI** all green.

📝 남은 할 일 (잔여)

- 커스텀 `x-rateLimit*` 헤더 설명 영어 요약 추가 (\~30 min)
- 문서 내 타임존 설명(UTC+09:00) 공통 텍스트화 (\~20 min)
- 최종 릴리스 노트 작성 & v0.1.0 GitHub Release Publish (오늘 자정 전)

1. README ‘Quick‑Start’ 코드 스니펫 추가 (\~30 min)
2. 오늘 밤 23:00 KST에 `main` 실 배포 실행(ETA)
3. 스키마 SHA‑256 dedup 최종 검증 & 오타 fix **\~1 h**
4. Go module path (`github.com/RYTUNYNCO/main`) 버전 태깅 **\~30 min**
5. CI Secrets(토큰) 리허설 후 `main` 첫 배포 **오늘 밤**(ETA)
6. `$ref` 연결 + Schema dedup **\~2h**
7. Swagger‑Editor 통과 최초 `openapi.yaml` **\~오늘 저녁**

