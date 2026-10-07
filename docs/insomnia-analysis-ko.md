# Insomnia 레포지토리 분석 & 활용 정리 (한국어)

> 작성일: 2026-10-07
> 분석 대상 레포: **https://github.com/bmshin94/insomnia**
> 원본(Upstream): **https://github.com/kong/insomnia**
> 분석 기준 커밋: `3b5c31c` (branch `claude/zen-cerf-w3hdbw`) / 버전 `13.2.0`

---

## 0. 목차

1. [레포지토리 정체 파악](#1-레포지토리-정체-파악)
2. [기술 스택](#2-기술-스택)
3. [모노레포 구조 전수조사](#3-모노레포-구조-전수조사)
4. [Insomnia가 하는 일](#4-insomnia가-하는-일)
5. [AI / LLM 내장 기능](#5-ai--llm-내장-기능)
6. [MCP(Model Context Protocol) 지원](#6-mcpmodel-context-protocol-지원)
7. [플러그인 시스템 & 스크립팅](#7-플러그인-시스템--스크립팅)
8. [보안 / 민감정보 처리](#8-보안--민감정보-처리)
9. [설치 및 사용법](#9-설치-및-사용법)
10. [플러그인? 스킬? MCP? — 정체 정리](#10-플러그인-스킬-mcp--정체-정리)
11. [API 토큰 필요 여부](#11-api-토큰-필요-여부)
12. [AI 에이전트 구축에 도움이 되는가](#12-ai-에이전트-구축에-도움이-되는가)
13. [React / PHP로 만들 수 있는가](#13-react--php로-만들-수-있는가)
14. [유튜브 강의 제작 가능성](#14-유튜브-강의-제작-가능성)
15. [수익화 아이디어 (티어별)](#15-수익화-아이디어-티어별)
16. [실행 로드맵](#16-실행-로드맵)
17. [주의사항 & 리스크](#17-주의사항--리스크)

---

## 1. 레포지토리 정체 파악

- 이 레포는 Kong 사의 오픈소스 API 클라이언트 **Insomnia**를 개인 계정으로 **포크(fork)** 한 것이다.
- `package.json`의 `repository` 필드가 `https://github.com/kong/insomnia`를 가리키고, `origin` 리모트는 `https://github.com/bmshin94/insomnia`이다.
- 라이선스: **Apache-2.0** (상업적 이용/수정/재배포 허용, 고지 유지 조건)
- 기본 브랜치: `develop` / 작업 브랜치: `claude/zen-cerf-w3hdbw`
- 포크 고유 변경: 커밋 `92deca6 docs: appended CLAUDE.md persona guide` (PR #1로 머지)

### 규모 지표

| 항목 | 값 |
|---|---|
| TS/TSX 파일 수 | 1,618개 |
| React Router 라우트 수 | 172개 |
| 워크스페이스 패키지 | 10개 |
| 버전 | 13.2.0 |
| Node 요구 버전 | 24.18.0 (`.nvmrc`) |
| npm 요구 버전 | 11+ |

---

## 2. 기술 스택

| 영역 | 기술 |
|---|---|
| 데스크탑 셸 | Electron (main / renderer 프로세스 분리) |
| UI | React + React Router (loader/action 패턴) + React Aria Components |
| 스타일 | TailwindCSS (`clsx`, `tailwind-merge`, `prettier-plugin-tailwindcss`) |
| 언어 | TypeScript 5.8.3 |
| 데이터베이스 | NeDB (`@seald-io/nedb`) — 파일 기반 내장 NoSQL |
| HTTP 엔진 | libcurl (`@getinsomnia/node-libcurl` 3.3.0) |
| 코드 에디터 | CodeMirror |
| CLI 프레임워크 | Commander.js |
| 빌드 | Vite + esbuild + npm workspaces |
| 테스트 | Vitest (유닛) + Playwright (E2E) |
| 린트 | ESLint 9 + Prettier 3.6.2 |
| 개발환경 | `flake.nix` (Nix 지원) |

> HTTP 클라이언트로 `fetch`가 아니라 **libcurl**을 쓰는 이유: 요청에 대한 최대한의 디버깅 가능성과 저수준 제어 확보.

---

## 3. 모노레포 구조 전수조사

```
packages/
├── insomnia/                      13.2.0  메인 Electron 앱 (본체)
├── insomnia-data/                 13.2.0  데이터 모델 + NeDB + 서비스 레이어
├── insomnia-api/                  13.2.0  Insomnia 클라우드 API 클라이언트
├── insomnia-inso/                 13.2.0  CLI 도구 (CI/CD 용)
├── insomnia-vcs/                  13.1.0  Git/VCS 동기화
├── insomnia-testing/              13.2.0  테스트 프레임워크
├── insomnia-smoke-test/           13.2.0  Playwright E2E
├── insomnia-analytics/            13.2.0  분석/트래킹 클라이언트
├── insomnia-scripting-environment/13.2.0  pm/insomnia 스크립팅 API
└── insomnia-component-docs/       0.0.1   Docusaurus 컴포넌트 문서
```

### 메인 앱 내부 (`packages/insomnia/src/`)

| 디렉토리 | 역할 |
|---|---|
| `main/` | Electron 메인 프로세스, IPC 핸들러, MCP 전송, LLM 설정 |
| `ui/` | React 컴포넌트, hooks, context, worker |
| `routes/` | React Router 라우트 172개 (clientLoader/clientAction) |
| `common/` | 양쪽 프로세스 공용 유틸, 설정 타입, 상수 |
| `network/` | 요청 실행 엔진 (gRPC, 인증, 인증서, 쿠키) |
| `templating/` | Nunjucks 렌더링 + QuickJS 샌드박스 |
| `plugins/` | 플러그인 로더 + context API |
| `sync/` `account/` | 동기화 / 인증·암호화 |
| `scripting/` | pre-request / after-response 스크립트 실행 |
| `cpp/` | Windows용 C++ 보안 래퍼 (`build-secure-wrapper.sh`) |
| `konnect/` | Kong Konnect 연동 |
| `basic-components/` | 공용 컴포넌트 라이브러리 |

### 데이터 모델 계층

```
Organization (조직)
 └── Project (local | remote/cloud | git-backed)
      └── Workspace (scope: 'collection' | 'design')   ← collection = 워크스페이스 그 자체
           ├── Base Environment (자동 생성)
           │    └── Sub-Environments
           ├── Cookie Jar (자동 생성)
           └── Request Group (폴더)
                └── Request (HTTP / GraphQL / gRPC / WebSocket / Socket.IO / MCP)
```

### CI/CD (`.github/workflows/`)

`test.yml`, `test-e2e.yml`, `test-cli.yml`, `sast.yml`(보안 스캔),
`release-build.yml`, `release-publish.yml`, `release-start.yml`, `release-recurring.yml`,
`homebrew.yml`, `update-changelog.yml`

### 에이전트 지원 자산

```
CLAUDE.md                    → @AGENTS.md 임포트 + 페르소나 가이드
AGENTS.md                    → 기술스택 / 엄격규칙 / 검증커맨드 / 구조 가이드
.claude/settings.json
.claude/skills/
├── fix-scripting-feature/   → SKILL.md + references/objects/ 문서 24개
├── fix-test-cli-ci/         → CI 디버깅 스킬
└── component-docs/          → 컴포넌트 문서 작성 스킬 (+ template.mdx)
```

---

## 4. Insomnia가 하는 일

**API를 호출·설계·테스트·목(mock)까지 처리하는 올인원 데스크탑 API 클라이언트** (Postman 경쟁 제품).

### 지원 프로토콜
REST/HTTP, GraphQL, WebSocket, SSE(Server-Sent Events), gRPC, Socket.IO, **MCP**

### 핵심 기능 5가지
1. **Debug** — 요청 전송 및 응답 분석 (libcurl 기반 저수준 제어)
2. **Design** — OpenAPI 네이티브 에디터 + 비주얼 프리뷰
3. **Test** — 네이티브 테스트 스위트 + Collection Runner
4. **Mock** — 클라우드 또는 셀프호스팅 목 서버
5. **CI/CD** — `inso` CLI로 린트/테스트 자동화

### 저장 방식 3종
| 방식 | 설명 |
|---|---|
| **Local Vault** | 100% 로컬 저장, 클라우드 전송 없음 |
| **Git Sync** | 임의의 Git 레포에 저장 (클라우드 경유 없음) |
| **Cloud Sync** | 클라우드 협업, E2EE 선택 가능 |

추가로 **Private Environments**: 환경 설정은 프로젝트 저장 방식과 무관하게 항상 로컬에만 저장.

---

## 5. AI / LLM 내장 기능

이 레포의 가장 중요한 발견 중 하나. **LLM이 앱에 내장되어 있다.**

`packages/insomnia/src/common/constants.ts:91`
```ts
export const LLM_BACKENDS = ['gguf', 'claude', 'openai', 'gemini', 'url'] as const;
```

`packages/insomnia/src/main/llm-config-service.ts`
```ts
export type AIFeatureNames = 'aiMockServers' | 'aiCommitMessages' | 'aiMcpClient';

export interface LLMConfig {
  backend: LLMBackend;
  model: string;
  modelDir?: string;
  apiKey?: string;
  url?: string;
  baseURL?: string;
  maxTokens?: number;
  temperature?: number;
  topP?: number;
  topK?: number;
  seed?: boolean;
  repeatPenalty?: number;
  // ...
}
```

### 지원 백엔드 5종
| 백엔드 | 설명 | API 키 |
|---|---|---|
| `gguf` | **로컬 LLM** (`node-llama-cpp`) — 내 PC에서 모델 실행 | 불필요 |
| `claude` | Anthropic Claude API | 필요 |
| `openai` | OpenAI API | 필요 |
| `gemini` | Google Gemini API | 필요 |
| `url` | 커스텀 엔드포인트 (Ollama, vLLM 등) | 서버에 따라 |

설정 UI: `packages/insomnia/src/ui/components/settings/llms/` 에 `claude.tsx`, `openai.tsx`, `gemini.tsx`, `gguf.tsx`, `url.tsx`

### AI 기능 3종
- `aiMockServers` — AI 기반 목 서버 자동 생성
- `aiCommitMessages` — AI 커밋 메시지 생성 (`routes/ai.generate-commit-messages.tsx`)
- `aiMcpClient` — AI가 MCP 서버를 호출

---

## 6. MCP(Model Context Protocol) 지원

의존성: `@modelcontextprotocol/sdk: ^1.17.5`

### 관련 파일 전체
```
insomnia-data/src/models/mcp-request.ts        McpRequest 모델
insomnia-data/src/models/mcp-response.ts       McpResponse 모델
insomnia-data/src/models/mcp-payload.ts        McpPayload 모델
insomnia-data/node-src/services/mcp-*.ts       서비스 레이어

insomnia/src/main/mcp/
├── transport-stdio.ts             stdio 전송
├── transport-streamable-http.ts   Streamable HTTP 전송
├── oauth-client-provider.ts       MCP OAuth 인증
├── client-requests.ts             클라이언트 요청
├── common.ts / types.ts
insomnia/src/main/network/mcp.ts
insomnia/src/main/mcp-generate-sampling-response.mjs

insomnia/src/ui/components/mcp/
├── mcp-pane.tsx / mcp-request-pane.tsx
├── mcp-roots-panel.tsx
├── mcp-notification-tab.tsx
└── mcp-url-bar.tsx
insomnia/src/ui/components/dropdowns/mcp-actions-dropdown.tsx
insomnia/src/ui/components/modals/mcp-certificates-modal.tsx
insomnia/src/ui/hooks/use-mcp-ready-state.ts

insomnia/src/common/mcp-utils.ts
insomnia/src/routes/ai.mcp-generate-sampling-response.tsx
insomnia/src/routes/organization.$organizationId.project.$projectId.workspace.$workspaceId.mcp.tsx
```

### 모델 정의 (`mcp-request.ts`)
```ts
export const TRANSPORT_TYPES = {
  STDIO: 'stdio',
  HTTP: 'streamable-http',
} as const;

export interface BaseMcpRequest {
  url: string;
  transportType: McpTransportType;
  description: string;
  headers: RequestHeader[];
  authentication: RequestAuthentication | {};
  env: EnvironmentKvPairData[];
  mcpStdioAccess: boolean;
  roots: Root[];
  subscribeResources: string[];
  connected: boolean;
  sslValidation: boolean;
  disableUserAgentHeader?: boolean;
}

export type McpServerPrimitiveTypes = 'tools' | 'resources' | 'prompts' | 'resourceTemplates';
```

MCP 4대 프리미티브(tools/resources/prompts/resourceTemplates), roots, resource subscription, OAuth, sampling, 인증서 검증까지 모두 지원. **Insomnia는 MCP "서버"가 아니라 MCP "클라이언트(테스트 도구)"다.**

---

## 7. 플러그인 시스템 & 스크립팅

### 확장 타입 (`src/plugins/index.ts`)
```ts
TemplateTag | Theme | RequestHook | ResponseHook
| RequestAction | RequestGroupAction | WorkspaceAction | DocumentAction
```

### 플러그인 context API (`src/plugins/context/`)
`app`, `data`, `network`, `request`, `response`, `store`

### 보안 샌드박스 (진행 중인 최신 작업)
- `src/templating/sandbox/` — **QuickJS** 기반 템플릿 태그 샌드박스
- 매니페스트 기반 모듈 권한 통제 (`Module 'fs' not permitted by manifest`)
- 데모 플러그인: `examples/insomnia-plugin-sandbox-demo/`
  - 설치: Preferences → Plugins → Reveal Plugins Folder → 폴더 복사 → Reload Plugins
  - 토글: Preferences → Scripting → "Run template tags in sandbox (experimental)"

### 스크립팅 객체 (`insomnia-scripting-environment/src/objects/`, 24개)
`insomnia`, `request`, `response`, `environments`, `variables`, `collection`, `folders`,
`cookies`, `headers`, `auth`, `certificates`, `proxy-configs`, `send-request`, `test`,
`urls`, `utils`, `console`, `execution`, `interpolator`, `properties`, `request-info`,
`async-objects`, `interfaces`, `index`

### 템플릿
Nunjucks가 Web Worker에서 실행됨. 사용법: `{{ _.variable_name }}`

---

## 8. 보안 / 민감정보 처리

| 메커니즘 | 설명 |
|---|---|
| **Vault 시스템 (AES-GCM)** | 환경변수 시크릿 암호화 (`EnvironmentKvPairDataType.SECRET`) |
| **Electron safeStorage** | OS 네이티브 암호화 (`window.main.secretStorage`) |
| **Private Environments** | 환경 설정은 절대 클라우드로 업로드되지 않음 |
| **C++ 보안 래퍼** | Windows 실행파일 래핑 (`src/cpp/`, `build-secure-wrapper.sh`) |
| **SAST 워크플로우** | `.github/workflows/sast.yml` 정적 보안 분석 |

인증 방식 지원: API Key, Basic, Bearer, OAuth 1/2, mTLS 인증서, Git 자격증명 등

**금지 패턴**: URL에 `?api_key=sk-xxxx` 하드코딩 → Git에 노출됨
**권장 패턴**: Environment에 SECRET으로 저장 → `{{ _.api_key }}` 로 참조

---

## 9. 설치 및 사용법

### A. 사용만 할 경우
https://insomnia.rest 에서 설치 파일 다운로드 (Mac/Windows/Linux).
로그인 없이 쓰려면 **Scratch Pad** 모드 사용.

### B. 소스로 개발/실행

```bash
# 0) Node 버전 맞추기 (필수)
fnm use "$(cat .nvmrc)"      # 또는 nvm use 24.18.0
node -v                       # v24.18.0
npm -v                        # 11 이상

# 1) 의존성 설치 (--ignore-scripts 사용 금지: Electron이 반쪽 설치됨)
npm ci

# 2) 개발 모드 실행
npm run dev
npm run dev:autoRestart        # 코드 변경 시 자동 재시작
```

### 검증 커맨드 (AGENTS.md 규칙)
```bash
npm run lint                   # ESLint 전체 워크스페이스
npm run type-check             # TypeScript 체크 전체
npm test                       # 테스트 전체
npm test -w packages/insomnia  # 메인 앱만
```

### E2E 테스트
```bash
npm run test:smoke:dev                 # 전체
npm run test:smoke:dev -- <제목일부>    # 필터
npm run test:crit:dev                  # Critical 프로젝트
```

### 빌드 / 패키징
```bash
npm run app-build
npm run app-package
```

### inso CLI
```bash
npm run inso-start
npm run inso-package
./packages/insomnia-inso/bin/inso run test       # 테스트 실행
./packages/insomnia-inso/bin/inso lint spec      # OpenAPI 린트
./packages/insomnia-inso/bin/inso export spec    # 스펙 추출
```

### 자주 발생하는 문제
1. **libcurl 바이너리 충돌** — 앱은 Electron 빌드, CLI는 Node 빌드를 사용
   ```bash
   npm run install-libcurl-electron   # 앱 작업 시
   npm run install-libcurl-node       # inso CLI 작업 시
   ```
2. **Node 버전 불일치** — 24.18.0 정확히 맞춰야 함
3. **Nix 사용자** — `nix develop` 으로 환경 구성 가능 (`flake.nix`)

### 앱 사용 흐름
```
1. Project 생성 → 저장방식 선택 (Local Vault / Git Sync / Cloud Sync)
2. Collection(Workspace) 생성
3. Request 추가 → METHOD + URL 입력 → Send
4. Environment에 {{ _.base_url }}, {{ _.api_key }} 변수 등록
5. Pre-request Script로 토큰 자동 발급
6. Tests 탭에서 검증 코드 작성
7. Runner로 컬렉션 전체 실행
8. CI에서는 inso CLI로 자동화
```

---

## 10. 플러그인? 스킬? MCP? — 정체 정리

**정답: 세 가지 모두 아니다. Insomnia는 그 전부를 담는 "완성된 애플리케이션"이다.**

| 질문 | 답 | 근거 |
|---|---|---|
| 플러그인인가? | 아니다. 플러그인을 **받아주는 호스트 앱** | `src/plugins/index.ts` 가 플러그인 로더 |
| 스킬인가? | 아니다. 단, **스킬을 품고 있다** | `.claude/skills/` 3개 |
| MCP인가? | MCP 서버가 아니다. **MCP 클라이언트(테스트 도구)** | `@modelcontextprotocol/sdk` + `src/main/mcp/` |

```
┌──────────── Insomnia (데스크탑 애플리케이션) ────────────┐
│  플러그인 호스트    ← 플러그인을 설치받아 실행            │
│  MCP 클라이언트     ← MCP 서버를 호출/테스트              │
│  스크립팅 엔진      ← JS 스크립트 실행 (pm/insomnia API)  │
│  .claude/skills     ← 이 코드를 고치는 개발자(AI)용 스킬   │
└──────────────────────────────────────────────────────────┘
```

관계 정리:
- 플러그인을 **만들어서** Insomnia에 꽂을 수 있다 (`insomnia-plugin-*` npm 패키지)
- MCP 서버를 **만들면** Insomnia로 테스트할 수 있다
- Insomnia 코드를 수정할 때 `.claude/skills`의 스킬이 AI 작업을 돕는다

---

## 11. API 토큰 필요 여부

| 하려는 일 | 토큰 필요? | 비고 |
|---|---|---|
| Scratch Pad만 사용 | 불필요 | 로그인조차 불필요 |
| Local Vault 로컬 저장 | 계정 로그인 필요 (토큰 아님) | 데이터는 로컬에만 |
| Cloud Sync / 조직 기능 | Insomnia 계정 필요 | 유료 플랜 필요 기능 있음 |
| Git Sync | **GitHub/GitLab PAT** | `routes/git-credentials.*` |
| 외부 API 호출 | **해당 API의 토큰** | Insomnia는 전달자 |
| AI 기능 (claude/openai/gemini) | **각 LLM API 키** | `LLMConfig.apiKey` |
| AI 기능 (gguf 로컬) | **불필요** | 내 PC에서 모델 실행 |
| 소스 빌드/개발 | 불필요 | npm만 동작하면 됨 |
| MCP 서버 테스트 | 서버에 따라 | OAuth 지원 (`oauth-client-provider.ts`) |

---

## 12. AI 에이전트 구축에 도움이 되는가

**결론: 매우 도움이 된다. 단, 역할을 정확히 이해해야 한다.**

Insomnia는 LangChain/CrewAI 같은 **에이전트 프레임워크가 아니다.** 대신 에이전트 개발 사이클에서 3가지 역할을 한다.

### 역할 1 — MCP 서버 개발의 계측 장비 (최대 가치)
```
MCP 서버 개발 시
 기존: 터미널에서 JSON-RPC 수동 입력
 Insomnia: tools/resources/prompts/resourceTemplates 목록 UI 확인
          stdio / streamable-http 양쪽 테스트
          OAuth 인증 플로우 테스트
          roots, resource subscription 테스트
          sampling 응답 테스트
          컬렉션으로 저장 → Git 커밋 → CI 자동 회귀테스트
```
에이전트 개발에서 가장 어려운 "서버 문제 vs LLM 문제" 구분을 서버만 떼어내 검증 가능.

### 역할 2 — 에이전트가 쓸 툴 API의 검증·문서화
- 외부 API 스키마/파라미터 확정 → 툴 정의(JSON Schema)로 변환
- OpenAPI 스펙 설계 → `inso lint` 로 CI 검증
- AI 목 서버로 가짜 응답 생성 → 에이전트 선개발

### 역할 3 — 레퍼런스 구현 (교과서)
| 파일 | 배울 내용 |
|---|---|
| `src/main/mcp/transport-stdio.ts` | stdio MCP 전송 구현 |
| `src/main/mcp/transport-streamable-http.ts` | HTTP MCP 전송 구현 |
| `src/main/mcp/oauth-client-provider.ts` | MCP OAuth 구현 |
| `src/main/llm-config-service.ts` | 멀티 LLM 백엔드 추상화 |
| `src/templating/sandbox/` | QuickJS 샌드박스 (툴 안전 실행) |
| `src/plugins/context/` | 플러그인 권한 모델 |

### 실전 로드맵
```
1단계: 앱 설치 → 설정에서 LLM API 키 연결 → AI 기능 체험
2단계: MCP Request 만들어 공개 MCP 서버에 연결 (감각 익히기)
3단계: MCP 서버 직접 구현 (TypeScript SDK) → Insomnia로 디버깅
4단계: Insomnia 플러그인으로 자체 기능 확장
5단계: src/main/mcp/ 참고해 독립 에이전트 앱 개발
```

---

## 13. React / PHP로 만들 수 있는가

### React
**이미 React로 만들어져 있다.** (React + React Router + React Aria + Tailwind)

동일한 것을 처음부터 만들 경우:

| 기능 | 웹(React만) | 데스크탑(React+Electron) |
|---|---|---|
| UI / 요청 편집기 | 가능 | 가능 |
| API 호출 | **CORS 제약** | 자유 |
| 임의 헤더/쿠키 설정 | 브라우저가 차단 | 가능 |
| gRPC / 소켓 | 거의 불가 | 가능 |
| 로컬 파일 저장 | 제한적 | 가능 |
| MCP stdio 전송 | **불가** (프로세스 실행 필요) | 가능 |

→ **웹만으로는 불가능. 백엔드 프록시 또는 Electron이 필수.** 이것이 Insomnia가 Electron인 이유.

### PHP
데스크탑 앱용으로는 비권장(NativePHP로 가능하긴 함). PHP의 적합한 자리:
1. **백엔드 프록시** — React UI → PHP가 실제 요청 대행 → CORS 해결
2. **테스트 대상 API 서버** (Laravel)
3. **MCP 서버 구현** (`logiscape/mcp-sdk-php` 등 SDK 존재)
4. **컬렉션 공유 웹서비스 / 결제·라이선스 서버**

### 권장 아키텍처
```
React (Vite + TS + Tailwind)        프론트 UI
        │ HTTP
PHP (Laravel) 또는 Node 프록시       요청 대행 + 로그 + 인증
        │
외부 API / MCP 서버
```
이 구조라면 "웹 기반 미니 API 클라이언트"를 2~3주에 구현 가능 (포트폴리오용 적합).

---

## 14. 유튜브 강의 제작 가능성

**가능하다.** Apache-2.0 라이선스상 코드 리뷰·강의 제작은 합법.

### 유리한 이유
1. 무료 오픈소스 → 시청자 진입장벽 없음
2. "API + AI 에이전트 + MCP" = 현재 최고 관심 주제
3. 난이도 스펙트럼 넓음 (설치 ~ Electron 아키텍처)
4. **한국어 MCP 테스트 콘텐츠가 거의 없음 → 블루오션**

### 시리즈 기획안

**시즌 1: 입문**
| # | 제목 | 길이 |
|---|---|---|
| 1 | Postman 대안, 무료 오픈소스 Insomnia 완전정복 | 12분 |
| 2 | API 요청 5분만에 보내기 (왕초보편) | 8분 |
| 3 | Environment 변수로 개발/운영 전환하기 | 10분 |
| 4 | API 키 안전하게 저장하기 (Vault 암호화) | 10분 |
| 5 | Git Sync — API 컬렉션을 Git으로 버전관리 | 15분 |

**시즌 2: MCP & AI (차별화 핵심)**
| # | 제목 |
|---|---|
| 6 | MCP가 무엇인가 — 10분 개념 정리 |
| 7 | Insomnia로 MCP 서버 테스트하기 |
| 8 | 내 PC에서 무료 AI 실행하기 (gguf 로컬 LLM) |
| 9 | Claude API 연결해서 목 서버 자동 생성 |
| 10 | MCP 서버 직접 만들고 Insomnia로 디버깅 |

**시즌 3: 고급/개발자**
| # | 제목 |
|---|---|
| 11 | Insomnia 플러그인 만들기 — 커스텀 템플릿 태그 |
| 12 | 1,618개 TS 파일 대형 오픈소스 코드 읽는 법 |
| 13 | Electron 앱 구조 해부 (main vs renderer) |
| 14 | React Router v7 loader/action 실전 패턴 |
| 15 | inso CLI로 CI/CD에 API 테스트 통합 |
| 16 | QuickJS 샌드박스 — 플러그인 보안 설계 분석 |

**시즌 4: 따라만들기**
"React로 Insomnia 클론 만들기" 10부작 → 포트폴리오 완성

### 제작 팁
- 1번 영상은 "Postman 유료화 대안" 각도로 (검색 수요 높음)
- OBS 화면녹화 + 터미널 폰트 확대 (가독성)
- 10분 이하 쇼츠로 분할 ("MCP 60초 설명")
- 설명란에 GitHub 포크 링크 → 스타 유입
- Insomnia 로고 사용 시 "비공식 강의" 명시 (상표 주의)

---

## 15. 수익화 아이디어 (티어별)

### 법적 전제
```
라이선스: Apache-2.0
허용: 상업적 사용 / 수정 / 재배포 / 특허 사용권
조건: 라이선스·저작권 고지 유지, 변경사항 명시
금지: "Insomnia", "Kong" 상표(이름/로고) 사용 → 리브랜딩 필수
```

### 티어 1: 즉시 시작 가능 (투자 거의 0)

#### 아이디어 1 — 유튜브 + 온라인 강의
| 항목 | 내용 |
|---|---|
| 수익원 | 애드센스 + 강의 플랫폼 + 멤버십 |
| 초기비용 | 0원 |
| 예상수익 | 애드센스 월 30~300만, 강의 1강좌 300~2,000만 |
| 난이도 | 하 |

전략: "Postman 대안" + "MCP 한국어 최초" 투 트랙

#### 아이디어 2 — 기술 블로그 / 유료 뉴스레터
```
무료 블로그로 SEO 선점 → 유료 뉴스레터(월 5,900원) 전환
주제: "AI 에이전트 개발자를 위한 API/MCP 위클리"
목표: 구독자 500명 × 5,900원 = 월 295만원
```

#### 아이디어 3 — 유료 Insomnia 플러그인 (강력 추천)
| 플러그인 | 타겟 | 가격 |
|---|---|---|
| 한국 결제 API 팩 (토스/KG이니시스/카카오페이 서명 자동생성) | 한국 개발자 | $29 평생 |
| 엔터프라이즈 Auth 팩 (AWS SigV4, HMAC, mTLS, JWT 자동갱신) | 기업 | $49 |
| 응답 데이터 시각화 (JSON → 차트/테이블) | 전체 | $19 |
| MCP 스키마 → 툴 정의 변환기 | AI 개발자 | $39 |
| AI 테스트 자동생성 (응답 기반 assertion 작성) | QA | $39 |
| 컬렉션 → 문서 자동생성 (Notion/Confluence) | 팀 | $29 |

수익모델: 무료 기본판(마케팅) + 유료 프로판 (Gumroad / Lemon Squeezy)
예시: 100명 × $29 ≈ 400만원, 유지보수 부담 낮음

### 티어 2: 중기 (1~3개월)

#### 아이디어 4 — MCP 서버 개발 & 판매 (최고 유망)
```
시장 상황: AI 에이전트 붐 → MCP 수요 급증 → 한국 서비스용 MCP 서버 부재
```
| MCP 서버 | 설명 | 수익모델 |
|---|---|---|
| 한국 금융 MCP | 오픈뱅킹/증권 API → AI 조회 | SaaS 월 2~5만 |
| 이커머스 MCP | 쿠팡/네이버 스마트스토어 연동 | 월 3~10만/셀러 |
| 국내 SaaS MCP | 잔디/두레이/카카오워크 | B2B 월 10만+ |
| 공공데이터 MCP | data.go.kr 래핑 | 프리미엄 구독 |
| 한글 문서 MCP | HWP 파싱 → AI 분석 | 틈새 독점 |

Insomnia의 역할: 개발·검증·회귀테스트 플랫폼
수익 시나리오: 셀러 50명 × 월 3만 = **월 150만 (반복수익)**

#### 아이디어 5 — API/MCP 테스팅 컨설팅 & SI
```
서비스: 기업 API를 Insomnia 컬렉션으로 체계화 + CI/CD 자동화 구축
단가: 소규모 300~800만 / 중견 1,500~3,000만
추가: 월 유지보수 50~150만
근거: inso CLI 기반 CI 통합 노하우는 희소 기술
```

#### 아이디어 6 — 프리미엄 컬렉션 마켓플레이스
```
"한국 API 컬렉션 팩" 판매
- 토스페이먼츠 전체 플로우 (요청 40개 + 테스트 + 환경설정)  $39
- 네이버/카카오 OAuth 완전판                               $29
- 쿠팡 윙 셀러 API 올인원                                  $59
- 공공데이터포털 인기 API 100선                            $49

가치 제안: 세팅 시간 8시간 → 10분
플랫폼: Gumroad / 자체 사이트 / GitHub Sponsors
```

#### 아이디어 7 — 유료 템플릿 & 보일러플레이트
```
"MCP 서버 스타터킷" (TypeScript / Python / PHP 3종)
구성: MCP SDK + 인증 + 테스트 + Insomnia 컬렉션 + Docker + CI
가격: $79 / 기업용 $299
```

### 티어 3: 장기·고수익 (3~12개월)

#### 아이디어 8 — 리브랜딩 SaaS 제품화 (최고 상한선)
```
제품명 예시: "KoAPI" / "ApiNest" / "MCPLab"  ("Insomnia" 상표 사용 금지)

차별화 포인트:
- 100% 한국어 + 한국 API 프리셋 내장
- 완전 온프레미스 (금융/공공 = 클라우드 금지 시장)
- MCP 전문 IDE 포지셔닝 (AI 에이전트 개발자 타겟)
- 국내 보안인증 대응 (ISMS-P)
- 한국어 기술지원

가격: Free / Pro 월 1.5만 / Team 월 3만·인 / Enterprise 연 2,000만+
```
| 시나리오 | 고객 구성 | 연매출 추정 |
|---|---|---|
| 보수적 | Pro 200명 | 약 3,600만 |
| 현실적 | Pro 500 + Team 20팀(10인) | 약 1억 6천 |
| 낙관적 | + Enterprise 5곳 | 약 2억 7천 |

리스크: upstream 추적 유지보수 부담, 상표·로고 전면 교체 필수, 1인 운영 난이도 높음

#### 아이디어 9 — 자체 플러그인 마켓플레이스 운영
```
플랫폼을 만들고 수수료 20~30% 수취
초기: 자체 플러그인 10개로 시작 → 외부 개발자 유치
수익: GMV 1억 × 25% = 2,500만
```

#### 아이디어 10 — 기업 교육 / 사내 워크샵
```
"API 테스팅 자동화 + MCP 실무" 2일 과정
단가: 1회 300~600만 (10~20명)
월 2회 = 월 600~1,200만
강점: 오픈소스라 교재 라이선스 비용 0
```

### ROI 랭킹
| 순위 | 아이디어 | 투자 | 수익잠재 | 추천도 |
|---|---|---|---|---|
| 1 | MCP 서버 SaaS | 중 | 매우높음 | ★★★★★ |
| 2 | 유료 플러그인 | 낮음 | 중상 | ★★★★★ |
| 3 | 유튜브 + 강의 | 낮음 | 중상 | ★★★★ |
| 4 | 컨설팅 / SI | 낮음 | 높음 | ★★★★ |
| 5 | 컬렉션 팩 | 매우낮음 | 중 | ★★★ |
| 6 | 기업 교육 | 중 | 높음 | ★★★ |
| 7 | SaaS 제품화 | 매우높음 | 매우높음 | ★★ |

---

## 16. 실행 로드맵

```
0~1개월   유튜브 시작 + 플러그인 1개 무료 배포
          목표: 브랜딩 + 피드백 (수익 0~50만)
            ↓
1~3개월   유료 플러그인 2~3개 + 컬렉션 팩 판매
          목표: 첫 유료 매출 (월 50~200만)
            ↓
3~6개월   MCP 서버 SaaS 1개 론칭 + 강의 출시
          목표: 반복수익 구축 (월 200~600만)
            ↓
6~12개월  컨설팅/교육으로 현금흐름 확보 + SaaS 제품화 결정
          목표: 월 500~1,500만
```

핵심 판단:
- **MCP 서버**가 현재 가장 큰 기회 (한국 서비스 연결 MCP 서버 부재 → 선점 시 사실상 표준)
- 유튜브는 **마케팅 채널**로 병행 → 플러그인/SaaS 판매 시 신뢰 자산
- 티어 3(SaaS 제품화)부터 단독 시작은 비권장 → 티어 1 → 2 → 3 순서 권장

---

## 17. 주의사항 & 리스크

| 항목 | 내용 |
|---|---|
| **상표** | "Insomnia", "Kong" 이름/로고는 Apache-2.0에 포함되지 않음 → 재배포 시 리브랜딩 필수 |
| **계정 요구** | 본체 기능 상당수가 Insomnia 계정 로그인 필요 (Scratch Pad는 예외) |
| **유료 기능** | 무제한 협업, Git Sync, 조직 생성, SAML/OIDC 등은 유료 플랜 |
| **NeDB** | 공식적으로 유지보수 중단된 라이브러리 (2016년 마지막 배포) — 아키텍처에 깊게 결합되어 교체 어려움 (DEVELOPMENT.md에 명시) |
| **inso CLI 번들링** | 렌더러 거의 전체를 번들해 Node에서 실행하는 구조적 문제 존재 |
| **libcurl 바이너리** | Electron/Node 두 빌드를 왕복 설치해야 함 |
| **upstream 추적** | 포크 유지 시 원본 업데이트 병합 비용 지속 발생 |
| **보안** | API 키를 URL/코드에 하드코딩 금지 → Vault SECRET 사용 |
| **개발환경** | `npm ci --ignore-scripts` 사용 금지 (Electron 반쪽 설치) |

---

## 참고 링크

| 항목 | URL |
|---|---|
| **이 레포 (포크)** | https://github.com/bmshin94/insomnia |
| **원본 레포 (upstream)** | https://github.com/kong/insomnia |
| 공식 웹사이트 / 다운로드 | https://insomnia.rest |
| 가격 정책 | https://insomnia.rest/pricing |
| 이슈 트래커 | https://github.com/kong/insomnia/issues |
| Inso CLI 문서 | https://docs.insomnia.rest/inso-cli/introduction |
| Slack 커뮤니티 | https://chat.insomnia.rest/ |
| Model Context Protocol | https://modelcontextprotocol.io |

### 레포 내 참고 문서
- `README.md` — 제품 개요, 저장 방식, 계정 정책
- `AGENTS.md` — 기술 스택, 엄격 규칙, 검증 커맨드, 구조 가이드
- `DEVELOPMENT.md` — 아키텍처 개요, inso 빌드 과정, 알려진 문제
- `CONTRIBUTING.md` — 기여 가이드
- `SECURITY.md` — 보안 취약점 신고
- `CHANGELOG.md` — 전체 변경 이력
- `packages/insomnia-smoke-test/README.md` — E2E 테스트 가이드
- `examples/insomnia-plugin-sandbox-demo/README.md` — 플러그인 샌드박스 데모
- `.claude/skills/` — 코드 수정용 AI 스킬 3종

---

_이 문서는 `bmshin94/insomnia` 레포지토리를 전수조사하여 작성된 분석 및 활용 전략 정리본입니다._
