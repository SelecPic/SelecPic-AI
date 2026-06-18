# 프로젝트 구조

SelecPic-AI 저장소의 폴더 구조와 각 모듈의 책임을 정의한다.
상위 방향은 [architecture/pipeline-overview.md](../architecture/pipeline-overview.md)를 따른다.

## 저장소 최상위

```text
SelecPic-AI/
  app/                     # FastAPI 애플리케이션
  docs/                    # 설계·결정·컨벤션 문서
  .vscode/                 # 공유 에디터 설정 (format on save 등)
  .pre-commit-config.yaml  # 커밋 전 ruff 자동 실행
  .gitignore
  requirements.txt
  README.md
```

## 앱 내부 (`app/`)

```text
app/
  main.py              # FastAPI 앱 진입점, lifespan, 라우터 등록
  api/                 # HTTP 엔드포인트 정의
    classification.py  # 얼굴 분류 API 라우터
    health.py          # 헬스체크 엔드포인트
  core/                # 설정, 의존성 주입
    config.py          # 환경변수 기반 설정 (pydantic-settings)
    deps.py            # FastAPI 의존성 함수
  pipeline/            # AI 파이프라인 로직
    detect.py          # YuNet 얼굴 감지
    align.py           # face_align 직접 구현 (Umeyama)
    embed.py           # AuraFace 임베딩
    cluster.py         # HDBSCAN 클러스터링
  schemas/             # Pydantic 요청/응답 스키마
    classification.py  # ClassificationRequest, ClassificationResult
```

## 레이어 의존 방향

```text
api ──▶ pipeline ──▶ (모델 파일, 외부: S3, pgvector)
  │         ▲
  └──▶ schemas, core ◀──┘
```

- `api`는 `pipeline`과 `schemas`에 의존한다.
- `pipeline` 모듈들은 서로 순서대로 의존한다: `detect → align → embed → cluster`.
- `core`는 어느 레이어에서도 참조 가능하다.
- `schemas`는 `pipeline`에 의존하지 않는다.

## 모듈별 책임

| 모듈 | 책임 |
|------|------|
| `api/` | HTTP 요청 수신, 응답 반환. 비즈니스 로직 없음 |
| `core/config.py` | 환경변수 파싱. `Settings` 클래스 하나만 존재 |
| `pipeline/detect.py` | YuNet 모델 로딩 및 얼굴 감지 실행 |
| `pipeline/align.py` | Umeyama 변환 계산 및 112×112 정렬 이미지 생성 |
| `pipeline/embed.py` | AuraFace 모델 로딩 및 512-dim 벡터 생성 |
| `pipeline/cluster.py` | HDBSCAN 실행, 클러스터 레이블 반환 |
| `schemas/` | API 입출력 데이터 형식 정의 (Pydantic v2) |

## 명명

폴더·파일·클래스 명명 규칙은 [code-style.md](./code-style.md)를 따른다.
라우터 파일은 리소스 이름(`classification.py`, `health.py`),
파이프라인 파일은 동작 이름(`detect.py`, `align.py`, `embed.py`, `cluster.py`)을 쓴다.
