# DeepSeek Harness 전수조사 분석 정리 (한국어)

> 작성일: 2026-09-29
> 대상 저장소: <https://github.com/deepseek-ai/deepseek-harness> (원본)
> 분석 대상 포크: <https://github.com/bmshin94/deepseek-harness>
> 분석 시점 버전: `0.1.6-alpha.2` / 라이선스: MIT

---

## 목차

1. [한 줄 요약](#1-한-줄-요약)
2. [저장소 실측 데이터](#2-저장소-실측-데이터)
3. [핵심 철학: Everything is a Plugin](#3-핵심-철학-everything-is-a-plugin)
4. [폴더 구조](#4-폴더-구조)
5. [에이전트 루프 동작 원리](#5-에이전트-루프-동작-원리)
6. [내장 도구(Tool) 목록](#6-내장-도구tool-목록)
7. [Profile / Bundle / Patch 실행 모델](#7-profile--bundle--patch-실행-모델)
8. [설치 및 사용법](#8-설치-및-사용법)
9. [플러그인 vs 스킬 vs MCP](#9-플러그인-vs-스킬-vs-mcp)
10. [API 토큰 필요 여부](#10-api-토큰-필요-여부)
11. [GitHub에서 유명한 이유](#11-github에서-유명한-이유)
12. [로컬 에이전트 구축 활용도](#12-로컬-에이전트-구축-활용도)
13. [React / PHP 로 만들 수 있는가](#13-react--php-로-만들-수-있는가)
14. [수익화 아이디어 10선](#14-수익화-아이디어-10선)
15. [주의사항 및 리스크](#15-주의사항-및-리스크)
16. [참고 링크](#16-참고-링크)

---

## 1. 한 줄 요약

DeepSeek Harness(`dsh`)는 완성된 챗봇이 아니라, **AI 코딩 에이전트를 직접 만들 수 있게 해주는 오픈소스 하네스(뼈대)** 이다.

- 모델은 "생각"만 할 수 있고, 파일 읽기/명령 실행/웹 검색 같은 손발은 못 한다.
- 그 손발을 달아주고 반복 실행을 관리하는 층이 하네스다.
- 비유: 모델 = 말, 하네스 = 마구(고삐·안장), 사용자 = 마부.

---

## 2. 저장소 실측 데이터

체크아웃된 소스를 직접 세어 확인한 수치다.

| 항목 | 수치 |
|---|---|
| npm 패키지 수 | **291개** (`packages/<group>/<pkg>` 구조) |
| TypeScript/TSX 코드 | **약 346,897 줄** (3,996개 파일) |
| 테스트 파일 | **1,428개** (테스트 디렉터리 297개) |
| 문서(.md) | **337개** (영어/중국어 병기) |
| 버전 | `0.1.6-alpha.2` (developer preview) |
| 라이선스 | MIT |
| 프론트엔드 | React 18.2 + Vite (client 55개 중 47개가 React 사용) |

### GitHub 공개 지표 (2026-09-29 확인)

| 항목 | 수치 |
|---|---|
| Stars | 약 **239,900** |
| Forks | 약 **28,800** |
| Watchers | 약 **1,000** |
| 공개 시점 | 2026년 8월 |
| 특이사항 | **GitHub 역사상 최단기간 10만 스타 돌파(약 48시간)** |

> 참고: 이 저장소 `CLAUDE.md` 상단의 "22만 개발자 주목" 문구는 자동 생성된 홍보 섹션이지만,
> 실제 스타 수(24만)와 비교하면 과장이 아니라 오히려 보수적인 수치였다.

### 패키지 그룹별 개수 (상위)

| 그룹 | 개수 | 역할 |
|---|---|---|
| `client/` | 55 | GUI 클라이언트 (React) |
| `session/` | 19 | 대화 영구 저장 (SQLite) |
| `util/` | 16 | 의존성 0 유틸 |
| `experimental/` | 16 | 실험 기능 (agent-team, browser-use 등) |
| `subagent/` | 10 | 서브 에이전트 위임 |
| `shell/` | 10 | bash / pwsh 실행 |
| `host/` | 8 | GUI 호스트 |
| `core/` | 8 | 에이전트/세션 API, 에이전트 루프 |
| `llm/` | 7 | 모델 제공자 어댑터 |
| `fs/` | 7 | 파일시스템 접근 |
| `api/` | 7 | 원격 BFF |

---

## 3. 핵심 철학: Everything is a Plugin

`docs/architecture.md` 원문:

> "There is no privileged core to patch"

- 기반 프레임워크는 **Cordis** (논문 `arXiv:2608.25512` 기반).
- 모델 어댑터, 도구 레지스트리, 세션 로그, **에이전트 루프 자체**까지 전부 플러그인이다.
- 따라서 어떤 구성요소든 설정(YAML)만으로 교체 가능하다.
- 모든 등록(registration)은 effect이며, 플러그인이 내려가면 자동으로 되감긴다.

비유: 일반 프레임워크가 "접착제로 붙은 레고 성"이라면, DSH는 "블록 한 박스"다. 바닥판조차 블록이다.

---

## 4. 폴더 구조

```
deepseek-harness/
├── apps/
│   ├── cli/            dsh 명령어 (모든 실행의 유일한 관문)
│   ├── web/            브라우저 UI (Vite + React)
│   ├── desktop/        Electron 데스크톱 앱
│   └── desktop-host/   데스크톱 내부 호스트
├── packages/           291개 플러그인 (본체)
├── python/             Python SDK (sdk + sdk-runtime)
├── native/             C++ 네이티브 애드온
├── vendor/             Cordis 프레임워크 벤더링 사본
├── benchmarks/         성능 게이트
├── docs/               문서 337개
├── snapshots/          녹화 세션 재생 테스트
├── scripts/            검증 게이트 / 코드 생성기
├── website/            VitePress 문서 사이트
├── .agents/            에이전트 작업 노트 및 스킬
└── .claude/skills/     Claude Code 전용 스킬
```

---

## 5. 에이전트 루프 동작 원리

```
사용자: "버그 고쳐줘"
  ↓
1. 모델에게 상황 + 사용 가능한 도구 목록 전달
  ↓
2. 모델이 도구 호출 결정 (예: read_file("app.js"))
  ↓
3. 하네스가 실제로 파일을 읽음  ← 모델은 못 하는 일
  ↓
4. 결과를 모델에게 반환
  ↓
5. 더 할 일이 있으면 1번으로 반복, 없으면 종료
```

- 용어: **step** = 모델 요청 1회 + 그 요청이 부른 도구들. **turn** = 0개 이상의 step.
- 담당 패키지: `packages/core/agent-loop`
- 핵심 원칙: **Model-visible ⟺ logged** (모델이 본 것은 반드시 세션 로그에 남는다)

---

## 6. 내장 도구(Tool) 목록

| 도구 패키지 | 기능 |
|---|---|
| `shell/tool-bash`, `tool-bash-persistent` | bash 실행 (일회성 / 상태 유지) |
| `shell/tool-pwsh`, `tool-pwsh-persistent` | PowerShell 실행 (Windows) |
| `fs/tool-fs`, `tool-fs-search` | 파일 읽기/쓰기/검색 |
| `fs/tool-str-replace-editor` | 문자열 치환 방식 코드 편집 |
| `web/tool-web` | 웹 검색 + 페이지 fetch |
| `lsp/tool-lsp` | LSP 연동 (정의 이동, 타입 정보) |
| `terminal/tool-terminal` | 영속 터미널 세션 |
| `todo/tool-todo` | 할 일 목록 관리 |
| `goal/tool-goal` | 세션 목표 관리 |
| `plan/` | 계획 수립 기록 |
| `subagent/tool-subagent`, `tool-subagent-control` | 서브 에이전트 위임/제어 |
| `jobs/tool-jobs` | 백그라운드 작업 |
| `workflow/tool-workflow`, `tool-ralph` | 워크플로 실행 |
| `skill/tool-skill` | 스킬 로딩 (Claude Skills 호환) |
| `interaction/tool-ask-user` | 모델이 사람에게 되묻기 |
| `session-query/tool-session-query` | 과거 세션 검색/조회 |
| `deliverables/tool-present` | 산출물 제시 |
| `extensions/tool-cordis` | 런타임 자기 수정 |
| `computer-use/`, `browser-use/` | 화면 조작, 브라우저 자동화 |
| `mcp/mcp-client` | 외부 MCP 서버 연결 |

### 그 밖의 주요 기능

- **Claude Code / Codex 브릿지**: `packages/hooks/hooks-claude-code`, `hooks-codex`
- **샌드박스**: `sandbox-local`, `sandbox-policy`, `sandbox-windows-acl`
- **GitHub 웹훅 자동 리뷰**: PR이 Draft → Ready 전환 시 리뷰 세션 자동 생성 (`docs/user/guide/github-review.md`)
- **스케줄러**: `packages/schedule/` — 예약 후속 작업
- **컨텍스트 압축**: `packages/compaction/` — 대화가 길어지면 자동 요약

---

## 7. Profile / Bundle / Patch 실행 모델

| 용어 | 의미 |
|---|---|
| **Bundle** | 플러그인 여러 개를 묶은 배포 단위 (세트 메뉴) |
| **Profile** | 번들들을 순서대로 쌓은 구성 (주문서) |
| **Patch** | 특정 행(row)을 id로 지정해 설정을 덮어쓰는 레이어 (요청사항 메모) |

적용 순서 (포토샵 레이어처럼 위가 우선):

```
5. --patch 오버레이 (실행 시 지정)
4. 홈 패치        $DSH_HOME/cordis.patch.yml
3. 프로필 패치     cordis.patch.yml
2. dsh-web-app    (브라우저 앱 번들)
1. dsh-base       (모델 + 도구 + 저장소 + 샌드박스 + 자격증명)
```

조립된 트리 확인:

```sh
dsh --profile web --dump-config
```

---

## 8. 설치 및 사용법

### 빠른 실행

```sh
npx @deepseek-ai/dsh web
```

기본값 `http://127.0.0.1:3080`, 로컬 실행 시 브라우저 자동 실행(`--no-open`으로 억제).

### 소스 빌드

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install
pnpm run build
pnpm dsh web
```

### 요구 환경

| 항목 | 요구사항 |
|---|---|
| Node.js | `^22.19.0 \|\| >=24.0.0` |
| 패키지 매니저 | `pnpm@11.7.0` |
| OS | macOS / Linux / Windows |

### 실행 모드 5가지

```sh
dsh web                      # 브라우저 UI
dsh headless "작업 내용"      # 서버 없이 1회 실행 후 종료
dsh --profile sdk            # JSON-RPC stdio 서버
dsh --profile sdk-minimal    # 최소 구성 SDK
dsh --profile acp            # ACP 자동화 전용 서버

dsh plugin --profile web add <패키지>   # 플러그인 설치
```

### 첫 설정

1. `dsh web` 실행
2. Settings → Models
3. DeepSeek 카드에 API 키 입력 후 저장
4. 모델 선택 후 대화 시작

키는 `$DSH_HOME/.credentials.yaml`에 저장되고, 설정 파일에는 참조만 남는다. UI는 저장 후 마스킹된 설명자만 받는다.

---

## 9. 플러그인 vs 스킬 vs MCP

DSH는 셋 중 하나가 아니라 **셋을 담는 런타임**이다.

| 개념 | 정체 | DSH와의 관계 |
|---|---|---|
| 플러그인 | 기능 모듈 | DSH가 플러그인 런타임 자체 |
| 스킬 | 모델에게 주는 지침서 | `packages/skill/`로 지원 (Claude Skills 호환) |
| MCP | AI-도구 연결 표준 프로토콜 | `packages/mcp/mcp-client`로 소비 |

비유: MCP = 콘센트 규격, 스킬 = 사용설명서, 플러그인 = 가전제품, DSH = 집과 전기 배선.

---

## 10. API 토큰 필요 여부

| 상황 | 토큰 필요 |
|---|---|
| 클라우드 모델 사용 | **필요** (`DEEPSEEK_API_KEY` 등) |
| 로컬 모델(Ollama / vLLM / LM Studio) | **불필요** (커스텀 프로바이더 등록, 더미 키) |
| 유닛 테스트 / 스냅샷 테스트 | 불필요 |
| e2e 테스트 | 키 없으면 자동 skip |

지원 프로바이더: `deepseek`(공식), `anthropic`, `openai`, `moonshotai`(Kimi), `zai`(GLM), 그리고 커스텀 게이트웨이.
지원 프로토콜: `openai-completions`, `openai-responses`, `anthropic-messages`.

---

## 11. GitHub에서 유명한 이유

1. **DeepSeek 브랜드 파워** — R1으로 업계를 뒤흔든 팀이 내부 하네스를 통째로 공개
2. **Claude Code / Codex의 오픈소스 대안** — 게다가 `hooks-claude-code` 브릿지로 이주 경로까지 제공
3. **아키텍처의 독창성과 완성도** — "코어 없음" 설계 + 논문 기반 + 34만 줄 / 1,428 테스트
4. **데모가 아닌 완제품** — Web UI + Electron + Python/TS SDK + 샌드박스 + GitHub 웹훅 리뷰
5. **중국 AI 오픈소스 생태계의 상징성** (Qwen, GLM, Kimi와 함께)
6. **타이밍** — "에이전트 인프라"가 진짜 프론티어로 인식되던 시점

---

## 12. 로컬 에이전트 구축 활용도

결론: **매우 높다.** 에이전트 개발에서 가장 번거로운 부분이 이미 구현되어 있다.

| 필요 기능 | 제공 여부 | 위치 |
|---|---|---|
| 로컬 모델 연결 | O | `llm-pi-ai` 커스텀 프로바이더 |
| 파일 읽기/쓰기 | O | `packages/fs/` |
| 터미널 실행 | O | `packages/shell/` |
| 프로세스 격리 | O | `packages/sandbox/` |
| 대화 영구 저장 | O | `packages/session/` (SQLite) |
| 컨텍스트 압축 | O | `packages/compaction/` |
| 도구 호출 파이프라인 | O | `core/tools` |
| 승인 프롬프트 | O | `interaction/tool-ask-user` |
| 서브 에이전트 | O | `packages/subagent/` |
| 백그라운드 작업 / 스케줄 | O | `packages/jobs/`, `packages/schedule/` |
| 웹 UI / 데스크톱 앱 | O | React 18 / Electron |
| 외부 도구 연결 | O | MCP 클라이언트 |
| 비즈니스 로직 | **직접 구현** | 플러그인으로 추가 |

### 완전 오프라인 구성

```
Ollama / vLLM (localhost)  ←→  DSH  ←→  localhost:3080 Web UI
```

사내 코드가 외부로 나가지 않는 폐쇄망 구성이 가능하다.

### 학습 로드맵 (제안)

1. `npx @deepseek-ai/dsh web`으로 먼저 감 잡기
2. `docs/cordis-primer.md` + `docs/cordis-tutorial/` 정독
3. 플러그인 1개 직접 제작 (`tool-todo` 구조 모방 추천)
4. Ollama 연결해 오프라인 구동
5. 커스텀 에이전트 완성

---

## 13. React / PHP 로 만들 수 있는가

### React — 가능. 이미 React로 만들어져 있음

- `packages/client/` 55개 중 **47개가 `react@^18.2.0` 사용**
- `apps/web/`은 Vite 기반 React 앱, 데스크톱도 동일 React 앱을 Electron으로 감쌈
- 따라서 UI 커스터마이징은 `packages/client/`에서 컴포넌트를 교체하면 된다.
- 주의: UI 문구는 반드시 다국어 사전과 `t`를 거쳐야 한다. 하드코딩은 `verify-client-ui-i18n` 게이트에서 거부된다.

### PHP — 본체 재작성은 불가, 연동은 충분히 가능

본체는 TypeScript 34만 줄 + Cordis + Node 네이티브 애드온이라 PHP 포팅은 현실적이지 않다. 대신 PHP가 DSH를 원격 조종하면 된다.

| 방법 | 설명 |
|---|---|
| HTTP API | `dsh web`이 제공하는 `/api` 호출 (`packages/api/`의 gateway·컨트롤러들) |
| CLI 호출 | `shell_exec('dsh headless "..."')` |
| JSON-RPC | `dsh --profile sdk`를 프로세스로 띄우고 stdio로 개행 구분 JSON-RPC 통신 |

권장 아키텍처:

```
Laravel / PHP 백엔드 (인증·결제·관리자·DB)
        ↓ HTTP / JSON-RPC
DSH (Node 프로세스, 격리 실행) — AI 에이전트 엔진
```

### 언어별 요약

| 언어 | 본체 개발 | 플러그인 제작 | 앱 연동 |
|---|---|---|---|
| TypeScript | O | O | 공식 SDK |
| React | O (이미 사용 중) | O (UI) | O |
| Python | 일부 (`ptc-runtime-python`) | 제한적 | 공식 SDK |
| PHP | X | X | HTTP / CLI / RPC |
| Go / Rust / Java | X | X | HTTP / CLI / RPC |

---

## 14. 수익화 아이디어 10선

> 전제: MIT 라이선스이므로 상업적 이용·수정·비공개 재배포 모두 허용된다.
> 단, 저작권 고지와 라이선스 사본을 포함해야 하고, **"DeepSeek" 상표는 사용하면 안 된다** (`BRAND_GUIDELINES.md` 참조).

| # | 아이디어 | 난이도 | 수익성 | 설명 |
|---|---|---|---|---|
| 1 | 산업 특화 AI 에이전트 SaaS | 중상 | 최상 | 병원·법무·제조·회계·이커머스 등 틈새 도메인. 도메인 전용 도구 플러그인 + 커스텀 시스템 프롬프트 |
| 2 | 온프레미스 보안 AI 코딩 어시스턴트 | 상 | 최상 | 금융·공공·방산·의료 등 망분리 조직. 구축비 + 유지보수 + 좌석 라이선스 |
| 3 | 유료 플러그인 마켓플레이스 | 하 | 중상 | README가 `dsh-plugin` 토픽을 공식 권장. 생태계 초기라 선점 효과 큼 |
| 4 | 관리형 호스팅 (Managed DSH Cloud) | 상 | 상 | 설치 난이도(Node 22+, pnpm, 291패키지)를 대신 해결. 경쟁 예상되므로 속도전 |
| 5 | 교육 · 강의 · 컨설팅 | 하 | 중상 | 24만 스타 대비 한국어 자료가 거의 없음. 가장 빠른 현금화 경로 |
| 6 | AI 코드리뷰 봇 (GitHub App) | 중 | 상 | `packages/webhook/` + `github-review.md`로 절반 구현됨. 결제·대시보드·멀티테넌시만 추가 |
| 7 | 버티컬 데스크톱 앱 | 중 | 중상 | `apps/desktop/` Electron 재활용. 개발자용이 아닌 일반인용 껍데기로 전환 |
| 8 | 로컬 프라이빗 AI 어플라이언스(하드웨어) | 최상 | 최상 | 미니PC + Ollama + DSH 사전 세팅 완제품. 중소기업 대상 |
| 9 | 에이전트 팀 오케스트레이션 SaaS | 상 | 상 | `experimental/agent-team` 활용. "AI 직원 팀 고용" 컨셉 |
| 10 | 템플릿 · 프리셋 판매 | 최하 | 중 | `packages/preset/agent-presets/` 구조 활용. YAML + 프롬프트만으로 제작 가능 |

### 단계별 전략 (제안)

| 기간 | 실행 항목 | 목표 |
|---|---|---|
| 1~2개월 | ⑤ 교육/콘텐츠, ⑩ 프리셋 판매 | 현금흐름 + 인지도 + 학습 |
| 3~6개월 | ③ 유료 플러그인, ⑥ 리뷰봇 MVP | 첫 SaaS 매출 |
| 6~12개월 | ② 온프레미스 SI, ① 산업 특화 SaaS | 본격 사업화 |

### 핵심 인사이트

1. **엔진이 아니라 껍데기를 판다** — DSH 자체는 무료다. 판매 대상은 도메인 지식·UX·서포트다.
2. **한국 시장은 온프레미스 수요가 크다** — 망분리 규제와 클라우드 거부감이 폐쇄망 AI 수요를 만든다.
3. **생태계 초기가 기회다** — 2026년 8월 공개. 선점 여지가 남아 있다.

---

## 15. 주의사항 및 리스크

### 공식 안전 고지 (`SAFETY.md`)

> "It has not undergone a security audit and must not be treated as secure or production-ready."

- 모델이 생성한 코드와 명령을 실제로 실행하고, 서드파티 플러그인을 로드하며, 네트워크·프로세스·자격증명·파일에 접근한다.
- 샌드박스와 승인 프롬프트는 위험을 줄이지만 격리를 보장하지 않는다.
- 권장: 최소 권한 실행, 일회용 VM/컨테이너 사용, 백업 유지, 플러그인·명령 사전 검토.

### 기술적 리스크

| 리스크 | 대응 |
|---|---|
| 알파 버전, 호환성 깨지는 변경 예고됨 | 버전 고정, 자체 추상화 레이어 |
| 러닝커브 (Cordis 선행 학습, `AGENTS.md` 17KB) | 단계적 학습 로드맵 |
| 무거운 설치 (291 패키지) | `sdk-minimal` 프로필 고려 |
| 로컬 소형 모델의 도구 호출 품질 | 도구 호출이 안정적인 모델 선택 |
| 사업 리스크 (DeepSeek 직접 상용화, 경쟁 진입, 상표권) | 틈새 도메인 차별화, 자체 브랜드 사용 |

---

## 16. 참고 링크

### 저장소

- 원본: <https://github.com/deepseek-ai/deepseek-harness>
- 분석 대상 포크: <https://github.com/bmshin94/deepseek-harness>
- 공식 문서 사이트: <https://deepseek-harness.github.io/deepseek-harness/>
- GitHub Discussions: <https://github.com/deepseek-ai/deepseek-harness/discussions>
- 플러그인 토픽: <https://github.com/topics/dsh-plugin>
- Cordis 프레임워크: <https://github.com/cordiverse/cordis>
- 설계 논문: <https://arxiv.org/abs/2608.25512>

### 저장소 내부 문서

- `README.md` — 실행 방법
- `SAFETY.md` — 안전 고지 (실행 전 필독)
- `AGENTS.md` / `CLAUDE.md` — 기여 규칙 전체
- `docs/architecture.md` — 아키텍처 (packages 수정 전 필독)
- `docs/cordis-primer.md`, `docs/cordis-tutorial/` — Cordis 입문
- `docs/user/guide/providers.md` — 모델 제공자 설정
- `docs/user/guide/github-review.md` — GitHub 웹훅 리뷰 오버레이
- `docs/user/guide/python-sdk.md` — Python SDK
- `docs/testing.md` — 테스트 정책
- `apps/cli/README.md` — CLI 전체 레퍼런스

### 외부 참고 기사

- <https://pasqualepillitteri.it/en/news/11573/deepseek-harness-fastest-github-stars-record>
- <https://agentnativedev.medium.com/deepseek-harness-hit-100k-stars-in-2-days-real-frontier-is-agent-infrastructure-ab263905b137>
- <https://daily.dev/posts/deepseek-releases-open-source-agent-harness-with-150-000-github-stars-in-days-wi2yf4gtc>
