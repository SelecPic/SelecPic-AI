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
Message Queue (RabbitMQ) ─────────────┘
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

Spring 백엔드와 이 서버는 **RabbitMQ 메시지 큐로만** 연동한다(비동기 단일 경로).

- Spring이 **요청 큐**에 분류 작업 메시지를 발행한다(producer).
- 이 서버가 **consumer**로 메시지를 소비해 파이프라인을 실행한다.
- 완료 후 **결과 큐**에 분류 결과를 발행하고, Spring이 이를 구독한다.

HTTP 엔드포인트는 운영용 헬스 체크(`/health`)에만 사용하며, 분류 요청은 받지 않는다.

## 데이터 흐름

1. Spring이 S3에 원본 이미지를 업로드하고 S3 경로(또는 Presigned URL)를 RabbitMQ 요청 큐에 발행한다.
2. 이 서버의 consumer가 메시지를 수신해 AI 파이프라인을 실행한다.
3. 파이프라인 완료 후 클러스터 결과(인물 클러스터 → 이미지 목록 매핑)를 RabbitMQ 결과 큐에 발행한다.
4. 클러스터별 대표벡터를 pgvector에 저장·갱신해 향후 증분 분류에 재사용한다.

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
