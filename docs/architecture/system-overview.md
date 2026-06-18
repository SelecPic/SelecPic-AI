# 시스템 구조

SelecPic-AI는 전체 SelecPic 시스템에서 AI 추론만 담당하는 워커 서버다.

## 전체 구조

```text
Flutter App
  │
  ▼
Spring API Server ─────────── PostgreSQL (메타데이터)
  │                                 ▲
  │ Presigned URL 발급               │ 분류 결과 저장
  ▼                                 │
AWS S3 (원본 이미지 저장)             │
  │                                 │
  ▼                                 │
Message Queue (Kafka / Redis) ───────┘
  │
  ▼
[이 서버] SelecPic-AI (FastAPI)
  │
  ├── YuNet (얼굴 감지)
  ├── face_align (Umeyama 유사변환)
  ├── AuraFace (임베딩)
  └── HDBSCAN (클러스터링)
       │
       ▼
  pgvector / FAISS (임베딩 저장소)
```

## Spring 백엔드와의 연동

Spring 백엔드는 `AiClassificationPort` → `FastApiAdapter`를 통해 이 서버를
호출한다. 이 서버는 두 가지 방식으로 작업을 받는다.

- **메시지 큐(Kafka/Redis)**: 비동기 대량 처리 (주요 경로)
- **REST API 직접 호출**: 동기 처리가 필요한 경우

## 데이터 흐름

1. Spring이 S3에 원본 이미지를 업로드하고 Presigned URL 또는 S3 경로를 메시지 큐에 발행한다.
2. 이 서버가 메시지를 수신해 AI 파이프라인을 실행한다.
3. 파이프라인 완료 후 클러스터 결과(인물 ID → 이미지 목록 매핑)를 Spring에 반환한다.
4. 생성된 임베딩 벡터는 pgvector에 저장해 향후 증분 분류에 재사용한다.

## AI 서버의 역할 범위

이 서버가 하는 것:

- 얼굴 감지 / 정렬 / 임베딩 / 클러스터링
- 임베딩 벡터 저장 (pgvector)

이 서버가 하지 않는 것:

- 사용자 인증
- 이미지 업로드 / 저장 (S3는 Spring이 관리)
- 앨범 메타데이터 관리 (PostgreSQL은 Spring이 관리)

## 관련 문서

- AI 파이프라인 상세: [pipeline-overview.md](./pipeline-overview.md)
