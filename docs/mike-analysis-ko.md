# Mike 프로젝트 분석 및 활용 가이드 (한국어)

이 문서는 Mike(MikeOSS) 저장소를 처음 받은 사람을 위해, 저장소 내용을 직접
확인한 결과를 정리한 분석 노트입니다. 프로젝트의 정체, 구조, 설치 방법,
로컬 에이전트 개발에 참고할 지점, 그리고 활용/수익화 방향을 다룹니다.

- 저장소: https://github.com/bmshin94/mike
- 원본(업스트림): https://github.com/Open-Legal-Products/mike
- 워크플로우 저장소: https://github.com/Open-Legal-Products/mike-workflows
- 공식 사이트: https://mikeoss.com
- 라이선스: GNU AGPL-3.0-only (`LICENSE`)
- 작성일: 2026-09-18

---

## 1. 프로젝트 정체

Mike(MikeOSS)는 **문서 검토, 초안 작성, 법률 리서치를 위한 오픈소스 법률 AI
플랫폼**입니다. 라이브러리나 플러그인이 아니라, 인증·권한·문서 저장·작업 큐·
감사 로그까지 갖춘 **완성형 풀스택 SaaS 애플리케이션**입니다.

구성 스택:

- Next.js 프론트엔드 (React, TypeScript, Tailwind, shadcn `new-york`)
- Express 백엔드 (문서 처리, DB 접근, LLM 연동)
- Supabase Auth / Postgres
- Cloudflare R2 호환 오브젝트 스토리지 (로컬은 RustFS)
- Redis 기반 작업 처리 + DB 기반 큐(`lib/dbq/`)

규모(저장소 실측):

| 항목 | 수치 |
| --- | --- |
| TypeScript/TSX 파일 | 약 1,050개 |
| 코드 라인 | 약 247,000줄 |
| DB 테이블 (`backend/schema.sql`) | 56개 |
| 마이그레이션 (`backend/migrations/`) | 91개 |
| 백엔드 도메인 모듈 | 18개 |

---

## 2. 저장소 구조

| 경로 | 역할 |
| --- | --- |
| `frontend/` | Next.js 웹 애플리케이션 |
| `backend/` | Express API, 문서 처리, DB 접근, LLM 연동 |
| `word-addin/` | Microsoft Word 작업창 애드인 (베타) |
| `packages/contracts/` | 프론트/백엔드 공용 직렬화 타입 (`@mike/contracts`) |
| `e2e/` | Playwright 브라우저 테스트 |
| `docker-compose.yml` | 앱 + Supabase + RustFS + Redis + Mailpit 전체 스택 |
| `docs/` | 아키텍처·배포·테스트·디자인 시스템 문서 |
| `backend/schema.sql` | 신규 설치용 전체 스키마 |
| `backend/migrations/` | 기존 배포용 증분 마이그레이션 |
| `loadtest/`, `scripts/`, `supabase/` | 부하 테스트, 보조 스크립트, 로컬 Supabase 설정 |

### 백엔드 도메인 모듈 (`backend/src/modules/`)

`audit`, `auth`, `chat`, `documents`, `downloads`, `library`, `memory`,
`models`, `orgs`, `project-chat`, `projects`, `quick-actions`,
`source-documents`, `tabular`, `uploads`, `user`, `word-chat`, `workflows`

레이어 규칙은 `docs/backend-architecture.md`가 기준이며,
`backend/src/__tests__/architecture.test.ts`가 테스트로 강제합니다
(예: `lib/`은 `modules/`를 import할 수 없고, 라우트는 DB를 직접 조회하지 않음).

---

## 3. 핵심 기능

1. **문서 기반 AI 어시스턴트** — `modules/chat/`
   - `engine/`에 프롬프트, SSE 스트리밍, 툴 디스패처, 컨텍스트 빌더가 분리되어 있음
   - `engine/verifyCitations.ts`가 모델이 제시한 인용을 실제 소스로 재검증 (환각 판례 방어)
2. **Tabular Review (표 형태 대량 검토)** — `modules/tabular/`
   - 문서 수십~수백 건에서 동일 항목을 추출해 표로 정리
   - 행/열/셀 단위 추출, 백그라운드 추출 잡, 스트리밍 생성 지원
3. **재사용 워크플로우** — `modules/workflows/`
   - 검토 절차를 템플릿화. 공식 워크플로우는 `mike-workflows` 저장소에서 동기화
   - `workflow_addons`, `workflow_open_source_submissions` 테이블로 애드온 생태계 전제
4. **문서 라이브러리 / 버전 관리** — `documents`, `library`, `uploads`, `source-documents`
   - PDF/DOCX/XLSX 파싱(`pdfjs-dist`, `mammoth`, `xlsx`, `libreoffice-convert`)
   - 문서 버전 생성·활성화·교체·삭제는 `modules/documents/` 파사드가 단독 소유
5. **AI 장기 기억** — `modules/memory/`, `lib/memory/`
   - `memory_files`, `memory_consolidation_states`, `memory_consolidation_results` 등으로 대화 요약·압축 저장
6. **미국 판례 리서치** — `lib/courtlistener.ts`
   - CourtListener 연동 + `courtlistener_citation_index` 등 인덱스 캐싱
7. **MCP 커넥터 (클라이언트 측)** — `lib/mcp/`
   - `client.ts`, `oauth.ts`, `servers.ts`, `providers.ts`
   - OAuth 동적 클라이언트 등록(RFC 7591) 지원, Slack 프리셋 제공
   - 쓰기 계열 툴은 사람 확인 절차가 없어 의도적으로 비활성
8. **Word 애드인** — `word-addin/`
   - 워드 작업창에서 문서 편집 제안 (`engine/wordDocumentEdits.ts`, `wordClientTools.ts`)
9. **엔터프라이즈 기능** — 조직/멤버/초대, MFA, SSO, RLS, 감사 로그(`audit_events`), 레이트 리밋, 변조 방지 내보내기

### 지원 LLM 프로바이더 (`backend/src/lib/llm/providers.ts`)

Anthropic, Google Gemini, OpenAI, OpenRouter, AI Gateway, OpenAI 호환 엔드포인트,
그리고 **Ollama(로컬 모델)**. 즉 특정 벤더에 종속되지 않고, 완전 오프라인 구동도 가능합니다.

---

## 4. 설치 및 사용법

요구사항: Docker, Node.js 22 이상, LLM API 키 1개(또는 Ollama).

```bash
git clone https://github.com/bmshin94/mike.git
cd mike

cp .env.example .env
cp backend/.env.example backend/.env

# 서로 다른 값으로 각각 생성해 backend/.env에 입력
openssl rand -hex 32   # DOWNLOAD_SIGNING_SECRET
openssl rand -hex 32   # USER_API_KEYS_ENCRYPTION_SECRET

# backend/.env 에 LLM 키 중 하나 입력
# ANTHROPIC_API_KEY=... / OPENAI_API_KEY=... / GEMINI_API_KEY=...
# 또는 Ollama 전용 사용: OLLAMA_BASE_URL, OLLAMA_MODEL

docker compose up --build
```

- 웹: http://localhost:3000 (계정 생성)
- Supabase 게이트웨이: http://localhost:54321
- 로컬 메일 수신(Mailpit): Compose에 포함된 메일 캡처 서비스 사용
- 자세한 안내: `docs/local-development.md`

개발 및 검증 명령:

```bash
npm run dev  --prefix backend      # API
npm run dev  --prefix frontend     # 웹

npm test     --prefix backend
npm run build --prefix backend
npm test     --prefix frontend
npm run lint --prefix frontend
npm run typecheck --prefix word-addin
npm run test:e2e                   # Playwright
```

> 주의: `.env.example`에 들어있는 Supabase 키/데모 크리덴셜은 **로컬 개발 전용
> 공개 값**입니다. 실제 배포 시 전부 교체해야 합니다(`docs/deployment.md`).

---

## 5. 자주 나오는 질문 정리

### 플러그인인가, 스킬인가, MCP인가?

세 가지 모두 아닙니다. Mike는 **독립 실행되는 풀스택 웹 애플리케이션**입니다.

- Claude Code 플러그인: 아님
- Skill(`SKILL.md`): 아님
- MCP 서버(툴 제공자): 아님
- **MCP 클라이언트(툴 소비자): 해당됨** — `backend/src/lib/mcp/`에서 외부 MCP
  서버(Slack 등)를 붙여 어시스턴트가 사용

저장소에 `CLAUDE.md` / `AGENTS.md`가 있는 것은 이 저장소를 AI 코딩 에이전트로
개발하기 위한 **개발 규칙 문서**이며, Mike 자체의 성격과는 무관합니다.

### API 토큰이 필요한가?

필수(택 1): `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`,
`OPENROUTER_API_KEY` 중 하나. **또는 Ollama만 사용하면 외부 토큰 0개**입니다.

필수(직접 생성하는 내부 시크릿, 비용 없음): `DOWNLOAD_SIGNING_SECRET`,
`USER_API_KEYS_ENCRYPTION_SECRET`, `MCP_CONNECTORS_ENCRYPTION_SECRET`,
`AUTH_HANDOFF_ENCRYPTION_SECRET`, `MANIFEST_SIGNING_KEY`.

선택: `COURTLISTENER_API_TOKEN`(판례 검색),
`SLACK_MCP_OAUTH_CLIENT_ID/SECRET`, `GOOGLE_MCP_OAUTH_CLIENT_ID/SECRET`,
`MIKE_WORKFLOWS_GITHUB_TOKEN`.

비용 감각: 대량 Tabular Review는 토큰 사용량이 크므로, 실험 단계에서는 Ollama나
저가 모델(Gemini Flash, Haiku 등)을 사용하는 것이 안전합니다.

### 왜 주목받는 프로젝트인가

1. 상용 법률 AI(예: Harvey) 수준의 기능 범위를 AGPL로 전면 공개
2. 보안 민감·고단가 산업(법률)에 자체 호스팅 선택지를 제공
3. 데모 수준이 아니라 조직/권한/RLS/감사 로그/MFA/SSO/레이트 리밋/E2E·뮤테이션
   테스트까지 갖춘 실전 제품
4. MS Word 애드인 포함 — 실무 워크플로우를 실제로 반영
5. 아키텍처 경계를 테스트로 강제하는 드문 규율
6. Ollama 지원으로 완전 오프라인 운용 가능
7. 법률 도메인을 제거하면 범용 "문서 AI 플랫폼" 골격으로 재사용 가능

### 로컬 에이전트 개발에 참고할 파일

| 주제 | 파일 |
| --- | --- |
| 툴 호출 루프 / 스키마 | `backend/src/modules/chat/engine/tools/toolDispatcher.ts`, `toolSchemas.ts` |
| SSE 스트리밍 | `engine/streaming.ts`, `engine/routeStreaming.ts`, `lib/assistantSse.ts`, `lib/sseHeartbeat.ts` |
| 멀티 프로바이더 추상화 | `lib/llm/providers.ts` (Ollama 포함) |
| 모델 라우팅/선택 | `lib/modelSelection.ts`, `lib/routerModels.ts` |
| 에이전트 장기 기억 | `modules/memory/`, `lib/memory/`, `docs/memory.md` |
| 출력 검증(환각 방어) | `engine/verifyCitations.ts`, `engine/citations.ts` |
| MCP 클라이언트 연결 | `lib/mcp/client.ts`, `oauth.ts`, `servers.ts` |
| 백그라운드 잡 큐 | `lib/dbq/`, `jobs/registry.ts`, `workers/` |
| 문서 파싱/변환 | `lib/pdfText.ts`, `lib/officeText.ts`, `lib/spreadsheet.ts`, `lib/convert.ts` |
| 에러/로그 위생 | `lib/httpError.ts`, `lib/safeError.ts`, `frontend/src/app/lib/userFacingError.ts` |

### React나 PHP로 만들 수 있는가

- **React: 이미 React 기반**입니다. `frontend/`이 Next.js App Router + TypeScript +
  Tailwind + shadcn 구성이라 프론트엔드를 다시 만들 필요가 없습니다.
- **PHP: 이론상 가능하지만 권장하지 않습니다.** Vercel AI SDK(스트리밍·툴 호출),
  `@modelcontextprotocol/sdk`, `pdfjs-dist`/`mammoth`/`docx` 등 Node 생태계 의존도가
  높고, PHP-FPM 환경에서 SSE 스트리밍이 까다롭습니다. 약 24.7만 줄의 도메인 로직
  재작성 비용도 큽니다.
- **권장 절충안**: 프론트/백엔드는 현 스택 유지, 기존 PHP 시스템(워드프레스, 사내
  ERP 등)은 Mike의 REST API를 호출하는 **얇은 어댑터**로 연동.

---

## 6. 라이선스 제약 (수익화 전 필수 확인)

Mike는 **AGPL-3.0-only**입니다. GPL보다 강한 네트워크 조항이 포함됩니다.

| 사용 형태 | 수정 소스 공개 의무 |
| --- | --- |
| 사내/개인용으로만 사용 (배포 없음) | 없음 |
| 수정해서 웹 서비스로 제공(SaaS) | 있음 |
| 바이너리/파일 형태로 배포·판매 | 있음 |
| 독립 프로그램에서 Mike API만 호출 | 사안별 판단(일반적으로 분리 가능) |

핵심은 "코드를 팔 수 없다"가 아니라 **"코드를 숨길 수 없다"** 입니다. 따라서
코드 독점이 아닌 가치(서비스, 운영, 도메인 지식, 데이터/템플릿, 교육)로 수익을
만드는 모델이 안전합니다. 실제 사업화 전에는 변호사 검토를 권장합니다.

---

## 7. 활용 및 수익화 아이디어

### 7.1 국내 법무 AI 구축 파트너 (현실성 최상)

한국 시장에는 동급 오픈소스 대안이 사실상 없고, 국내 판례·법령은 CourtListener에
없으므로 **로컬라이징 자체가 진입장벽이자 차별점**이 됩니다.

필요한 작업:

- 한국어 UI/i18n 정리
- 판례·법령 소스 교체: `lib/courtlistener.ts`를 참고해 국가법령정보/대법원
  종합법률정보 연동 모듈(예: `lib/korLaw.ts`) 추가
- 국내 계약 유형별 워크플로우 템플릿(근로계약, NDA, 임대차, 하도급, 이용약관 등)
- 온프레미스 설치 옵션 (Ollama 기반 완전 사내망 구동)

수익 구조 예시:

| 항목 | 가격대(예시) |
| --- | --- |
| 초기 구축(온프레미스) | 1,500만 ~ 5,000만원 |
| 연 유지보수 | 구축비의 15~20% |
| 커스텀 워크플로우 개발 | 건당 300~800만원 |
| 교육/온보딩 | 회당 100~300만원 |

타겟: 중소 로펌, 기업 법무팀, 노무·세무 사무소, 특허법인.

### 7.2 버티컬 전환 (법률 외 산업)

Mike의 본질은 "문서 대량 업로드 → 항목 추출 → 표 정리 → 근거 인용"입니다.
이 패턴이 필요한 산업은 많습니다.

| 산업 | 유스케이스 | 추출 항목 예시 |
| --- | --- | --- |
| 건설/조달 | 입찰공고·시방서 검토 | 공사기간, 하자보증, 지체상금, 자격요건 |
| 의료/제약 | 임상시험 문서, IRB | 프로토콜 번호, 피험자 수, 이상반응 |
| 보험 | 약관·손해사정 | 보장범위, 면책조항, 자기부담금 |
| 부동산 | 등기부·임대차계약 | 근저당, 임차인, 특약, 갱신조건 |
| 세무/회계 | 세무조사 대응 | 과세연도, 쟁점, 근거 법령 |
| 제조 | ESG/공급망 실사 | 인증 만료일, 유해물질, 감사 결과 |
| 연구 | 논문 스크리닝 | 표본 수, 방법론, 결론, 한계 |
| 공공 | RFP 분석, 규제 대응 | 제출기한, 평가배점, 필수요건 |

우선 진입 추천: **건설 입찰 검토** — 문서가 길고, 누락 리스크가 크고, 마감이
촉박하며, 담당자의 지불 의향이 높습니다.

### 7.3 워크플로우 / 애드온 마켓플레이스

Mike는 워크플로우를 별도 저장소에서 동기화하고, 애드온·기여 제출 테이블까지
갖추고 있어 생태계형 판매에 적합합니다. 워크플로우(프롬프트 + 추출 스키마)는
Mike 코드의 파생물이 아니므로 **독점 판매가 가능**합니다.

- 한국형 계약 검토 팩: 월 구독
- 산업별 전문 팩: 팩당 판매
- 마켓플레이스 운영 수수료

### 7.4 매니지드 호스팅 SaaS

```
Free : 문서 20개, Ollama 모델, 제한된 기능
Pro  : 월 4.9만원/인 — 문서 500개, 상용 모델, Word 애드인
Team : 월 9.9만원/인 — 조직 관리, 감사 로그, SSO, 공유
Enterprise : 연 계약 — 온프레미스, 전용 모델, SLA
```

`user_api_keys` 테이블과 `USER_API_KEYS_ENCRYPTION_SECRET`이 이미 있으므로
**BYOK(고객 자체 API 키)** 모델을 쓰면 LLM 원가 부담 없이 플랫폼 수수료만
받는 구조를 만들 수 있습니다. AGPL 준수를 위해 소스 링크와 수정 사항을
공개하되, Mike는 이미 공개 코드이므로 실질 부담은 작습니다.

### 7.5 콘텐츠 및 교육 (초기 자본 불필요)

- 기술 콘텐츠(유튜브/블로그): "오픈소스 법률 AI 구조 분해" 시리즈
- 유료 강의: AI 문서 SaaS 아키텍처 실전 (Mike를 교재로)
- 보일러플레이트 + 유료 지원 패키지 (파생물은 AGPL 유지)
- 아키텍처 컨설팅 세션

### 7.6 Word 애드인 역량 상품화

`word-addin/`은 독립적 가치가 큰 자산입니다(Office.js 작업창, OAuth 다이얼로그,
문서 편집 툴 연동). 애드인 개발 대행, 보일러플레이트, 자체 구독 제품으로 전개
가능합니다. 완전 독점 제품을 원하면 애드인은 새로 구현하고 패턴만 참고하는 편이
라이선스상 안전합니다.

### 7.7 90일 실행 로드맵

| 기간 | 할 일 |
| --- | --- |
| 1~2주 | Docker로 전체 스택 구동, 전 기능 체험, 한국 계약서로 Tabular Review 테스트 |
| 3~4주 | 한국어 UI 정리 + 국내 법령/판례 API 연동 PoC |
| 5~8주 | 한국형 워크플로우 5종 제작, 데모 영상 제작 |
| 9~12주 | 중소 로펌·법무팀 아웃바운드, 무료 PoC 3곳, 첫 유료 구축 계약 |
| 병행 | 개발 과정 콘텐츠화로 인바운드 리드 확보 |

---

## 8. 요약

- Mike는 플러그인·스킬·MCP 서버가 아니라 **완성형 오픈소스 법률 AI 플랫폼**이며,
  외부 MCP 서버를 소비하는 **MCP 클라이언트** 기능을 갖고 있습니다.
- LLM 키 1개(또는 Ollama)와 자체 생성 시크릿만 준비하면 Docker로 전체 스택을
  단일 명령으로 구동할 수 있습니다.
- 로컬 에이전트를 만들려는 관점에서는 프로바이더 추상화, 툴 디스패처, SSE
  스트리밍, 장기 기억, DB 기반 작업 큐가 가장 가치 있는 참고 자산입니다.
- 프론트엔드는 이미 React(Next.js)이므로 재작성이 불필요하고, PHP 재작성은
  비용·생태계 측면에서 비권장이며 API 연동이 합리적입니다.
- 수익화는 AGPL 특성상 "코드 독점"이 아닌 **구축·운영·도메인 지식·템플릿·교육**
  중심 모델이 적합하며, 국내 로컬라이징과 비법률 버티컬 전환이 가장 유망합니다.
