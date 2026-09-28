# book-to-skill 전수조사 · 활용 · 수익화 분석 (한국어)

> 작성: Claude Code 세션 (페르소나: 카리나 ✨)
> 작성일: 2026-09-28
> 대상 커밋: `c4c670e` (branch `claude/blissful-gauss-ad4cx9`)

## 📌 관련 GitHub 주소

| 구분 | URL |
|---|---|
| **이 레포 (포크)** | https://github.com/bmshin94/book-to-skill |
| **원본(업스트림)** | https://github.com/virgiliojr94/book-to-skill |
| 문서 사이트 | https://booktoskill.is-a.dev/ |
| Agent Skills 표준 | https://github.com/agentskills/agentskills |
| skills CLI | https://skills.sh |
| 사용 사례 모음 | https://github.com/virgiliojr94/book-to-skill-use-cases |
| 스폰서 | https://github.com/sponsors/virgiliojr94 |
| Trendshift | https://trendshift.io/repositories/27038 |
| 참조 논문 | https://arxiv.org/abs/2607.17598 (Progressive Disclosure) |
| 이 분석 세션 | https://claude.ai/code/session_017dY4noXuaihrsuLq7d3D5Z |

---

## 1. 정체 — 한 줄 요약

> **내가 가진 책·문서(PDF/EPUB/DOCX/HTML/RTF/MOBI/TXT/MD…)를 AI 에이전트가
> "필요할 때만" 꺼내 읽는 Agent Skill 폴더로 변환해주는 컨버터.**

- 라이선스: MIT (버전 `1.4.0`)
- `bmshin94/book-to-skill`는 원본 포크 + `CLAUDE.md` 페르소나 가이드만 추가한 상태
  - `c4c670e Merge pull request #1 from bmshin94/feat/claude-guide`
  - `63efeb1 docs: appended CLAUDE.md persona guide`

## 2. 레포 구조 전수조사 (추적 파일 111개 / Python 약 4,277줄)

| 위치 | 정체 | 핵심 |
|---|---|---|
| `SKILL.md` (817줄, 47KB) | **프로젝트의 심장** | 에이전트가 따라 실행하는 Step 0~11 명세서. 코드가 아니라 "작업 지시서" |
| `book_to_skill/` | 파이썬 추출 엔진 | `utils.py`(1,284줄: CLI·멀티소스·챕터 감지), `parsers/`(pdf/epub/docx/html/rtf/calibre/text), `sanitize.py`, `dependencies.py`, `pdf_inspector_integration.py` |
| `scripts/extract.py` | 얇은 진입점 shim (26줄) | 구버전 호출 호환 |
| `tools/` | 측정·검증 | `discovery_tax.py`(토큰 절감 실측), `validate_skill.py`(claude/copilot/amp/hermes 렌즈), `scan_generated_skill.py`(프롬프트 인젝션 스캔) |
| `tools/evals/` | 평가 하네스 | manifest / paper_flat / replay / score |
| `tests/` | 42개 테스트 | 경로 탈출, XXE, 제로폭 유니코드, CJK 토큰, 다국어 챕터 감지 |
| `docs/` | mkdocs-material 사이트 | how-it-works / usage / install / faq / performance / architecture |
| `docs/research/progressive-disclosure-evals.md` | 논문 검증 원장 | "게이트 통과 전 프로덕션 반영 금지" |
| `AGENTS.md` | 에이전트 실행 계약서 | "측정하라, 주장하지 마라", 저작권 본문 커밋 금지 |
| `.github/workflows/` | CI | ci / codeql / deploy-docs (+ Bandit, Zizmor, dependabot) |

## 3. 동작 구조 — 2단 분리 설계

```
[1단] 결정적 Python 추출기 (항상 같은 결과, 테스트 가능)
문서 → scripts/extract.py → <tempdir>/book_skill_work-<pid>/
                            ├── full_text.txt   (소스 마커 포함 병합 텍스트)
                            └── metadata.json   (페이지/단어/토큰/챕터/ToC)
                                   ↓
[2단] 생성기 = 에이전트가 SKILL.md 명세를 따름 (지능 필요)
Step 1.5 기술서/일반서 질문 → BOOK_TYPE
Step 2   추출 실행
Step 2.5 사전 비용 견적 후 사용자 확인
Step 2.6 대형 책은 REPL 방식(grep/sed)으로 부분 접근
Step 3   구조 분석 (제목/저자/챕터/ToC)
Step 4   목적 질문 → DEPTH (reference | study)
Step 7   챕터별 요약 (예산 = BOOK_TYPE × DEPTH)
Step 8   glossary / patterns / cheatsheet
Step 9   마스터 SKILL.md + 챕터·토픽 색인
Step 9.5 생성물 보안 스캔
Step 10  정리 및 리포트
Step 11  (선택) GitHub 퍼블리시 — 기본 --private
                                   ↓
<SKILLS_HOME>/<slug>/
├── SKILL.md      ~4,000 토큰   ← 항상 로드
├── chapters/*.md 각 ~1,000 토큰 ← 물어볼 때만 로드 (핵심!)
├── glossary.md   ~1,500 토큰
├── patterns.md   ~2,000 토큰
└── cheatsheet.md ~1,000 토큰   ← 의사결정 표
```

### 실제 동작 검증 (이 세션에서 실행)

합성 3챕터 마크다운으로 `scripts/extract.py` 실행 결과:

```
chapters: 3 (numeric)          ← 챕터 자동 감지 성공
Words: 61 / estimated_tokens: 81
metadata.json: chapters_method="numeric", chapter_headings_sample 3건 정확
Workdir -> .../work   (PID별 격리로 동시 실행 충돌 방지)
```

`python3 scripts/extract.py --check` 결과: 이 컨테이너엔 선택 의존성 전부 미설치.
대부분 stdlib 폴백 존재, **MOBI/AZW만 Calibre 필수(폴백 없음)**.

## 4. 핵심 개념 — Discovery Loop Tax (탐색 루프 세금)

질문 1개에 컨텍스트로 들어가는 토큰 수 (tiktoken cl100k_base 실측):

| 책 | 통째로 붓기 | 에이전트 직접 탐색 | book-to-skill | 절감 |
|---|---:|---:|---:|:--:|
| Think Python 2 | 119,264 | 12,152 | ~5,000 | 24× / 2.4× |
| Working Backwards | 175,253 | 33,444 | ~5,000 | 35× / 6.7× |
| AI Engineering | 256,287 | 77,866 | ~5,000 | 51× / 15.6× |

- 통째로 붓기 비용은 **매 턴 재청구**된다는 점이 핵심.
- 변환 1회 비용은 약 **$1/권** (Sonnet 4.5, $3/$15 per MTok 기준).
- 재현: `python3 tools/discovery_tax.py --full-text <full_text.txt> --target-chapter 5`

## 5. 설계 철학 — "요약이 아니라 구조"

| 요약 (지향하지 않음) | 구조 (지향) |
|---|---|
| "저자는 5 Whys를 설명한다" | "**5 Whys**: 근본원인 추적. 근본원인이 조직적일 때 사용. 3회 이하 중단은 안티패턴" |

SKILL.md Quality Rules 8개:
1. 요약이 아니라 구조 추출
2. 저자의 정확한 명명 보존 ("5 Whys" ≠ "왜를 여러 번")
3. 완전성보다 밀도
4. 실무자 어조 ("Use X when Y")
5. SKILL.md 앞쪽 우선 배치 (압축은 뒤부터 잘림)
6. 챕터는 온디맨드
7. **원문 절대 복사 금지** (저작권 안전장치)
8. 토픽 색인이 라우팅의 핵심

## 6. 보안 설계 (문서 → 컨텍스트 공급망 하드닝)

- `sanitize.py` — 제로폭(`U+200B/200C/200D/2060/FEFF`) + 유니코드 태그블록(`U+E0000–E007F`) 제거.
  보이지 않는 문서 내 명령 주입 차단. 제거 개수 보고, 가시 콘텐츠 0이면 소스 거부.
- `parsers/docx.py` — DTD/엔티티 선언 시 파싱 거부 (XXE / Billion Laughs).
- 서브프로세스 인자 주입 방지 — 경로 절대화 후 `pdftotext`/`pdfinfo`/`ebook-convert` 전달.
- `tools/scan_generated_skill.py` — 생성된 스킬 전체를 인젝션/권한확대/유출 패턴 스캔 (매칭 텍스트는 노출하지 않음).
- CI — CodeQL, Bandit(HIGH 게이트), Zizmor, 의존성 CVE 리뷰.

## 7. 오빠에게 주는 실익

1. 산 책 되살리기 — `/ddia replication` 한 줄로 해당 챕터 근거 답변
2. 사내 문서 통합 — `docs/` 폴더 전체를 스킬 하나로 (ADR, 런북, 온보딩)
3. 브랜드/디자인 시스템 — 60페이지 브랜드북을 팀이 질의하는 스킬로
4. 논문 클러스터 — 여러 논문 + 내 노트를 통합, 새 자료는 fold-in(Mode 4)
5. API 비용 절감 — 매 세션 PDF 붓기 → 1회 변환
6. 할루시네이션 방지 — 내 실제 사본에 grounded

---

## 8. Q&A

### Q. 설치 및 사용법

```bash
# 권장: skills CLI (모든 호스트)
npx skills add virgiliojr94/book-to-skill

# Claude Code 수동 (개인/글로벌)
git clone https://github.com/virgiliojr94/book-to-skill.git ~/.claude/skills/book-to-skill
# 프로젝트 로컬 (팀과 git 공유)
git clone https://github.com/virgiliojr94/book-to-skill.git .claude/skills/book-to-skill

# 이 포크를 쓰려면
git clone https://github.com/bmshin94/book-to-skill.git ~/.claude/skills/book-to-skill
```

호스트별 스킬 루트:

| 호스트 | 경로 |
|---|---|
| Claude Code | `~/.claude/skills/` |
| GitHub Copilot CLI | `~/.copilot/skills/` (설치 후 `/skills reload`) |
| Amp / Codex / 크로스에이전트 | `~/.agents/skills/` |
| Hermes Agent | `${HERMES_HOME:-~/.hermes}/skills/<category>/` |

추출기만 CLI로 (스킬 등록 아님):

```bash
pip install "book-to-skill[pdf,epub,docx] @ git+https://github.com/virgiliojr94/book-to-skill.git"
book-to-skill --check
book-to-skill ~/book.pdf --mode text
```

사용:

```bash
/book-to-skill ~/books/ddia.pdf
/book-to-skill ~/papers/p1.pdf ~/notes/memo.txt unified-research   # 통합
/book-to-skill ~/workspace/project-docs/ project-knowledge         # 폴더
/book-to-skill "~/books/*.epub" my-library                         # glob
/book-to-skill ~/new-paper.pdf ~/.claude/skills/project-knowledge  # fold-in
```

생성된 스킬 사용:

```bash
/designing-data-intensive-apps
/designing-data-intensive-apps replication
/designing-data-intensive-apps ch05
```

형식별 추출기:

| 형식 | 최고 품질 | 폴백 |
|---|---|---|
| PDF(일반) | `pdftotext`(poppler) | pypdf → pdfminer.six |
| PDF(기술서) | `docling` (표·코드 보존, ~1.5s/page) | text 체인 |
| EPUB | `ebooklib`+`beautifulsoup4` | stdlib zipfile |
| DOCX | `python-docx` | stdlib ZIP/XML |
| HTML | `trafilatura`(무거움, 17패키지) | bs4 → html.parser |
| RTF | `striprtf` | regex |
| MOBI/AZW/AZW3 | Calibre `ebook-convert` | **없음 (필수)** |
| TXT/MD/RST/ADOC | 내장 | — |

스캔 PDF는 텍스트 레이어가 없어 즉시 중단됨 → `ocrmypdf input.pdf output.pdf` 선행 필요.

### Q. 플러그인? 스킬? MCP?

**스킬(Agent Skill)이고, 정확히는 "스킬을 만드는 스킬"(meta-skill).**

| 구분 | 여부 | 근거 |
|---|---|---|
| Skill | ✅ | 루트 `SKILL.md` + frontmatter(`name`, `description`), Agent Skills 오픈 표준 |
| MCP | ❌ | MCP 서버/JSON-RPC 프로세스 없음 |
| Plugin | ❌ | `.claude-plugin/plugin.json` 구조 아님 |
| CLI | 🔶 절반 | `pip install` 시 추출기만 CLI로 제공 (스킬 등록 안 됨) |

MCP가 아닌 이유: 책 읽기는 상시 연결 서비스가 아니라 필요할 때 꺼내는 참조 자료이고,
마크다운 파일 하나로 4개 호스트(Claude/Copilot/Amp/Hermes)를 동시 지원할 수 있음.

### Q. API 토큰이 필요해?

| 층 | 필요 여부 |
|---|---|
| 추출기 (Python) | ❌ 완전 불필요. 100% 로컬, 네트워크 호출 없음 |
| 생성기 (분석/작성) | ⚠️ 간접 필요 — 이미 쓰는 에이전트가 처리, **별도 키 불필요** |
| GitHub 퍼블리시 (Step 11, 선택) | ⚠️ `gh` CLI 인증 |

비용: 도구 자체 $0(MIT) / 변환 약 $1/권(1회성) / 변환 후 질문당 ~5,000토큰.
프라이버시: 추출·분석은 로컬. 단 클라우드 모델을 쓰면 그 텍스트는 일반 프롬프트와 동일하게 제공자 약관 적용.

### Q. 왜 GitHub에서 유명할까?

레포 내 증거: Trendshift 배지 2개(repo #27038), PR 번호 #228까지, README 3개 언어(en/ru/zh-CN),
GitHub Sponsors + BACKERS.md, 자체 문서 사이트, v1.4.0 릴리스 + git-cliff 자동 CHANGELOG.

분석:
1. **타이밍** — Agent Skills 표준 확산기에 "스킬을 만드는 스킬"을 제시
2. **공감되는 통증** — "책 사서 한 번 읽고 3개월 뒤 다 잊음"
3. **주장 대신 측정** — tiktoken 실측 + 재현 명령어 공개, `discovery_tax.py`가 자기 한계("모델이지 실측 아님")를 먼저 고백
4. **벤더 중립** — Claude/Copilot/Amp/Codex/Hermes 모두 지원, `validate_skill.py --lens`로 호스트별 검증
5. **저작권 정면 처리** — 책 내용 무포함, Quality Rule #7, 퍼블리시 기본 private("public" 단어만 공개 허용)
6. **엔지니어링 품질** — 42개 테스트(보안/다국어/CJK), 논문 검증 원장, 동시 실행 충돌 같은 실패 모드까지 설계

### Q. 로컬 에이전트 구축에 도움이 될까?

도움 됨. 다만 "쓰는 것"보다 "설계를 배우는 것"의 가치가 더 큼.

바로 재사용 가능:
- `book_to_skill/parsers/` 7종 파서 (MIT)
- `sanitize.py` + `scan_generated_skill.py` (문서 다루는 모든 에이전트에 필수)
- `discovery_tax.py` (내 파이프라인 비용 정량화)
- `validate_skill.py` (멀티호스트 규칙 검사)

배울 설계 패턴 3개:
1. **Progressive Disclosure** — 항상 로드는 색인만, 본문은 필요할 때
2. **결정적/확률적 분리** — 파싱·감지는 Python, 의미 추출만 LLM
3. **실행 계약서** — 에이전트에 코드가 아니라 절차서(Step + 검증 게이트 + Quality Rules)를 준다

한계: 이 자체는 에이전트 프레임워크가 아니며 호스트가 필요. 로컬 LLM으로 돌리려면 생성 단계를
직접 구현해야 함. 스캔 PDF/이미지 내 텍스트 불가. 챕터 자동 감지는 명시적 헤딩 필요
(Pro Git, Moby-Dick은 자동 분절 실패).

### Q. React나 PHP로 만들 수 있어?

가능. 단 파트별 전략이 다름.

| 파트 | 이식성 |
|---|---|
| 문서 파싱 | Python 유리, JS도 충분히 가능 |
| 챕터 감지 / 토큰 계산 / 새니타이즈 | 순수 로직 → 100% 이식 가능 |
| 의미 추출 (LLM 호출) | 언어 무관 |
| UI / 스킬 관리 / 미리보기 | **React가 Python보다 우월** |

**React (Next.js) 스택 — 추천**

```
Next.js App Router
├── 업로드: react-dropzone
├── 파싱: pdf-parse / unpdf / pdfjs-dist, epub2+cheerio+jszip,
│         mammoth.js(DOCX), @mozilla/readability+jsdom(HTML), rtf-parser
├── 토큰: js-tiktoken / gpt-tokenizer
├── 챕터 감지: Python 정규식 로직 이식
├── 생성: @anthropic-ai/sdk 스트리밍
├── 미리보기: Monaco Editor + 트리 뷰
└── 산출: JSZip으로 스킬 폴더 zip
```

장점: 실시간 진행률·챕터 트리·비용 견적 시각화, 인라인 편집 후 부분 재생성,
팀 공유, 설치 불필요, WASM 클라이언트 파싱으로 프라이버시 확보.
약점: `docling` 급 기술서 표/코드 추출 대안이 JS에 없음 → Python 마이크로서비스 하이브리드.

**PHP 스택 — 가능하나 제한적**

`smalot/pdfparser`, `phpoffice/phpword`, `kiwilan/php-ebook`, `ZipArchive`,
`symfony/dom-crawler`, `yethee/tiktoken-php`, Guzzle → Anthropic API (SSE).
유리한 경우: 기존 Laravel/WordPress 인프라 보유, **WordPress 플러그인화**,
공유 호스팅 배포. 약점: 파서 생태계가 얇고 docling 대안 없음, 긴 작업은 큐 필수.

**권장 구조**

```
Next.js (React) 프론트 + API
   └─(기술서 PDF만)→ Python 마이크로서비스(docling, FastAPI)
   └─────────────→ Anthropic API (생성)
   └─────────────→ 결과: 스킬 폴더 zip / GitHub 푸시
```

원본 엔진을 재구현하지 말고 그대로 재사용(MIT)하고, 그 위에 React UI + 한국어 특화를 얹는 편이
투입 대비 가치가 큼.

---

## 9. 수익화 아이디어 상세

> 전제: 원본은 MIT라 상업적 이용/수정/재배포 자유(저작권 고지 유지).
> 그러나 **처리 대상 문서의 저작권은 별개**.
> 수익화는 ① 고객이 소유한 문서 ② 퍼블릭 도메인 ③ 명시적 라이선스 자료로 한정.

| # | 아이디어 | 난이도 | 수익 잠재력 |
|---|---|---|---|
| 1 | 사내 문서 → 에이전트 스킬 구축 컨설팅 (B2B) | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| 2 | 웹 SaaS (Book to Skill Cloud) | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 3 | 한국어/CJK 특화 포크 (+HWP) | ⭐⭐ | ⭐⭐⭐⭐ |
| 4 | OCR 파이프라인 애드온 | ⭐⭐⭐ | ⭐⭐⭐ |
| 5 | 스킬 마켓플레이스 (저자 직판) | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| 6 | MCP 서버 래퍼 | ⭐⭐ | ⭐⭐⭐ |

### 1위. 사내 문서 → 에이전트 스킬 구축 서비스 (B2B)

- 고객 자기 문서이므로 저작권 리스크 없음
- 경쟁(RAG 구축 업체) 대비 차별점: "구조 추출 + 토큰 24~51배 절감"을 **실측 수치로** 제시

| 티어 | 내용 | 가격(예시) |
|---|---|---|
| 진단 | 문서 감사 + 토큰 비용 리포트(`discovery_tax.py` 실측) | 200~500만원 |
| 구축 | 문서 100~500건 → 스킬팩 + 팀 배포 | 1,000~3,000만원 |
| 운영 | 월간 fold-in 업데이트 + 품질 측정 | 월 100~300만원 |
| 교육 | 팀 자체 운영 워크샵 | 회당 300만원 |

타겟: 런북 많은 인프라팀, ADR 쌓인 플랫폼팀, 온보딩 지옥 스타트업, 금융/의료 규정 문서, 브랜드북 보유 에이전시.
세일즈 무기: 미팅에서 `tools/discovery_tax.py`를 고객 문서로 라이브 실행해 절감액을 숫자로 제시.

### 2위. 웹 SaaS — Book to Skill Cloud

| 원본 CLI | SaaS 차별점 |
|---|---|
| 터미널 설치 | 브라우저 드래그&드롭 |
| 텍스트 로그 | 실시간 진행률 + 챕터 트리 |
| 결과 수정 = 재실행 | 인라인 편집 + 부분 재생성 |
| 개인용 | 팀 워크스페이스 + 버전 관리 |
| `gh` CLI 필요 | 원클릭 배포 |
| 영어 중심 | 한국어 최적화 |

| 플랜 | 가격 | 내용 |
|---|---|---|
| Free | ₩0 | 월 1권, 50p 이하, 워터마크 |
| Pro | ₩19,000/월 | 월 10권, 무제한 페이지, 기술서 모드(docling) |
| Team | ₩79,000/월 | 5석, 공유 워크스페이스, fold-in 자동화 |
| Enterprise | 문의 | 온프레미스, SSO, 감사 로그 |
| 종량제 | 권당 ₩3,000~5,000 | 라이트 사용자 |

핵심 무기 = **BYOK + 클라이언트 사이드 파싱**: 브라우저 WASM 파싱으로 원본 파일을 서버에 올리지 않고,
사용자 자기 API 키로 모델 비용을 부담시켜 원가를 호스팅+docling 컨테이너로 축소(마진 90%+),
동시에 "파일을 저장하지 않는다"로 저작권 논란을 차단.

리스크: 원본이 무료 OSS라 유료화 근거(UI/팀/한국어/무설치)가 필요. 업로드 저작권은 ToS + 파일 미저장 + 기본 private로 대응.

### 3위. 한국어/CJK 특화 포크

확인된 빈틈:
- `config.py`에 `CJK_CHARS_PER_TOKEN = 1.5`로 CJK 토큰 추정은 존재
- 챕터 감지 지원 언어: Chapter / Capítulo / Κεφάλαιο(그리스어) / அத்தியாயம்(타밀어) / అధ్యాయము(텔루구어)
- **한국어 "제1장 / 1장 / 제 1 편"은 미지원** → 기회

개발 항목: 한국어 챕터 정규식, 한글 토큰 계수 보정, 국내 전자책(리디/예스24/알라딘 DRM-free EPUB),
**HWP/HWPX 파서(킬러 기능 — 관공서·대학·기업 문서)**, 한국어 스킬 생성(glossary/patterns/cheatsheet).

수익: OSS 기여 → 신뢰 → 컨설팅 유입 / HWP 파서 유료 라이선스 / 국내 SaaS / 공공 SI 입찰.
전략: **한국어 챕터 감지 PR을 업스트림에 먼저 기여**(그리스어·타밀어·텔루구어 PR이 머지된 전례 있음).

### 4위. OCR 파이프라인 애드온

원본이 명시적으로 거부한 영역(무거운 의존성·속도) = 빈 시장.
`스캔 PDF → 전처리 → OCR → 레이아웃 복원 → book-to-skill`.
기술: ocrmypdf, PaddleOCR(한국어 강함), Surya, Tesseract.
타겟: 절판 도서, 대학 스캔 강의자료, 기록물 디지털화. 모델: 페이지 종량제(₩50~100/p) 또는 SaaS 애드온.

### 5위. 스킬 마켓플레이스 (저작권 주의)

| 허용 | 금지 |
|---|---|
| 퍼블릭 도메인(구텐베르크) | 시중 판매 기술서 |
| 오픈 라이선스 문서(RFC, MDN, 리눅스 문서) | 출판사 저작물 |
| OSS 공식 문서 | 유료 강의 교재 |
| **저자가 직접 올린 자기 책** | 라이선스 불명 논문 |

현실적 모델 = **저자 직판 플랫폼**: 저자가 공식 스킬 등록 → 개발자 구매(₩5,000~15,000) → 수수료 20~30%.
저작권자와 협력하는 구조라 리스크가 없음.

### 6위. MCP 서버 래퍼

`convert_document`, `query_skill`, `list_skills`, `fold_in` 툴 노출.
Agent Skills 미지원 호스트/IDE 커버. 난이도 낮음.
수익: 호스팅형 유료 MCP 엔드포인트, 기업 사내 MCP 게이트웨이 구축. 인지도 확보용으로 활용 후 1위 컨설팅으로 전환.

### 권장 실행 순서

```
0~1개월  한국어 챕터 감지 PR 업스트림 기여 → 신뢰 + 코드베이스 이해
1~2개월  HWP/HWPX 파서 추가 + React 웹 UI 프로토타입(원본 엔진 재사용)
2~3개월  "사내 문서를 AI 스킬로" 콘텐츠 마케팅 → 리드 수집
3~6개월  B2B 컨설팅 1~2건 수주 → 첫 현금흐름 + 실사례 확보
6개월~   사례 기반 SaaS 출시 (BYOK, 프라이버시 우선)
```

이유: 컨설팅이 현금흐름을 먼저 만들고, 거기서 얻은 실제 문제가 SaaS 기능이 되며, OSS 기여가 무료 마케팅이 됨.

### 핵심 인사이트 3개

1. **툴이 아니라 결과를 판다** — `discovery_tax.py`의 숫자가 최고의 세일즈 도구
2. **원본과 경쟁하지 말고 빈틈을 채운다** — 한국어, HWP, OCR, UI
3. **저작권을 회피하지 말고 설계에 넣는다** — 원본이 신뢰를 얻은 이유

---

## 10. 검증 근거 (이 세션에서 실제 확인한 것)

| 항목 | 방법 | 결과 |
|---|---|---|
| 레포 규모 | `git ls-files` / `wc -l` | 추적 파일 111개, Python 4,277줄 |
| 포크 관계 | `git remote -v`, `git log` | `bmshin94/book-to-skill`, 원본 + CLAUDE.md 페르소나 |
| 의존성 상태 | `python3 scripts/extract.py --check` | 선택 의존성 전부 미설치, 폴백 가능 / Calibre만 필수 |
| 추출기 동작 | 합성 3챕터 MD로 실행 | 챕터 3개 numeric 감지, metadata.json 정상 생성 |
| 테스트 실행 | `python3 -m pytest -q` | **미실행** — 이 컨테이너에 pytest 미설치 |

> 주의: 위 표의 마지막 행처럼, 실행하지 못한 검증은 실행한 것처럼 기술하지 않음.
> 성능 수치(24~51×, $1/권)는 레포 `docs/performance.md`에 기재된 업스트림 실측값을 인용한 것이며,
> 이 세션에서 재측정하지는 않았음.
