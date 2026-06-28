# SelecPic-AI

SelecPic의 AI 추론 서버다. Spring 백엔드가 RabbitMQ에 발행한 작업을 consumer로 받아
얼굴 감지 → 정렬 → 임베딩 → 클러스터링 파이프라인을 실행하고 결과를 결과 큐에 발행한다.

자세한 파이프라인 설계는 [docs/architecture/pipeline-overview.md](docs/architecture/pipeline-overview.md) 참고.

## 구성

```text
SelecPic-AI/
  app/     # FastAPI 애플리케이션
  docs/    # 설계·결정·컨벤션 문서
```

## 서버 실행

```sh
# 가상환경 활성화 (Windows)
.venv\Scripts\activate

# macOS / Linux
source .venv/bin/activate

pip install -r requirements.txt
uvicorn app.main:app --reload
```

기본 포트: `http://localhost:8000`
API 문서: `http://localhost:8000/docs`

## 개발 규칙

- 코드 스타일: [docs/conventions/code-style.md](docs/conventions/code-style.md)
- 폴더 구조: [docs/conventions/project-structure.md](docs/conventions/project-structure.md)

VSCode에서 Ruff 확장을 설치하면 저장 시 자동으로 포맷 + 린트가 적용된다.
커밋 전 pre-commit 훅도 동일하게 동작한다 (`pre-commit install`로 등록).

## 검증

```sh
ruff format --check .
ruff check .
```
