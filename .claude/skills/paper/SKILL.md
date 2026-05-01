---
name: paper
description: "Papers 포스트 생성 파이프라인 (From AI 섹션). arXiv/학술 저널/기술 블로그 URL을 요약하여 포스트로 발행하고 terry-papers 지식그래프에 노드로 추가한다. virtual(가상 논문), synthesis(다중 소스 종합)도 지원. --visibility=public(기본)/private/group 지원. 커버 이미지 없을 시 /image-gen 사용. essays/notes 발행은 /post 스킬(terry-obsidian) 사용."
argument-hint: "<URL | virtual 자연어요청 | synthesis URL1 URL2 ...> [--visibility=public|private|group --group=<g>] [--tags=TAG1,TAG2] [--memo=메모] [--featured] [--pdf=<path>]"
---

# Papers 포스트 생성 파이프라인

입력: $ARGUMENTS

> 이 스킬은 **외부 콘텐츠 요약 전용**이다. `export-knowledge.mjs`를 실행해 KB 노드를 자동 추가한다. Terry 본인이 쓴 essays와 notes(짧은 메모 또는 ChatGPT 대화 요약)는 `/post` 스킬(terry-obsidian canonical)을 사용할 것.

## Step 0) 타입 자동 감지

- `https://arxiv.org/` 포함 → **Papers 경로 (arXiv 논문)** — `docs/POST_LOADING_ARXIV.md` 참조
- `nature.com`, `ieee.org`, `acm.org` 등 학술 저널 URL → **Papers 경로 (학술 논문)** — `docs/POST_LOADING_ETC.md` 참조
- 그 외 `http(s)://` URL (Claude 블로그, Generalist 블로그, 기업 기술문서 등) → **Papers 경로 (기술 블로그)** — `docs/POST_LOADING_BLOG.md` 참조
- `virtual` 키워드로 시작 → **Virtual Paper 경로** (가상 논문, Research Proposal)
- `synthesis` 키워드로 시작 → **Synthesis 경로** (다중 소스 종합)

### 예시

```
/paper https://arxiv.org/abs/2505.22159          → Papers (arXiv 논문)
/paper https://nature.com/articles/...           → Papers (학술 논문)
/paper https://claude.com/blog/...               → Papers (기술 블로그)
/paper https://generalistai.com/blog/...         → Papers (기술 블로그)
/paper virtual --group=snu 촉각 잔차 학습 논문      → Virtual Paper
/paper synthesis https://github.com/u/r https://x.com/u/s/123  → Synthesis
```

### PDF Fallback 규칙 (URL 접근 불가 시)
- URL이 접근 불가(403/402/303 등)이고 사용자가 `--pdf=<path>` 또는 별도 PDF 파일 경로를 제공한 경우:
  1. WebFetch 먼저 시도 → 실패 시 PDF로 fallback
  2. PDF에서 `pymupdf`로 텍스트 추출 + 이미지 추출
  3. 추출된 텍스트로 메타데이터(제목, 저자, 날짜) 파악
  4. 추출된 이미지에서 cover 후보 선택
  5. 나머지 파이프라인은 해당 소스 경로(학술/블로그)와 동일하게 진행
- PDF 파일은 `posts/papers/<slug>/paper/<slug>.pdf`에 복사

### 인덱싱 규칙 (terryum-ai `docs/INDEXING.md` 참조)
- 공개 포스트: 양수 ID (`posts/global-index.json`의 `next_public_id` 사용)

### 공개 범위 (visibility) 옵션

- `--visibility=public` (기본): Git에 파일 저장, `posts/global-index.json` 공개 엔트리 편입, Cloudflare Workers 배포, Obsidian sync
- `--visibility=private`: 비공개 단일 사용자 발행
  - Supabase `private_content` 테이블에만 저장 (Git/CF 배포 스킵)
  - global-index.json의 비공개 엔트리 (gitignored)
  - 자신의 로그인 세션에서만 노출
- `--visibility=group --group=snu|korea` → 그룹 전용 포스트
  - **Git에 파일을 생성하지 않음** — Supabase `private_content` 테이블에 직접 저장
  - MDX 콘텐츠 → `content_ko`, `content_en` 컬럼
  - 메타데이터 → `meta_json` 컬럼 (visibility, allowed_groups 포함)
  - 커버/썸네일/OG 이미지 → Supabase Storage `private-covers/{slug}/` 버킷
  - global-index.json에 엔트리 추가 (gitignored)
  - **Git 커밋/푸시 불필요**
  - 사이트에서 해당 그룹 로그인 세션이 있어야만 노출됨

> **새 그룹을 도입할 때**: 슬러그는 `terryum-ai/src/lib/group-auth.ts`의 `ALLOWED_GROUPS` allowlist + `.env.local`의 `CO_<SLUG>_PASSWORD` + `.env.example` 안내, 세 곳을 같이 갱신해야 한다. allowlist에 없으면 그룹 로그인이 거부된다.

---

## Papers 경로

입력: `<URL> [--tags=TAG1,TAG2] [--memo=메모] [--featured]`
URL 종류에 따라 소스별 분기가 있지만, **MDX 구조와 meta.json 스키마는 동일**.

### 소스별 분기 (Step R4에서 적용)
- **arXiv**: `docs/POST_LOADING_ARXIV.md` 참조
- **학술 저널**: `docs/POST_LOADING_ETC.md` 참조
- **블로그**: `docs/POST_LOADING_BLOG.md` 참조 — PDF 없음, WebFetch로 HTML 수집, Video 임베딩 지원

모든 단계를 권한 요청 없이 완료할 것.

**작업 디렉토리**: 이 스킬은 `terry-papers` repo에 canonical이 있지만, 실제 파일 생성/Git 작업은 **`terryum-ai` repo 기준**으로 수행한다. `posts/`, `public/posts/`, `scripts/` 경로는 모두 `/Users/terrytaewoongum/Codes/personal/terryum-ai/` 기준.

### Step R1) 요약 규칙 로드
`docs/PAPERS_SUMMARY_RULES.md` 읽기

### Step R2) URL 파싱
arXiv ID 추출, 슬러그 결정 (`YYMM-<short-name>`)

### Step R3) Graph Analysis (새 논문 추가 전)
a. `posts/index.json` 로드 → `concept_index`, `posts` 배열 확인
b. 새 논문의 key_concepts와 기존 논문의 concept_index 비교
c. **taxonomy 배치 제안**:
   - `posts/taxonomy.json`의 nodes 확인
   - primary: key_concepts/domain에서 가장 자연스러운 taxonomy 노드
   - secondary: 두 번째 겹침 노드 (있으면)
d. **관계 후보 생성**:
   - concept 2개+ 겹침 → `related`
   - 같은 VLA/method + 이 논문이 개선 → `builds_on` 또는 `extends`
   - 같은 task, 다른 방법 → `compares_with`
   - 기존 논문의 한계를 보완 → `fills_gap_of`
e. **outlier 판단**: 기존 clusters와 concept 겹침이 1개 이하이면 경고 출력
f. meta.json 생성 전에 아래를 출력:
   ```
   📊 Graph Context:
   - Taxonomy: robotics/brain (primary), robotics/arm (secondary)
   - Related: [slug] (builds_on), [slug] (compares_with)
   - Clusters: vla-robotics 클러스터에 합류
   ```

### Step R4) 메타데이터 수집
`https://export.arxiv.org/abs/<id>` API → 제목, 저자, 초록, v1 제출일

### Step R5) PDF (저장하지 않음)
- **PDF 원본은 Git에 저장하지 않는다** — arXiv 링크로 원문 접근
- Figure 추출이 필요할 때만 임시 다운로드 후 추출 완료 시 삭제

### Step R6) Figure 추출 + R2 업로드
- `GET https://arxiv.org/html/<id>v1/` → 200이면 img src 전수 추출 + 다운로드
- **404이면 PDF fallback** (Semantic Scholar 등 외부 사이트 탐색 금지):
  a. PDF를 임시 다운로드: `curl -sL https://arxiv.org/pdf/<id> -o /tmp/<slug>.pdf`
  b. `python /Users/terrytaewoongum/Codes/personal/terryum-ai/scripts/extract-paper-pdf.py /tmp/<slug>.pdf posts/papers/<slug>/`
  c. `extraction_report.json` 읽기 → figures/captions 자동 적용
  d. `suggested_cover`를 `cover.webp`로 PIL 변환
  e. 임시 PDF 삭제: `rm /tmp/<slug>.pdf`
- **figure 추출 실패 또는 cover 후보 없음** → `/image-gen` 스킬로 커버 이미지 생성
  - 프롬프트: 논문 제목/핵심 주제를 반영한 추상적 기술 일러스트, 16:9 비율, 텍스트 없이
  - 생성된 이미지를 `posts/papers/<slug>/cover.webp`로 저장
  - API 키가 없거나 생성 실패 시 → 커버 없이 진행 + 경고 출력
- **⚠ 투명배경 제거 — 필수, 분기 무관**: 위 세 경로(HTML 추출·PDF fallback·/image-gen) 어떤 것을 탔든, 이 단계 종료 직전에 반드시 1회 실행한다. 다크모드에서 투명 픽셀이 검정으로 비쳐 판독 불가가 되는 사고를 막는다 (2026-04 post #13 사고).
  ```bash
  python /Users/terrytaewoongum/Codes/personal/terryum-ai/scripts/flatten-transparent-figures.py posts/papers/<slug>/
  ```
  대상: 추출된 `fig-*.png/webp`, `cover.webp`, `cover_Original.*` 포함 포스트 디렉토리 전부. 스크립트는 실제 투명 픽셀이 없는 파일은 자동으로 skip.
- **추출된 이미지는 R2에 업로드**: `node /Users/terrytaewoongum/Codes/personal/terryum-ai/scripts/upload-to-r2.mjs --slug=<slug>`
- 로컬 이미지 파일은 Git에 추가하지 않음 (.gitignore로 제외됨)

### Step R7) MDX 생성
`docs/PAPERS_SUMMARY_RULES.md` 기준으로 `ko.mdx` + `en.mdx` 생성
- frontmatter: title, summary, card_summary, cover_caption, published_at (현재 ISO), source_type, source_url, source_date, display_tags, figures, tables, references

### Step R8) meta.json 생성
- **status 항상 `"published"`** (draft 옵션 없음)
- `--featured` 시 `featured: true`
- `display_tags`: `--tags=` 값, `terrys_memo`: `--memo=` 값
- **Graph Analysis 결과 포함**: `taxonomy_primary`, `taxonomy_secondary`, `relations`
- **`thumb_source` 선택** — 추출된 figure 중 아래 우선순위로 선택:
  1. **실물 로봇/하드웨어/실험 장면 사진** (실사 이미지 최우선)
  2. **Overview 개념도/시스템 스키마** (아키텍처 전체 흐름)
  3. cover.webp fallback (미지정 시 자동)
  - 그래프/차트/표는 썸네일로 부적절 → 선택하지 않음
  - 선택한 figure를 `"thumb_source": "./fig-N.png"`으로 기록
- **필수 메타데이터 필드**:
  - `source_title`: arXiv 원문 제목
  - `source_author`: "1저자 et al." 형식
  - `source_authors_full`: 전체 저자 목록 (이름 + 소속)
  - `source_url`, `source_type`, `source_date`
  - `google_scholar_url`: 논문 제목 기반 Scholar 검색 URL
  - `source_project_url`: 프로젝트 페이지 URL (있으면)

### Step R9) 빌드 스크립트 실행

**권장 (통합 파이프라인):**
```bash
cd /Users/terrytaewoongum/Codes/personal/terryum-ai
node scripts/post-publish-pipeline.mjs --slug=<slug>
node scripts/sync-references.mjs --slug=<slug>
```
`post-publish-pipeline.mjs`가 thumbnails / og-image / embeddings / index 를 병렬 실행하고, 이어서 flatten + R2 업로드를 직렬로 처리한다 (~30~60초 절약). `sync-references`만 paper 전용이라 개별 호출.

<details>
<summary>개별 호출 (디버깅·재시도용 reference)</summary>

```bash
cd /Users/terrytaewoongum/Codes/personal/terryum-ai
node scripts/generate-thumbnails.mjs
node scripts/generate-index.mjs
node scripts/generate-og-image.mjs
# ⚠ 투명배경 제거 — generate-thumbnails / generate-og-image가 alpha를 유지한 채 산출할 수 있으므로
# R2 업로드 직전에 public 자산도 한 번 flatten 한다.
python scripts/flatten-transparent-figures.py public/posts/<slug>/
node scripts/upload-to-r2.mjs --slug=<slug>
node scripts/sync-references.mjs --slug=<slug>
node scripts/generate-embeddings.mjs --slug=<slug>
```
</details>
- `generate-thumbnails.mjs`: `cover.webp`에서 288×288 `cover-thumb.webp`를 `public/posts/<slug>/`에 생성 (리스트 카드 썸네일용)
- `upload-to-r2.mjs`: 이미지를 Cloudflare R2에 업로드 (cover, **cover-thumb**, figures, OG)
- **썸네일 생성 → flatten → R2 업로드 순서 필수**: 투명 이미지가 R2 엣지에 1년 immutable로 캐시되면 나중에 교체해도 먹히지 않는다. 업로드 **전에** 반드시 opaque로 만들 것.
- OG 이미지는 `public/posts/<slug>/og.png`에 생성 (Cloudflare Worker 번들에 포함되어 서빙, R2 아님)

### Step R10) 포스트 검증
```bash
node /Users/terrytaewoongum/Codes/personal/terryum-ai/scripts/validate-post.mjs <slug>
```
- **에러(❌)가 있으면 반드시 수정 후 재검증** — 에러가 0일 때까지 반복
- 경고(⚠️)는 가능하면 해결, 불가능하면 사유 기록
- 검증 항목: meta.json 필수필드, figures 캡션(ko/en), 파일 존재, 제목 형식, index.json 정합성

### Step R11) Build 검증 (CF 호환 번들)
```bash
cd /Users/terrytaewoongum/Codes/personal/terryum-ai
npx opennextjs-cloudflare build
```
- Next.js 빌드 + OpenNext 어댑터로 CF Workers 호환 번들(`.open-next/`) 생성
- 이 번들이 Step R12.1에서 그대로 업로드되므로 **별도 `npm run build`는 불필요** (중복 next build 방지)
- 실패 시 에러 수정 후 재실행

### Step R12) Git 커밋 + 푸시
```bash
cd /Users/terrytaewoongum/Codes/personal/terryum-ai
git pull --rebase origin main
git add posts/ public/posts/
git commit -m "feat(post): add <slug> (ko/en)"
git push
```

### Step R12.1) Cloudflare Workers 배포 (필수)

**Git push만으로는 공개 사이트에 반영되지 않는다** — `posts/index.json`이 빌드 시점 번들에 import되므로, 새 포스트는 Worker 재배포 후에야 `/posts` 목록과 `tab=essays/papers/notes` 필터에 노출된다.

```bash
cd /Users/terrytaewoongum/Codes/personal/terryum-ai
npx opennextjs-cloudflare deploy
```

- Step R11에서 생성한 `.open-next/` 번들을 그대로 CF Workers에 업로드한다 (재빌드 없음)
- 배포 후 확인: `npx wrangler deployments list | head -5` 로 새 "Created" 항목이 생겼는지 확인
- 기대되는 확인 URL: `https://www.terryum.ai/posts/<slug>` → HTTP 200
- **CF Dashboard의 Workers Builds (Git auto-build)는 의도적으로 끊어둔 상태** — 수동 배포가 유일한 진실의 원천. 실수로 재연결하지 말 것.

### Step R12.5) Knowledge Base 업데이트 (terry-papers)
```bash
node /Users/terrytaewoongum/Codes/personal/terryum-ai/scripts/export-knowledge.mjs   # 기본 출력: ~/Codes/personal/terry-papers
node /Users/terrytaewoongum/Codes/personal/terryum-ai/scripts/sync-obsidian-kg.mjs   # KG → vault/Papers KB/ 시각화
cd ~/Codes/personal/terry-papers && git add papers/ knowledge-index.json \
  && git commit -m "kb: <slug>" && git push && cd -
```
- 별도 KB 레포는 없다 — `papers/<slug>.json`과 `knowledge-index.json`은 `terry-papers` 레포에 직접 커밋된다
- Terry's memo가 있는 포스트만 의미 있는 변경이 생기지만, 매 /paper마다 갱신해서 인덱스를 최신 상태로 유지
- `sync-obsidian-kg.mjs`는 vault의 `Papers KB/` 폴더를 매번 비우고 재생성 — typed-edge wikilinks가 Obsidian Graph View에 즉시 반영. 실패 시 경고만 출력하고 계속 진행 (vault 미설정도 비차단).

### Step R12.6) Obsidian Vault Sync (full posts)
- Run: `node /Users/terrytaewoongum/Codes/personal/terryum-ai/scripts/sync-obsidian.mjs --slug=<slug>`
- If vault directory does not exist or script fails, print warning and continue (non-blocking)
- This ensures the new/updated post appears in Obsidian immediately
- 위 R12.5의 `sync-obsidian-kg.mjs`와는 별개 — 이쪽은 `Public/Papers/<slug>.mdx` 본문, KG sync는 `Papers KB/<slug>.md` 시각화 노드

### Step R12.7) 서베이 교차 연결 (terry-surveys) — 자동 호출 필수

새 포스트가 기존 서베이의 참고문헌에 인용되어 있는지 확인하고, 매칭되면 서베이 챕터의 해당 ref에 `[post]` 링크를 자동으로 추가한다. **이 단계는 건너뛰지 않는다** — 매칭이 없으면 조용히 종료되므로 비용이 낮다.

```bash
cd /Users/terrytaewoongum/Codes/personal/terry-surveys

# 1) 인덱스 최신화 (서베이 쪽에서 ref 변경이 있었을 수 있음)
python3 bibtex/refs_index.py build

# 2) 새 포스트 slug로 매칭 검색
python3 bibtex/refs_index.py match <new-slug>
```

판정 규칙 (Tier 1 우선):
- **Tier 1 (arXiv/DOI/Nature ID exact match) 매칭이 하나라도 있으면** → 해당 서베이의 ref에 즉시 `[#NN]` 링크를 삽입 (false positive 위험 없음)
- **Tier 3 (slug-token fuzzy, `⚠️  REQUIRES HUMAN REVIEW`) 매칭만 있으면** → **자동 링크 금지**. 사용자에게 매칭 후보를 보고하고 확인을 받은 뒤에만 삽입
- 어느 Tier든 매칭이 있으면 → `/link-post-to-surveys <new-slug>` 스킬을 즉시 호출
  - link-post-to-surveys가 매칭된 모든 서베이 챕터의 `## 참고문헌` / `## References` 항목에 `[post]` 링크를 삽입하고 각 서베이를 `python3 build.py <survey-name>`으로 리빌드한다
  - **snu-tactile-hand가 매칭에 포함되면** 추가로 `bash /Users/terrytaewoongum/Codes/personal/terry-surveys/surveys/snu-tactile-hand/scripts/push-private.sh "link post <new-slug> to snu-tactile-hand"` 를 실행해 private repo에 즉시 반영 (Cloudflare Pages 자동 재배포 트리거)
- 매칭 score가 모두 < 3이면 → 서베이에 이 논문이 인용되지 않은 것으로 간주하고 조용히 종료
- 매칭은 있는데 link-post-to-surveys 실행이 실패하면 → 실패 이유를 기록하고 Step R13의 레슨에 추가

### Step R13) 예외 발생 시 레슨 기록
포스팅 과정에서 **예외/우회/실패 후 복구**가 있었다면:
1. 문제와 해결 방법을 `memory/posting-lessons.md`에 추가 (memory 시스템 사용)
2. 원인이 파이프라인 규칙 누락이면 해당 docs 파일도 업데이트
3. 검증 스크립트에서 잡지 못한 문제면 `validate-post.mjs`에 체크 추가

---

## Papers 경로 플래그 파싱 규칙

- `--tags=VLA,robotics`: display_tags 배열로 변환
- `--memo=내용`: terrys_memo 필드로 저장
- `--featured`: meta.json에 `featured: true` 추가

## PDF fallback 상세 (`extract-paper-pdf.py`)

```
사용법: python /Users/terrytaewoongum/Codes/personal/terryum-ai/scripts/extract-paper-pdf.py <pdf_path> <output_dir> [--max-pages=38]
출력:  output_dir/fig-N.{ext} 파일들 + extraction_report.json
```

`extraction_report.json` 구조:
```json
{
  "figures": [
    {"seq_num": 1, "file": "fig-1.png", "page": 5, "caption": "...", "is_cover_candidate": false},
    {"seq_num": 3, "file": "fig-3.png", "page": 12, "caption": "The overview...", "is_cover_candidate": true}
  ],
  "suggested_cover": "fig-3.png",
  "total_extracted": 5
}
```

**의존성**: `pymupdf` + `pypdf` (이미 설치됨). 미설치 시 `pip install pymupdf pypdf`

## Papers 주의사항

- 허위 수치/사실 생성 절대 금지 — 결과 숫자는 표/본문/캡션 확인된 수치만
- Figure 캡션: 원문 전체 작성, 생략/축약 금지
- **MDX Figure 캡션 i18n 규칙**: `ko.mdx`의 `<Figure caption="...">` 값은 반드시 `meta.json`의 해당 figure `caption_ko` 값을 사용. `en.mdx`는 `caption` (영문) 사용. `meta.json` 작성 시 `caption`과 `caption_ko` 두 필드를 동시에 생성할 것
- `cover.webp`가 없으면 가장 대표적인 figure를 cover.webp로 복사
- 이미지 리네임 후 본문 경로 미치환 상태로 커밋 금지

---

## Virtual Paper 경로 (가상 논문)

입력: `virtual [--group=<group>] [--from=<path>] <자연어 요청>`

원본 논문이 없는 **가상 논문(Research Proposal)** 포스팅. 기존 포스트/arXiv 논문을 종합해 새로운 연구 방향을 제안하거나, 사용자가 제공한 MD 파일을 논문 형식으로 포스팅한다.

```
/paper virtual --group=snu 촉각 잔차 학습 후속 논문
/paper virtual --group=korea --from=~/path/to/draft.md
/paper virtual 공개 가상 논문 제안
```

### Step V0) 공개 범위 확인

- `--group=<group>` 지정 → 해당 그룹 전용 비공개 포스트 (Supabase에만 저장)
  - 유효 그룹: `snu`, `korea` 등 (`access_groups` 테이블 기준)
  - `admin`은 모든 그룹의 콘텐츠를 볼 수 있음 (별도 설정 불필요)
- **`--group` 미지정 시**: 반드시 사용자에게 확인:
  > "그룹이 지정되지 않았습니다. 공개로 포스팅하시겠습니까? (공개/snu/korea 등)"
  - 공개 선택 → `visibility: "public"`, Git에 저장
  - 그룹 선택 → `visibility: "group"`, Supabase에만 저장

### Step V1) 콘텐츠 소스 결정

두 가지 모드:

**A) MD 파일 제공 (`--from=<path>`)**:
- 제공된 MD 파일을 읽어 Papers 포스트 양식으로 변환
- 기존 내용을 최대한 보존하되, 섹션 구조를 Papers 포스트 형식에 맞춤

**B) 자연어 요청 (기본)**:
- 사용자의 자연어 설명 + 기존 포스트/arXiv 논문 참조를 기반으로 AI가 논문 요약을 생성
- 기존 포스트 참조: `posts/index.json` + `posts/index-private.json`에서 관련 포스트 탐색
- arXiv 참조 시: WebFetch로 초록 수집, 핵심 내용 추출
- **반드시 실제 문헌 근거 기반으로 작성** — 허구의 실험 결과 수치 생성 금지
- 가설/예상 결과는 "Hypothetical:" 또는 "가설:" 접두어 사용

### Step V2) 슬러그 + ID 결정

- 슬러그: `YYMM-<short-name>` (가상의 출판 예상 시기 사용)
- ID: `posts/global-index.json`의 `next_public_id` 사용 (공개/비공개 모두 양수 ID)
- global-index.json에 엔트리 추가

### Step V3) Graph Analysis

Papers 경로 Step R3과 동일:
- `posts/index.json` + `posts/index-private.json` 로드
- taxonomy 배치, 관계 후보 생성

### Step V4) MDX 생성 (ko.mdx + en.mdx)

Papers 포스트와 **동일한 섹션 구조** (`docs/PAPERS_SUMMARY_RULES.md` 기준):

1. **TL;DR**
2. **가상 논문 고지** (TL;DR 바로 아래):
   ```
   > **이 포스트는 실제 논문이 아닌 가상 논문(Research Proposal)입니다.** 아래 내용은 연구 방향을 구체화하기 위해 최신 문헌 근거를 기반으로 구성한 연구 시나리오입니다.
   ```
3. **문제**
4. **핵심 아이디어**
5. **구체적 방법** (`<Collapsible>`)
6. **가설과 근거** (실제 논문의 "주요 결과" 대신 — 문헌 근거 기반 가설 제시)
7. **적용 시나리오** (선택)
8. **달성점과 한계점**

#### frontmatter 차이점

```yaml
---
locale: "ko"
title: "<한국어 제목>"
summary: "<2-3문장 요약>"
card_summary: "<카드용 짧은 요약>"
terrys_memo: ""
translation_of: null
translated_to:
  - "en"
references:
  - title: "<참조 논문 제목>"
    author: "<저자>"
    description: "<이 논문과의 관계/역할>"
    arxiv_url: "<있으면>"
    scholar_url: "<있으면>"
    project_url: "<있으면>"
    post_slug: "<사이트 내 포스트 있으면>"
    category: "foundational|recent"
---
```

- `terrys_memo`: **항상 빈 문자열** (AI 작성 금지)
- `references`: 인용한 실제 논문들의 상세 정보 (부록 '주요 참조 논문' 섹션에 표시됨)

### Step V5) meta.json 생성

Papers 포스트와 동일 구조 + Virtual Paper 전용 필드:

```json
{
  "post_id": "<slug>",
  "slug": "<slug>",
  "post_number": "<next_public_id>",
  "published_at": "<현재 ISO 날짜>",
  "updated_at": "<현재 ISO 날짜>",
  "status": "published",
  "content_type": "papers",
  "tags": ["Virtual Paper"],
  "display_tags": ["Virtual Paper (Research Proposal)"],
  "source_type": "Virtual Paper (Research Proposal)",
  "source_title": "<논문 영문 제목>",
  "source_author": "Anonymous",
  "source_authors_full": ["Anonymous"],
  "source_url": "",
  "source_date": "<published_at과 동일>",
  "source_project_url": "",
  "google_scholar_url": "",
  "first_author_scholar_url": "",
  "visibility": "group",
  "allowed_groups": ["<group>"],
  "domain": "<최상위 분야>",
  "subfields": [],
  "key_concepts": [],
  "methodology": [],
  "taxonomy_primary": "<taxonomy 경로>",
  "taxonomy_secondary": [],
  "relations": [],
  "ai_summary": {
    "one_liner": "<한 줄 요약>",
    "problem": "<문제>",
    "solution": "<해법>",
    "key_result": "<핵심 가설/기대 결과>",
    "limitations": ["실험 검증 없음 (Research Proposal)"]
  },
  "figures": [],
  "tables": [],
  "references": [],
  "terrys_memo": "",
  "featured": false
}
```

**주요 차이점**:
- `tags`: `["Virtual Paper"]`만 사용
- `display_tags`: `["Virtual Paper (Research Proposal)"]`
- `source_author`: `"Anonymous"`
- `source_url`, `google_scholar_url`, `first_author_scholar_url`: **빈 문자열** (클릭 불가하도록)
- `source_date`: 포스팅 날짜 (가상이므로)
- `limitations`에 항상 `"실험 검증 없음 (Research Proposal)"` 포함

### Step V6) 커버 이미지 생성

- `/image-gen` 스킬로 논문 주제에 맞는 개념적 커버 이미지 생성
- 프롬프트: 논문 제목/핵심 주제 반영, 16:9 비율, 텍스트 없이, 추상적 기술 일러스트
- `cover.webp` + `cover-thumb.webp` + `og.png` 생성

### Step V7) Supabase 업로드 (비공개인 경우)

**`visibility: "group"`일 때** (Git에 저장하지 않음):

```javascript
// private_content 테이블에 upsert
{
  slug: "<slug>",
  content_type: "papers",
  group_slug: "<group>",
  title_ko: "<한국어 제목>",
  title_en: "<영문 제목>",
  content_ko: "<ko.mdx 전체 내용>",
  content_en: "<en.mdx 전체 내용>",
  meta_json: <meta.json 객체>,
  cover_image_url: "<slug>/cover.webp",
  status: "published"
}
```

Supabase MCP의 `execute_sql`로 직접 INSERT/UPSERT:

```sql
INSERT INTO private_content (slug, content_type, group_slug, title_ko, title_en, content_ko, content_en, meta_json, cover_image_url, status)
VALUES ('<slug>', 'papers', '<group>', '<title_ko>', '<title_en>', '<ko_mdx>', '<en_mdx>', '<meta_json>::jsonb', '<slug>/cover.webp', 'published')
ON CONFLICT (slug) DO UPDATE SET
  content_ko = EXCLUDED.content_ko,
  content_en = EXCLUDED.content_en,
  title_ko = EXCLUDED.title_ko,
  title_en = EXCLUDED.title_en,
  meta_json = EXCLUDED.meta_json,
  cover_image_url = EXCLUDED.cover_image_url;
```

커버/썸네일/OG 이미지 → Supabase Storage `private-covers/{slug}/` 버킷 업로드
(업로드 스크립트: `/Users/terrytaewoongum/Codes/personal/terryum-ai/scripts/upload-private-content.mjs` 참조)

### Step V7-public) Git 저장 + CF 배포 (공개인 경우)

**`visibility: "public"`일 때**:
- Papers 경로 Step R9~R12와 동일 (빌드 스크립트 → CF 호환 빌드 검증 → 커밋 → 푸시)
- **Step R12.1 (`npx opennextjs-cloudflare deploy`)도 동일하게 실행** — git push만으로는 사이트에 반영되지 않는다

### Step V8) global-index 업데이트

- `posts/global-index.json`에 엔트리 추가 (gitignored)
- `next_public_id` 증가

### Step V9) Obsidian Vault Sync

```bash
node /Users/terrytaewoongum/Codes/personal/terryum-ai/scripts/sync-obsidian.mjs --slug=<slug>
```
실패 시 경고만 출력, 계속 진행

### Step V10) 예외 발생 시 레슨 기록

R13과 동일

---

## Virtual Paper 주의사항

- **허위 실험 결과 생성 금지**: 가설은 반드시 실제 문헌 근거 기반. "Hypothetical:" 접두어 사용
- **Google Scholar/저자 링크 비활성화**: 클릭해도 아무 곳으로도 가지 않도록 빈 문자열
- **가상 논문 고지 필수**: TL;DR 아래에 반드시 blockquote로 "이 포스트는 실제 논문이 아닌 가상 논문" 고지
- **Terry's memo AI 작성 금지**: 항상 빈 문자열
- **references 섹션 충실히 작성**: 인용한 모든 실제 논문의 title, author, description, URL, 사이트 내 post_slug 포함
- **figures 섹션**: 논문 내 주요 다이어그램/표를 직접 생성하여 포함 (GFM 마크다운 테이블 활용, 테이블 앞뒤 빈 줄 필수)

---

## Synthesis 경로 (다중 소스 종합)

입력: `synthesis URL1 [URL2 ...] [--tags=TAG1,TAG2] [--memo=메모] [--featured]`

여러 소스(GitHub, 블로그, 트윗, 논문 등)를 종합하여 **Papers-style paper 요약 포스트**를 생성한다. Virtual Paper와 달리 실제 콘텐츠 기반이므로 가상 논문 고지가 불필요하다.

```
/paper synthesis https://github.com/user/repo https://x.com/user/status/123
/paper synthesis https://blog.example.com/post https://arxiv.org/abs/2505.12345 --tags=AI,Agent
```

### Step SY1) 소스 수집

- 각 URL에 대해 WebFetch로 내용 수집
- **접근 불가 소스** (트윗 등 HTTP 402/403): 사용자에게 텍스트 복사 요청
- **사용자 제공 이미지**: 대화에서 공유된 이미지를 figure로 활용
- GitHub URL → README 기반 메타데이터 + 프로젝트 설명
- 트윗 → blockquote로 본문에 인용, 출처 링크 명시

### Step SY2) Slug + 메타데이터 결정

- **slug**: `YYMM-<short-name>` (주 소스의 날짜 기준)
- **source_type**: 주 소스 사이트명 (예: `"GitHub"`, `"Blog + Twitter"`)
- **source_url**: 가장 대표적인 URL (보통 첫 번째)
- **source_author**: 주 저자
- **google_scholar_url**: `null` (블로그 소스와 동일 취급)

### Step SY3) Graph Analysis

Papers 경로 Step R3과 동일

### Step SY4) MDX 생성

Papers 포스트와 **동일한 섹션 구조** (`docs/PAPERS_SUMMARY_RULES.md` 기준):
- TL;DR, 문제, 핵심 아이디어, 주요 결과, 달성점/한계점, Terry's Memo
- **가상 논문 고지 불필요** — 실제 콘텐츠 기반
- 트윗/소셜 미디어 인용은 blockquote로:
  ```
  > "인용 텍스트" — [@username](https://x.com/user/status/ID)
  ```
- 구체적 방법 섹션: 기술 디테일이 있을 때만 (Collapsible)

### Step SY5) meta.json 생성

Papers 포스트와 동일 구조. 차이점:
- `source_type`: 주 소스 사이트명
- `google_scholar_url`: `null`
- `source_project_url`: GitHub URL 등 (있으면)

### Step SY6) references (MDX frontmatter)

- **모든 소스를 references에 등록** — 빈 URL 금지
- 트윗: `project_url` 필드에 트윗 URL
- GitHub: `project_url` 필드에 repo URL
- arXiv: `arxiv_url` 필드
- 사이트 내 관련 포스트: `post_slug` 필드
- `description`: 각 소스가 이 포스트에서 어떤 맥락으로 사용되었는지

### Step SY7~SY12) 빌드 → 검증 → 커밋 → 푸시 → **CF 배포**

Papers 경로 Step R9~R13과 동일:
빌드 스크립트 → validate-post → `npx opennextjs-cloudflare build` (R11) → git commit + push → **`npx opennextjs-cloudflare deploy` (R12.1)** → KB 업데이트 → Obsidian sync → 예외 레슨 기록
