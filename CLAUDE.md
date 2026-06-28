# SelecPic-AI

## 프로젝트 개요

**SelecPic** (ClassiPic) — "사진이 주인을 찾아가는 AI 공유 앨범"

여러 사람이 함께 촬영한 사진을 AI 얼굴 인식으로 자동 분류·공유하는 서비스. 이 레포는 **FastAPI 기반 AI 추론 서버**로, Spring 백엔드로부터 메시지 큐를 통해 작업을 받아 얼굴 감지 → 정렬 → 임베딩 → 클러스터링 파이프라인을 실행한다.

- GitHub 조직: [SelecPic](https://github.com/orgs/SelecPic/repositories)
- Jira 프로젝트: CLAS (티켓 번호 접두사 `CLAS-XX`)

---

## 전체 시스템 아키텍처

```
Flutter App
  │
  ▼
Spring API Server ──────────── PostgreSQL (Metadata)
  │                                  ▲
  │ Presigned URL 발급                │ 분류 결과 저장
  ▼                                  │
AWS S3 (원본 이미지 저장)              │
  │                                  │
  ▼                                  │
Message Queue (RabbitMQ) ────────────┘
  │
  ▼
[이 서버] FastAPI AI Worker
  │
  ├── Face Detection (YuNet)
  ├── Face Alignment (직접 구현 - Umeyama 유사변환)
  ├── Face Embedding (AuraFace)
  └── Clustering (HDBSCAN)
       │
       ▼
  pgvector / FAISS (임베딩 저장소)
```

Spring 백엔드가 `AiClassificationPort` → `FastApiAdapter`를 통해 이 서버를 호출한다.

---

## AI 파이프라인

### 파이프라인 흐름

```
S3에서 이미지 읽기
  │
  ▼
YuNet (얼굴 감지 + 5점 랜드마크 추출)
  │
  ▼
face_align (직접 구현 — Umeyama 유사변환, 112×112 ArcFace 기준점으로 정렬)
  │
  ▼
AuraFace (512-dim 임베딩 벡터 생성)
  │
  ▼
HDBSCAN (sklearn, eps=0.15, cosine distance — 인물별 클러스터링)
  │
  ▼
클러스터 결과 → Spring 백엔드로 반환 / pgvector 저장
```

### 핵심 설계 결정사항

**face_align — 직접 구현 유지**
- `insightface.utils.face_align` 대신 `_umeyama()` 함수와 `_ARCFACE_DST` 상수를 코드 내 직접 구현
- 외부 의존성 제거 (insightface, skimage 불필요 → OpenCV + numpy만 사용)
- image_size=112 고정이므로 분기 로직 불필요, 30줄 닫힌형식 수식으로 완결
- 변환행렬 `np.allclose=True`, 픽셀 차이 0으로 기존과 동등성 검증 완료

**HDBSCAN — 라이브러리(sklearn) 유지**
- 직접 구현(UnionFind 코사인 임계값) 대비 ARI 0.601 vs 0.219로 정확도 약 2.7배 우수
- 확장성: O(n²) 회피, 가변 밀도 군집 자동 처리
- 클러스터링은 전체 파이프라인 비용 0.1% 미만 → 의존성 유지 부담 없음

---

## 기술 스택

| 역할 | 기술 |
|------|------|
| API 프레임워크 | FastAPI + Uvicorn |
| 얼굴 감지 | YuNet (OpenCV DNN) |
| 얼굴 임베딩 | AuraFace |
| 얼굴 정렬 | 직접 구현 (OpenCV + numpy, Umeyama) |
| 클러스터링 | HDBSCAN (sklearn) |
| 임베딩 저장 | pgvector / FAISS |
| 메시지 큐 연동 | RabbitMQ |
| 데이터 검증 | Pydantic v2 |
| 코드 포맷터 | Ruff |
| 환경 변수 | python-dotenv |

---

## 프로젝트 구조 (목표)

```
SelecPic-AI/
├── app/
│   ├── main.py              # FastAPI 앱 엔트리포인트
│   ├── api/                 # 라우터 (엔드포인트 정의)
│   ├── core/                # 설정, 의존성 주입
│   ├── pipeline/            # AI 파이프라인 로직
│   │   ├── detect.py        # YuNet 얼굴 감지
│   │   ├── align.py         # face_align 직접 구현
│   │   ├── embed.py         # AuraFace 임베딩
│   │   └── cluster.py       # HDBSCAN 클러스터링
│   └── schemas/             # Pydantic 스키마
├── requirements.txt
├── .pre-commit-config.yaml  # ruff linter + formatter
└── .vscode/settings.json    # formatOnSave (ruff)
```

---

## 개발 환경 세팅

```bash
# 가상환경 활성화 (Windows)
.venv\Scripts\activate

# 서버 실행
uvicorn app.main:app --reload

# pre-commit 훅 설치
pip install pre-commit
pre-commit install
```

---

## 코드 컨벤션

### 들여쓰기 / 포맷
- **스페이스 2칸** (AI/Flutter/Web 진영 공통)
- 저장 시 ruff 자동 포맷 (VSCode `formatOnSave: true`)
- 최대 줄 길이: 120자
- pre-commit hook: `ruff --fix` + `ruff-format` 자동 실행

### 네이밍
- 클래스: `PascalCase`
- 함수/변수: `snake_case`
- 상수: `UPPER_SNAKE_CASE`

### 주석
- WHY가 불명확한 경우에만 작성 (숨겨진 제약, 수학적 불변식 등)
- 할 일: `# TODO:` 표시
- `_umeyama()` 같은 수학 로직은 예외적으로 설명 주석 허용

---

## Git 컨벤션

### 브랜치 전략 (Git Flow)
- `main`: 배포 브랜치
- `develop`: 개발 통합 브랜치
- `feature/CLAS-XX-설명`: 기능 개발 브랜치

```bash
# 예시
git checkout -b feature/CLAS-54-face-detection-api
```

### 커밋 메시지
```
[CLAS-XX] type: 메시지
```

| type | 설명 |
|------|------|
| feat | 새로운 기능 |
| fix | 버그 수정 |
| docs | 문서 수정 |
| style | 포맷팅, 공백 등 |
| refactor | 리팩토링 |
| test | 테스트 코드 |
| chore | 빌드, 설정 등 |

```bash
# 예시
git commit -m "[CLAS-54] feat: fastapi 코드 포맷터 적용 및 healthcare api 구현"
```

---

## 현재 상태

- `app/main.py`: 비어있음 (FastAPI 정상 작동 확인만 완료)
- `healthcare_api.py`: FastAPI 학습용 샘플 코드 (실제 프로젝트 코드 아님)
- AI 파이프라인 구현 전 단계

### 다음 구현 목표
1. `app/main.py` — FastAPI 앱 기본 구조 (health check 엔드포인트)
2. `app/pipeline/detect.py` — YuNet 얼굴 감지
3. `app/pipeline/align.py` — face_align 직접 구현 (Umeyama)
4. `app/pipeline/embed.py` — AuraFace 임베딩
5. `app/pipeline/cluster.py` — HDBSCAN 클러스터링
6. `app/api/` — Spring 백엔드 연동 API 엔드포인트
