# 치즈모아 AI 서버 기능명세서

> **한 줄 요약** — 이 서버는 RabbitMQ로 받은 사진 묶음을 얼굴 인식 파이프라인으로 처리해, 인물별 클러스터 결과를 결과 큐로 돌려주는 **비동기 AI 워커**다.

| 항목 | 내용 |
|------|------|
| 레포 | SelecPic-AI (FastAPI 기반 AI 추론 서버) |
| 연동 | RabbitMQ 비동기 단일 경로 (Spring producer → AI consumer → 결과 큐) |
| 실행 구조 | 별도 워커 프로세스(consumer) + FastAPI(헬스 체크 전용) |
| 핵심 알고리즘 | 대표벡터 기반 증분 클러스터링 |
| 저장소 | pgvector (개별 얼굴 임베딩 + 클러스터 대표벡터). Milvus/Qdrant는 향후 확장 옵션 |

---

## 1. 서버의 책임 범위

| 한다 ✅ | 하지 않는다 ❌ |
|---------|----------------|
| 얼굴 감지·정렬·임베딩·클러스터링 | 사용자 인증 (Spring) |
| 증분 클러스터링 + 대표벡터 갱신 | 이미지 업로드·저장 (S3는 Spring) |
| 개별 얼굴 임베딩 + 대표벡터 저장·조회 (pgvector) | 앨범 메타데이터·공개 상태 관리 (PostgreSQL은 Spring) |
| 사용자 보정(병합/분리/이동)을 대표벡터에 반영 | 사람의 검토(미검토/검토완료) 상태 관리 |
| 분류 결과 발행 (RabbitMQ 결과 큐) | 결제·권한·모임 운영 |

---

## 2. 연동 구조 (RabbitMQ)

```text
Spring (producer)
   │  ① 요청 큐에 분류 작업 발행
   │  ①' 사용자 수동 보정(병합/분리/이동)을 보정 큐에 발행
   ▼
[요청 큐] classify.request     [보정 큐] cluster.feedback
   │                                 │
   ▼                                 ▼
치즈모아 AI 워커 (consumer)  ── 처리 실패 ──▶ [DLQ] classify.dlq
   │  ② AI 파이프라인 실행 / 대표벡터 갱신
   │  ③ 결과 큐에 발행
   ▼
[결과 큐] classify.result
   │
   ▼
Spring (consumer)  ④ 결과 구독 → DB 반영
```

- 분류 요청은 **HTTP로 받지 않는다.** 오직 메시지 큐로만 수신한다.
- AI 워커는 **두 종류의 입력 메시지**를 소비한다: 분류 작업(`classify.request`)과 사용자 보정(`cluster.feedback`).
- HTTP는 운영용 헬스 체크 전용이다(8장).

---

## 3. 실행 구조

| 컴포넌트 | 역할 | 진입점 |
|----------|------|--------|
| **AI 워커** | RabbitMQ consumer. 파이프라인 실행의 본체 | `app/worker.py` |
| **FastAPI 앱** | 헬스/레디니스 체크만 제공 | `app/main.py` |
| **공통 파이프라인** | 두 컴포넌트가 공유하는 AI 로직 | `app/pipeline/*` |

> 워커와 FastAPI는 **같은 레포·같은 코드**를 공유하고 배포 단위만 분리한다.
> AI 추론은 CPU/GPU 바운드라 HTTP 이벤트 루프와 분리해 블로킹·장애 격리를 확보한다.

---

## 4. 핵심 알고리즘 — 대표벡터 기반 증분 클러스터링

새 사진이 들어올 때마다 전체를 다시 군집화하지 않고, **기존 클러스터의 대표벡터에 신규분을 붙이는** 증분 방식.

```text
① 신규 사진 임베딩 생성        detect → align → embed (512-dim)
        │
② 신규 임베딩끼리 클러스터링    HDBSCAN
        │   → 신규 클러스터 N개
        │   → 각 클러스터 대표벡터 = 멤버 임베딩의 L2 정규화 평균
        ▼
③ 기존 대표벡터와 매칭          신규 대표벡터 ↔ 기존 대표벡터 코사인 유사도 비교
        │   ├─ 임계값 이내  → 가장 유사한 기존 클러스터에 덩어리째 병합
        │   └─ 임계값 밖    → 신규 인물 클러스터 생성
        ▼
④ 대표벡터 재계산·갱신          병합된 클러스터의 대표벡터를 다시 정규화 평균으로 갱신 → pgvector 저장
```

**대표벡터 정의**: 클러스터 멤버 임베딩들의 **L2 정규화 평균(centroid)**.
- 얼굴 인식 표준 방식이며 코사인 비교와 수학적으로 일관됨.
- 증분 갱신(running mean)이 저렴해 이 구조와 잘 맞음.
- (대안: 이상치 강건성이 필요하면 medoid 검토 — 현재는 채택 안 함)

---

## 5. 저장소 책임 분리

```text
AI 서버 (pgvector)              Spring (PostgreSQL)
─────────────────              ───────────────────
• 개별 얼굴 임베딩 전부     ↔   • 대표벡터 사본 (앱 조회용)
• 클러스터별 대표벡터            • 앨범·인물·공개상태 등 메타데이터
• 군집·재군집·대표벡터 재계산     • 결과 메시지로 대표벡터 수신·저장
```

- **AI 서버 pgvector** = 군집 계산의 원천. 개별 임베딩을 보관해야 향후 재군집·분리·대표벡터 재계산이 가능.
- **Spring Postgres** = 앱이 쓰는 대표벡터 + 비즈니스 메타데이터. 결과 큐로 받아 저장.
- 대표벡터가 양쪽에 있는 건 용도가 달라서 의도된 것(AI=계산용 원천 / Spring=서비스 조회용).
- **인물 식별의 연속성은 AI 몫**: AI가 대표벡터 매칭으로 같은 사람을 업로드 간 **같은 `cluster_id`로 유지**해 보낸다. Spring은 `cluster_id` ↔ 이름(예: "민준") 매핑만 담당하며 **AI는 이름을 모른다**. 즉 사진을 사람별로 묶는 일은 AI가 끝내고, Spring은 번호표에 이름표만 붙인다(Spring이 다시 군집하지 않음).
- **저장소 선택 근거**: 저장 벡터 규모(모임당 수만 건, 모임 단위 격리 쿼리)는 pgvector(HNSW/IVFFlat)로 충분. Milvus/Qdrant는 수억 건·고QPS 단계에서 검토. 저장소 접근은 인터페이스로 추상화해 교체 여지를 남긴다.

---

## 6. 메시지 스키마

### 6.1 요청 메시지 (classify.request)

```jsonc
{
  "job_id": "uuid",          // 작업 식별자 (멱등 처리 키)
  "group_id": "uuid",        // 모임 ID (대표벡터 격리 단위)
  "event_id": "uuid",        // 이벤트(촬영 단위) ID
  "images": [
    { "image_id": "uuid", "s3_key": "string" }
  ]
}
```

> 증분/최초 분석을 별도 플래그로 구분하지 않는다. 서버가 `group_id`의 기존 대표벡터
> 존재 여부로 판단한다(없으면 최초 군집, 있으면 증분 매칭).

### 6.2 결과 메시지 (classify.result)

```jsonc
{
  "job_id": "uuid",
  "status": "succeeded",     // "succeeded" | "partial" | "failed"
  "clusters": [
    {
      "cluster_id": "uuid",      // 대표벡터와 1:1 (신규/기존 모두)
      "is_new": true,            // 이번에 새로 생긴 인물인지
      "image_ids": ["uuid"]
    }
  ],
  "common_album": ["uuid"],     // 단체·배경 사진 (특정 인물 귀속 불가)
  "hard_to_classify": [         // "분류가 어려워요" 앨범 — 사진별 사유 태그
    { "image_id": "uuid", "reason": "ambiguous" }
  ],
  "failed_images": [            // 기술적 분석 실패 (재시도 대상, 위와 별개)
    { "image_id": "uuid", "reason": "timeout" }
  ]
}
```

- `clusters`: 자신 있게 인물로 묶인 결과. **AI는 신뢰도 등급(`tier`)을 부여하지 않는다** — 사람의 `미검토/검토완료`는 Spring/앱의 상태이지 AI 출력이 아니다.
- `hard_to_classify`: Figma "분류가 어려워요" 앨범에 대응. 사진별 `reason`:
  - `eyes_closed` (눈감음) · `blurry` (흔들림) · `ambiguous` (애매 = 인물 매칭 저신뢰)
  - (TBD: `no_face`(얼굴없음)·`back`(뒷모습)·`duplicate`(중복) 추가 여부는 백엔드와 합의)
- `failed_images`: 타임아웃 등 **기술적 실패**. 화질·매칭 문제인 `hard_to_classify`와 구분한다.

### 6.3 보정 피드백 메시지 (cluster.feedback)

사용자가 앱에서 인물을 병합·분리하거나 사진을 다른 인물로 옮기면, Spring이 그 사실을 발행한다.
AI 워커는 이를 받아 **pgvector의 멤버십·대표벡터를 갱신**해 다음 증분 매칭의 정확도를 유지한다.

```jsonc
{
  "group_id": "uuid",
  "action": "merge",   // "merge" | "split" | "reassign"

  // action="merge": 여러 클러스터를 하나로
  "merge":    { "target_cluster_id": "uuid", "source_cluster_ids": ["uuid"] },
  // action="split": 한 클러스터를 사용자 지정 그룹으로 분리
  "split":    { "cluster_id": "uuid", "groups": [["image_id"], ["image_id"]] },
  // action="reassign": 특정 사진을 다른 인물로 이동
  "reassign": { "image_id": "uuid", "from_cluster_id": "uuid", "to_cluster_id": "uuid" }
}
```

> 처리 결과(갱신된 대표벡터/클러스터)는 동일하게 `classify.result` 형식으로 결과 큐에 발행해 Spring이 동기화한다.

---

## 7. 정책 반영 (치즈모아 정책 → 서버 동작)

이 서버의 출력은 치즈모아 서비스 정책을 따른다. **정책 위반 출력은 백엔드가 거부**하므로 필수.

| 정책 | 서버 동작 |
|------|-----------|
| 저신뢰 매칭 사진 | 인물에 자신 있게 못 붙인 사진은 `hard_to_classify`에 `reason: ambiguous`로. **매칭 임계값은 설정값(`core/config.py`), 하드코딩 금지** |
| "분류가 어려워요" + 자동삭제 금지 | 눈감음·흔들림·애매 사진은 `hard_to_classify`에 사유 태그로만 담음. 삭제는 사용자/백엔드 몫 |
| 단체·배경 사진 = 공통 사진첩 | 특정 인물 귀속 불가 사진은 `common_album`으로 |
| 분석 중 비노출 | `status: succeeded`일 때만 결과 발행. 부분 결과 노출 금지 |
| 새 인물 = 임시 라벨 | 서버는 `cluster_id`+`is_new`만 부여. 이름 지정은 사용자 |
| 병합/분리/이동 (사용자 보정) | `cluster.feedback`로 받아 대표벡터·멤버십 갱신 → 다음 매칭 정확도 유지 (6.3) |
| 분석 실패 / 타임아웃 | 이미지 단위 처리, 실패분은 `failed_images`로. 메시지 단위 재처리는 DLQ로(3회) |
| 원본 무손실 | 서버는 원본을 변형·삭제하지 않음. 산출물만 생성 |
| 증분 분석 | 기존 클러스터 유지 + 신규분만 대표벡터 매칭으로 병합 (4장) |
| 새벽 배치 (대표벡터 재계산·재군집) | **여부·방식 미정(TBD)** — 정책상 언급되나 아직 확정 안 됨. 개별 임베딩 보관으로 향후 구현 여지만 확보 |
| 얼굴 임베딩 = 생체정보 | 대표벡터 pgvector 저장 시 암호화·격리 전제. 보존기간·파기 TBD |
| 전원 동의 전제 | 미동의자 차단은 백엔드/계약에서 처리 → **얼굴 마스킹 로직 MVP 불필요** |

---

## 8. 운영 (HTTP 엔드포인트)

| Method | Path | 설명 |
|--------|------|------|
| GET | `/health` | 프로세스 생존 확인. 즉시 200 |
| GET | `/health/ready` | 모델(YuNet·AuraFace) 로딩 + RabbitMQ·pgvector 연결 확인 |

---

## 9. 예외 처리

- **멱등성**: 동일 `job_id` 재수신 시 중복 분석 방지.
- **재시도**: 메시지 처리 실패는 재큐잉, 3회 초과 시 **DLQ(`classify.dlq`)**로 격리.
- **부분 실패**: 일부 이미지만 실패하면 작업 전체를 죽이지 않고 `failed_images` + `status: partial`.
- **외부 의존 장애**(S3·pgvector·모델): 원본 무손실 우선. 임베딩·산출물은 재생성 가능 자원으로 취급.

---

## 10. 미정 항목 (TBD)

| # | 항목 | 비고 |
|---|------|------|
| 1 | **새벽 배치 (대표벡터 재계산·재군집)** | 여부·주기·방식 모두 미정 |
| 2 | `hard_to_classify` 사유 태그 범위 | 눈감음·흔들림·애매 외 얼굴없음·뒷모습·중복 추가 여부 |
| 3 | 저신뢰(애매) 판정 임계값 | 인물 매칭을 `ambiguous`로 떨굴 기준 |
| 4 | 증분 매칭 임계값 | 신규↔기존 대표벡터 병합 기준 코사인 거리 |
| 5 | `split` 보정 처리 방식 | 사용자 지정 그룹대로 분리 vs 해당 클러스터 재군집 |
| 6 | 얼굴 임베딩 보존기간·파기 시점 | 생체정보 정책 |
| 7 | RabbitMQ 큐·라우팅 키 네이밍 | `classify.request`·`cluster.feedback` 등은 임시안 |

---

## 관련 문서

- 시스템 구조: [architecture/system-overview.md](../architecture/system-overview.md)
- 파이프라인: [architecture/pipeline-overview.md](../architecture/pipeline-overview.md)
- 폴더 구조: [conventions/project-structure.md](../conventions/project-structure.md)
