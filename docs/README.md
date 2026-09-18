# Documentation

## Run and deploy Mike

- [Local development](local-development.md) — Docker Compose, local services,
  registration, Ollama, and first-run setup
- [Manual and production deployment](deployment.md) — managed infrastructure,
  environment variables, database upgrades, and deployment safety
- [Troubleshooting](troubleshooting.md) — common local and production problems
- [Safe local testing](safe-local-testing.md) — disposable resources, synthetic
  documents, and secret handling

## Features and clients

- [Scoped memory](memory.md) — app and project Markdown memory, permissions,
  asynchronous learning, deletion, and operations
- [CourtListener integration](courtlistener.md) — live US case-law tools and
  optional bulk data
- [Microsoft Word add-in](../word-addin/README.md) — concise setup and command
  reference
- [Word add-in development and deployment](word-addin-development.md) — manual
  setup, sideloading, builds, storage behavior, testing, and troubleshooting
- [Tamper-evident exports](tamper-evident-exports.md) — document hashes and
  optional signed manifests

## Backend

- [Backend architecture](backend-architecture.md) — domain modules over a
  shared kernel: module anatomy, the service contract, the layering rules, and
  the fitness test that enforces them

## Frontend

- [Design system](design-system.md) — color/typography/spacing tokens, the shared
  `components/ui` primitives, and the accessibility baseline

## Testing and CI

- [End-to-end tests in CI](e2e-ci.md)
- [Backend unit-test coverage](testing-coverage.md)
- [Frontend unit-test coverage](frontend-testing.md)
- [Mutation testing and the SSE load harness](test-depth.md)

## Orientation notes

- [Mike 프로젝트 분석 및 활용 가이드 (한국어)](mike-analysis-ko.md) — 저장소
  구조, 설치 절차, 자주 나오는 질문, 로컬 에이전트 참고 지점, 라이선스 제약과
  활용/수익화 방향을 정리한 한국어 분석 노트

## Historical design and investigation notes

These files preserve the context of completed work. They are not current setup
or architecture guidance.

- [Legal workflows design spec](superpowers/specs/2026-06-29-legal-workflows-design.md)
- [Word add-in assistant scroll-jump report](word-addin-chat-scroll-report.md)

Contribution and disclosure policies live in [CONTRIBUTING.md](../CONTRIBUTING.md)
and [SECURITY.md](../SECURITY.md).
