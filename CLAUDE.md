# CLAUDE.md — Papers 콘텐츠 워크스페이스

## 이 워크스페이스의 역할

논문 포스팅, 지식그래프, 참고문헌 관리에 집중하는 워크스페이스.
홈페이지 코드 수정, 인프라 변경은 `terry-artlab-homepage`에서 한다.

| 이곳에서 하는 것 | 이곳에서 하지 않는 것 |
|---|---|
| 논문 포스팅 (`/post`) | 홈페이지 코드/컴포넌트 수정 |
| 블로그/에세이 포스팅 | Next.js 라우트 변경 |
| 지식그래프 관리 (sync-papers, sync-references) | 빌드 스크립트 수정 |
| R2 이미지 업로드 | Supabase 스키마 변경 |
| 논문 추천 (`/paper-search`) | Vercel 배포 설정 |
| 포스트 삭제 (`/del`) | ACL/인증 시스템 수정 |
| 소셜 공유 (`/share`) | |
| Obsidian 동기화 | |

## 프로젝트 구조

```
terry-papers/
├── CLAUDE.md              ← 이 파일
├── posts/                 ← terry-artlab-homepage/posts/ 심링크
│   ├── papers/            ← 논문 MDX + meta.json
│   ├── essays/            ← 에세이
│   ├── memos/             ← 메모
│   └── index.json         ← 포스트 인덱스 (generate-index.mjs로 생성)
├── scripts/               ← terry-artlab-homepage/scripts/ 심링크
├── .claude/skills/        ← 스킬 심링크 (post, paper-search, del, share 등)
└── .env.local             ← 환경변수 (R2, Supabase)
```

## 핵심 명령어

```bash
# 논문 포스팅
/post https://arxiv.org/abs/XXXX.XXXXX

# 블로그 포스팅
/post --type=essays --from="~/Documents/Obsidian Vault/From Terry/Drafts/slug.md"

# 다중 소스 종합
/post synthesis URL1 URL2

# 논문 추천
/paper-search

# 소셜 공유
/share #N

# 포스트 삭제
/del #N
```

## Git 규칙

- 이 워크스페이스에서 terry-artlab-homepage에 push할 때 반드시 `git pull --rebase origin main` 후 push
- 콘텐츠 커밋과 코드 커밋을 분리

## 환경변수 (.env.local)

terry-artlab-homepage의 .env.local과 동일한 키 필요:
- `NEXT_PUBLIC_SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`
- `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME`, `R2_PUBLIC_URL`
- `NEXT_PUBLIC_R2_URL`

## 비공개 포스트 규칙

- `--visibility=group --group=snu` 포스트는 Supabase에만 저장 (Git에 흔적 없음)
- `posts/global-index.json`은 gitignore (비공개 ID 포함)
- 공개 포스트만 Git에 커밋

## 참고

- 홈페이지 개발: `terry-artlab-homepage`
- Obsidian 운영: `terry-obsidian`
- Survey 콘텐츠: `terry-surveys`
- 사이트: https://terry.artlab.ai
