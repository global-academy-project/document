# Embedding·RAG·Ingest·Qdrant 오류 및 업데이트 기록

- 기록일: 2026-11-04
- 범위: `main-ai-model`의 임베딩·RAG·ingest와 개발 Compose의 Qdrant 연결

## 요약

임베딩 API는 벡터를 반환하는 서비스이며, 직접 Qdrant에 저장하지 않는다. 지식 데이터를 적재할 때는 `kb-ingest`가 요청을 받아 임베딩 API를 호출하고, 반환된 dense/sparse 벡터와 payload를 Qdrant에 upsert한다. RAG 검색은 질의 텍스트를 임베딩한 뒤 Qdrant에서 검색한다.

```text
문서 청크 -> kb-ingest -> Embedding API -> kb-ingest -> Qdrant
사용자 질의 -> RAG backend -> Embedding API -> Qdrant 검색 -> 검색 결과
```

## 오류 내역과 원인

| 관찰된 오류 | 원인 | 조치 또는 확인 방법 |
| --- | --- | --- |
| Embedding API `401 Unauthorized` | `/v1/embeddings`의 Bearer 인증이 없거나 `EMBEDDING_API_KEY`와 불일치 | Swagger의 Authorize 또는 요청의 `Authorization: Bearer <키>`를 확인한다. ingest 인증 토큰과는 별개다. 컨테이너 간 직접 임베딩 요청은 `200`으로 확인했다. |
| 수동 임베딩 요청 `422 JSON decode error` | JSON 배열의 마지막 항목 뒤에 trailing comma가 있었다. | 마지막 항목 뒤 쉼표를 제거한다. |
| `/ingest/chunks` `401 Unauthorized` | ingest API의 `KB_INGEST_API_TOKEN`이 없거나 실행 컨테이너 값과 요청 Bearer 토큰이 일치하지 않는다. | 요청에 `Authorization: Bearer <KB_INGEST_API_TOKEN>`을 보낸다. `EMBEDDING_API_KEY`를 대신 쓰지 않는다. `.env` 변경 후 `kb-ingest`를 재생성한다. |
| `/ingest/chunks` `422 Unprocessable Entity` | 처음에는 `point_id`에 `guide-chunk-1`처럼 Qdrant가 허용하지 않는 문자열을 사용했다. 임베딩 API의 `422`는 별도로 JSON 문법 오류에서 발생했다. | `point_id`에는 UUID 문자열 또는 음이 아닌 정수를 사용한다. API에서 이 형식을 사전 검증한다. |
| Qdrant upsert `404`, ingest 응답 `502 Bad Gateway` | 임베딩 API는 `200`을 반환했지만 Qdrant에 대상 컬렉션이 없었다. Compose 시작만으로 초기화 스크립트가 실행되지는 않았다. | RAG 스키마의 `kb_accidents`, `kb_guides` 컬렉션을 초기화한다. ingest는 내부 오류 상세를 응답에 노출하지 않고 서버 로그에 예외를 남긴다. |
| Qdrant upsert `400` | Qdrant point ID 형식이 유효하지 않은 상태로 upsert를 시도했다. | `point_id` 형식을 ingest 요청 단계에서 검증하도록 변경했다. |

## 코드 및 설정 변경

- `main-ai-model/app/ingest_main.py`
  - `POST /ingest/chunks`를 추가했다.
  - `KB_INGEST_API_TOKEN` Bearer 인증을 적용했다.
  - collection, 청크 수, 텍스트, point ID 및 RAG 필수 payload 필드를 검증한다.
  - `index_chunks()`에 적재를 위임하고 처리한 청크 수를 반환한다.
  - 임베딩·Qdrant 백엔드 오류의 상세 원인은 서버 로그에 기록하고 클라이언트에는 일반화된 `502`를 반환한다.
- `main-ai-model/app/rag/indexing.py`
  - 기존 `index_chunks()`가 청크 임베딩 후 Qdrant에 upsert한다. 요청당 최대 32개 청크를 배치 처리한다.
- `main-ai-model/requirements.ingest.txt`
  - ingest 설정에 필요한 `pydantic-settings`를 추가했다.
- `main-infra-manage/docker/docker-compose.yml`
  - `kb-ingest`에 내부 주소 `http://python-embedding-model:60110/v1/embeddings`, embedding 설정과 인증 키를 전달한다.
  - Qdrant는 Compose 내부 주소 `http://qdrant-server:6333`을 사용한다.
- `main-ai-model/scripts/init_qdrant.py`
  - 기본 Qdrant 주소가 Compose 서비스명 `qdrant-server`를 사용하도록 변경됐다.
- `main-ai-model/tests/test_ingest_api.py`
  - 적재 위임, 인증 실패, 잘못된 point ID 검증을 테스트한다.

## API 사용 계약

호스트에서 호출하는 주소는 `http://localhost:8001/ingest/chunks`다. 인증에는 `.env`의 `KB_INGEST_API_TOKEN`을 사용한다. 요청 예시의 UUID와 payload 값은 예시이므로 실제 원본·청크 ID와 고유 point ID로 바꿔야 한다.

```bash
curl -X POST 'http://localhost:8001/ingest/chunks' \
  -H "Authorization: Bearer ${KB_INGEST_API_TOKEN}" \
  -H 'Content-Type: application/json' \
  -d '{
    "collection": "kb_guides",
    "chunks": [{
      "point_id": "00000000-0000-0000-0000-000000000001",
      "text": "작업 전 위험요인을 확인합니다.",
      "payload": {
        "kb_source_id": 1,
        "kb_chunk_id": 1,
        "source_type": "KOSHA_GUIDE",
        "site_id": 0,
        "flagged": false,
        "title": "안전 가이드"
      }
    }]
  }'
```

성공 응답은 `{"indexed": 1}` 형식이다. `kb_accidents`에는 `ACCIDENT_REPORT`, `SIF`, `NEAR_MISS`; `kb_guides`에는 `KOSHA_GUIDE`, `RISK_GUIDE`, `CHECKLIST`, `LAW` source type을 사용한다.

## Qdrant 초기화와 현재 상태

- `docker compose up --build`는 `scripts/init_qdrant.py`를 자동 실행하지 않는다.
- `init_qdrant.py`는 컬렉션과 payload 인덱스를 만드는 스키마 도구이며, 문서 point를 적재하지 않는다.
- 이번 작업 중 `kb_accidents`, `kb_guides` 및 RAG payload 인덱스를 생성했다. 초기화는 당시 실행 환경에 `qdrant_client`가 없어 Qdrant REST API로 수행했다.
- 실제 ingest 경로를 임시 UUID 청크 하나로 검증했다. 응답은 `200`, `indexed=1`이었으며 점검 후 해당 point를 삭제했다.
- 마지막 확인 시 두 컬렉션은 존재하고 `points_count`는 모두 `0`이다. 실제 지식 데이터는 아직 적재되지 않았다.
- Compose 내부에서는 `qdrant-server:6333`, 호스트에서는 공개 포트 `localhost:60011`을 사용한다.

## 검증 결과 및 후속 작업

- ingest API와 RAG indexing 테스트: `6 passed`.
- Compose 설정 검증과 `kb-ingest` 이미지 빌드: 통과.
- 새 ingest 컨테이너에서 라우트 등록 및 실제 Embedding API → Qdrant 적재 → 임시 point 삭제까지 확인했다.
- 실제 문서 적재 전에 유효한 point ID, DB 원본·청크 ID, collection과 source type을 전달해야 한다.
- 초기화 자동화가 필요하면 별도의 일회성 Compose init 서비스 또는 배포 단계에서 `init_qdrant.py`를 실행하도록 추가해야 한다.
- `.env`의 API 키·토큰·비밀번호는 문서나 요청 예시에 기록하지 않는다. 작업 과정에서 환경 파일 인증정보가 대화에 노출됐으므로 교체가 필요하다.