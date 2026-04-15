# terry-papers — Research Paper Posting Workspace

[한국어](#한국어) | **English**

> Content workspace for managing research paper posts, knowledge graphs, and references on [terry.artlab.ai](https://terry.artlab.ai).

This workspace is part of the [terry-artlab-homepage](https://github.com/terryum/terry-artlab-homepage) ecosystem. For full site documentation, architecture, and setup guide, see the [main repository README](https://github.com/terryum/terry-artlab-homepage#readme).

---

## What This Does

This workspace manages the **content side** of paper posting — summarizing arXiv papers, blog posts, and multi-source materials into bilingual (KO/EN) research summaries, then publishing them to [terry.artlab.ai/posts](https://terry.artlab.ai/ko/posts).

All content operations are performed via Claude Code skills:

| Command | What It Does | Example |
|---------|-------------|---------|
| `/post` | Summarize & publish a paper | `/post https://arxiv.org/abs/2303.04137` |
| `/post synthesis` | Combine multiple sources | `/post synthesis URL1 URL2` |
| `/paper-search` | Recommend papers via knowledge graph | `/paper-search` |
| `/share` | Share to social media | `/share #38` |
| `/del` | Delete a post safely | `/del #5` |

## How It Works

```
terry-papers/
├── posts/        → symlink to terry-artlab-homepage/posts/
├── scripts/      → symlink to terry-artlab-homepage/scripts/
├── .claude/skills/ → symlinks to homepage skills
└── .env.local    → R2 + Supabase credentials
```

This workspace symlinks into `terry-artlab-homepage` — editing posts here edits the same files. The benefit is **separation of concerns**: you can run `/post` here while another terminal runs homepage development in `terry-artlab-homepage`.

### Posting Pipeline

```
/post https://arxiv.org/abs/XXXX.XXXXX
  → Fetch metadata + figures from arXiv
  → Upload images to Cloudflare R2 CDN
  → Generate ko.mdx + en.mdx + meta.json
  → Update index.json + knowledge graph
  → Generate OG image
  → Sync bidirectional references
  → Git commit + push
  → Sync to Obsidian vault
```

## Setup

1. **Clone** this repo alongside `terry-artlab-homepage`:
   ```bash
   cd ~/Codes/personal
   git clone https://github.com/terryum/terry-papers.git
   ```

2. **Ensure symlinks point correctly** (they reference `terry-artlab-homepage` in the same parent directory)

3. **Copy `.env.local`** from `terry-artlab-homepage` (or create your own with the same keys)

4. **Use Claude Code** to start posting:
   ```bash
   cd terry-papers
   claude
   # Then: /post https://arxiv.org/abs/2303.04137
   ```

---

# 한국어

# terry-papers — 논문 포스팅 워크스페이스

> [terry.artlab.ai](https://terry.artlab.ai)의 논문 포스팅, 지식그래프, 참고문헌을 관리하는 콘텐츠 워크스페이스.

이 워크스페이스는 [terry-artlab-homepage](https://github.com/terryum/terry-artlab-homepage) 생태계의 일부입니다. 전체 사이트 문서, 아키텍처, 설정 가이드는 [메인 리포지토리 README](https://github.com/terryum/terry-artlab-homepage#readme)를 참고하세요.

## 하는 일

arXiv 논문, 블로그, 다중 소스를 한국어/영어 양국어 연구 요약으로 변환하여 [terry.artlab.ai/posts](https://terry.artlab.ai/ko/posts)에 발행합니다.

| 명령어 | 기능 | 예시 |
|--------|------|------|
| `/post` | 논문 요약 + 발행 | `/post https://arxiv.org/abs/2303.04137` |
| `/post synthesis` | 다중 소스 종합 | `/post synthesis URL1 URL2` |
| `/paper-search` | 지식그래프 기반 논문 추천 | `/paper-search` |
| `/share` | 소셜미디어 공유 | `/share #38` |
| `/del` | 포스트 안전 삭제 | `/del #5` |

## 사용법

```bash
cd ~/Codes/personal/terry-papers
claude
# /post https://arxiv.org/abs/XXXX.XXXXX
```

## License

MIT
