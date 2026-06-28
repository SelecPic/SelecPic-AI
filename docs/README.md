# SelecPic-AI 문서

SelecPic AI 서버 저장소의 문서 인덱스다.

## architecture — 설계

- [pipeline-overview.md](./architecture/pipeline-overview.md) — AI 파이프라인 전체 흐름과 각 단계 설명
- [system-overview.md](./architecture/system-overview.md) — 전체 시스템 구조와 Spring 백엔드 연동

## spec — 기능 명세

- [feature-spec.md](./spec/feature-spec.md) — 치즈모아 AI 서버 기능명세서 (RabbitMQ 연동·증분 클러스터링·정책 반영)

## decisions — 결정 기록 (ADR)

- [001-face-align-custom.md](./decisions/001-face-align-custom.md) — face_align 직접 구현 결정
- [002-hdbscan-sklearn.md](./decisions/002-hdbscan-sklearn.md) — HDBSCAN sklearn 라이브러리 유지 결정

## conventions — 개발 규칙

- [code-style.md](./conventions/code-style.md) — Python/FastAPI 코드 컨벤션
- [project-structure.md](./conventions/project-structure.md) — 폴더 구조와 책임
