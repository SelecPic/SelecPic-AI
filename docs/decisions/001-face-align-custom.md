# ADR 001: face_align을 직접 구현한다

## Status

Accepted

## Context

AuraFace 임베딩 모델은 112×112 크기로 ArcFace 기준점에 정렬된 얼굴 이미지를
입력으로 요구한다. 이 정렬 작업(`face_align`)을 외부 라이브러리로 처리하는
방법과 직접 구현하는 방법 중 선택해야 했다.

외부 옵션으로는 `insightface.utils.face_align`이 있었다. 이 함수는 내부적으로
`skimage.transform.SimilarityTransform`을 사용해 Umeyama 유사변환을 계산한다.

## Decision

`_umeyama()` 함수와 `_ARCFACE_DST` 상수를 코드 내에 직접 구현한다.
insightface와 skimage에 의존하지 않는다.

## Rationale

- **의존성 제거**: insightface는 설치 크기가 크고 빌드 환경에 따라 설치 실패
  가능성이 있다. skimage도 불필요한 의존성이다. OpenCV + NumPy만으로 완결된다.
- **단순성**: image_size=112 고정이므로 분기 로직이 필요 없다. 30줄 닫힌형식
  수식으로 완결된다.
- **검증 완료**: 변환행렬 `np.allclose=True`, 픽셀 차이 0으로 기존 insightface
  구현과 동등성을 검증했다.

## Consequences

- `requirements.txt`에서 insightface, skimage를 제거한다.
- `app/pipeline/align.py`에 `_umeyama()`와 `_ARCFACE_DST`를 직접 유지한다.
- Umeyama 수식은 수학적 불변식이므로 예외적으로 설명 주석을 허용한다.
- 향후 image_size 변경이 생기면 `_ARCFACE_DST` 기준점과 로직을 함께 검토해야 한다.
