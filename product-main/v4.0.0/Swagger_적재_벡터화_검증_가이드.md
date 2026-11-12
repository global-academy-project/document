# Swagger에서 지식 데이터 적재·벡터화·검증하기

대상 주소: `http://localhost:8001/docs#/`  
API 서비스: Docker Compose의 `kb-ingest`  
기준 경로: `main-ai-model/out`은 컨테이너의 `/data/out`, `main-ai-model/out/vectors`는 `/data/vectors`로 연결된다.

## 1. 먼저 알아둘 처리 경로

### 권장: 한 번의 요청으로 장부 등록부터 Qdrant 적재까지

`POST /ingest/accidents` 또는 `POST /ingest/guides`를 사용한다.

```text
Swagger 요청
  → 요청 형식·분류 검증
  → MySQL agent_kb 원본·청크 장부 등록
  → 장부와 청크 검증
  → 임베딩 생성
  → Qdrant upsert
  → agent_kb 완료 상태(EMBEDDED) 기록
```

수동으로 한두 건을 시험하거나 정상 적재를 확인할 때는 이 경로를 우선 사용한다. 같은 `source_type`, `external_id`, `chunk_index`로 다시 보내면 UUID v5 point ID 규칙으로 같은 청크를 식별한다. `external_id`를 새 자료에 재사용하지 않는다.

### 기존 청크 파일 사용: 벡터 파일 생성 후 별도 업로드

`POST /ingest/vectors/local/file`은 서버에 마운트된 JSONL 청크 파일을 검증하고, 장부 ID를 연결한 뒤 임베딩을 만들어 JSONL 벡터 파일에 저장한다. 이 요청만으로는 Qdrant에 올리지 않는다. 이어서 `POST /ingest/vectors/local/load`를 호출해야 Qdrant에 업로드된다.

`/ingest/vectors/local/load`는 Qdrant에 upsert하지만 장부의 완료 상태를 기록하지 않는다. 따라서 완전한 업무 흐름의 기본 수단이 아니라, 벡터 파일을 검토한 뒤 별도 적재해야 할 때만 사용한다. 업로드 뒤에도 장부 상태가 `EMBEDDED`가 아닐 수 있다.

## 2. 서비스와 인증 확인

1. 브라우저에서 `http://localhost:8001/ready`를 열어 `{"status":"ok"}` 응답을 확인한다.
2. Swagger UI `http://localhost:8001/docs#/`를 연다.
3. 화면 상단 **Authorize**를 누르고 Compose `.env`의 `KB_INGEST_API_TOKEN` 값을 입력한 뒤 승인한다. `EMBEDDING_API_KEY`나 `QDRANT_API_KEY`가 아니다. Swagger 인증 입력란에는 토큰 값만 넣는다.
4. 잠금 표시가 붙은 적재 API를 실행한다. 인증 토큰은 문서, 화면 캡처, 저장소에 기록하지 않는다.

401은 토큰 누락·불일치, 503은 서비스의 인증·임베딩·Qdrant·장부 설정을 먼저 확인한다.

## 3. 권장 경로로 사고 자료 적재하기

Swagger에서 `POST /ingest/accidents`를 펼치고 **Try it out**을 누른다. 아래 예시는 형식 확인을 위한 가상 1청크 자료다.

```json
{
  "source": {
    "source_type": "SIF",
    "external_id": "SIF-DEMO-001",
    "title": "거푸집 자재반입, 가공, 운반 – 감전 (2018)",
    "url": "https://www.data.go.kr/data/15161362/openapi.do",
    "publisher": "한국산업안전보건공단",
    "license": "이용허락범위 제한 없음",
    "content_sha256": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
    "flagged": false
  },
  "chunks": [
    {
      "chunk_index": 0,
      "text": "[재해개요] 2018년 거푸집 자재 반입·가공·운반 과정에서 감전 사고가 발생했습니다.",
      "embedding_text": "거푸집 자재반입, 가공, 운반 – 감전\n[재해개요] 2018년 거푸집 자재 반입·가공·운반 과정에서 감전 사고가 발생했습니다.",
      "locator": "재해개요·유발요인·감소대책(예시)",
      "token_count": 20,
      "payload": {
        "severity": "FATAL",
        "accident_type": ["ELECTRIC"],
        "hazard": ["ELECTRICAL"],
        "work_type": ["ELECTRICAL"],
        "cas": [],
        "ghs": [],
        "year": 2018,
        "natural_key": {
          "dataset": "sif",
          "record": "SIF-DEMO-001",
          "section": "case",
          "part": 0
        },
        "tier": "B",
        "cate": "건설업",
        "disaster_type": "감전",
        "situation": null
      }
    }
  ]
}
```

**Execute** 후 HTTP `200`과 아래 응답 형태를 확인한다.

```json
{
  "indexed": 1,
  "source_id": 123,
  "chunks": [
    {
      "chunk_index": 0,
      "chunk_id": 456,
      "point_id": "서버가 반환한 UUID"
    }
  ]
}
```

실제 반환 ID는 위 예시와 다르다. `source_id`와 `chunk_id`가 장부 ID이며 `point_id`가 Qdrant point ID다. `indexed`가 보낸 청크 수와 같은지 확인한다.

적재된 SIF 청크의 Qdrant payload에는 요청의 `source` 정보와 `chunks[].payload` 정보가 최상위 필드로 합쳐진다. `natural_key`는 객체로 중첩되며, 장부 연결 ID는 아래처럼 마지막 두 필드에 기록된다.

```json
{
  "source_type": "SIF",
  "site_id": 0,
  "flagged": false,
  "title": "거푸집 자재반입, 가공, 운반 – 감전 (2018)",
  "locator": "재해개요·유발요인·감소대책(예시)",
  "url": "https://www.data.go.kr/data/15161362/openapi.do",
  "text": "[재해개요] 2018년 거푸집 자재 반입·가공·운반 과정에서 감전 사고가 발생했습니다.",
  "severity": "FATAL",
  "accident_type": ["ELECTRIC"],
  "hazard": ["ELECTRICAL"],
  "work_type": ["ELECTRICAL"],
  "cas": [],
  "ghs": [],
  "year": 2018,
  "external_id": "SIF-DEMO-001",
  "chunk_index": 0,
  "natural_key": {
    "dataset": "sif",
    "record": "SIF-DEMO-001",
    "section": "case",
    "part": 0
  },
  "license": "이용허락범위 제한 없음",
  "publisher": "한국산업안전보건공단",
  "content_sha256": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  "tier": "B",
  "cate": "건설업",
  "disaster_type": "감전",
  "situation": null,
  "kb_source_id": 12,
  "kb_chunk_id": 706
}
```

> [!IMPORTANT]
> `kb_source_id`와 `kb_chunk_id`가 맨 아래에 있는 것은 기존 사고 컬렉션 payload의 필드 나열 순서와 맞추기 위한 것입니다. JSON/Qdrant에서 객체 필드 순서는 검색·필터·저장 의미에 영향을 주지 않으므로, 이 순서만으로 데이터가 달라지지는 않습니다. 연동 코드에서는 필드의 위치가 아니라 키 이름으로 값을 읽으세요.

## 4. 권장 경로로 지침·법령 적재하기

Swagger에서 `POST /ingest/guides`를 펼치고 **Try it out**을 누른다.

```json
{
  "source": {
    "source_type": "LAW",
    "external_id": "산업안전보건기준에관한규칙",
    "title": "산업안전보건기준에 관한 규칙",
    "file_path": "raw/산업안전보건기준에 관한 규칙(고용노동부령)(제00450호)(20260302).hwp",
    "flagged": false
  },
  "chunks": [
    {
      "chunk_index": 1,
      "text": "제20조(출입의 금지 등) 사업주는 다음 각 호의 작업 또는 장소에 울타리를 설치하는 등 ...",
      "embedding_text": "산업안전보건기준에 관한 규칙 > 제1편 총칙 > 제2장 작업장 제20조(출입의 금지 등) ...",
      "locator": "제20조 (2/3)",
      "payload": {
        "natural_key": {
          "law_key": "산업안전보건기준에관한규칙",
          "article_key": "20",
          "part": 1
        },
        "article_label": "제20조",
        "article_title": "출입의 금지 등",
        "parts_total": 3,
        "repeated_lead_lines": 2,
        "oversize": false,
        "path": ["제1편 총칙", "제2장 작업장"],
        "hierarchy": {
          "편": "제1편 총칙",
          "장": "제2장 작업장"
        },
        "hazard": [],
        "work_type": [],
        "cas": [],
        "year": 2025,
        "article_sha256": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
        "law_meta": {
          "effective_date": "2026-03-02",
          "promulgation_no": "고용노동부령 제450호",
          "promulgation_date": "2025-09-01",
          "amendment_type": "일부개정",
          "source_sha256": "eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee"
        }
      }
    }
  ]
}
```

`POST /ingest/guides`는 컬렉션을 `kb_guides`로 고정한다. 사고 API는 `kb_accidents`로 고정한다. 각 API의 `source_type`은 사고 API에서 `ACCIDENT_REPORT` 또는 `SIF`, 지침 API에서 `KOSHA_GUIDE`, `RISK_GUIDE`, `CHECKLIST`, `LAW`만 허용한다. `url` 대신 `file_path`를 제공해도 되지만 둘 중 하나는 필요하다.

`LAW` 요청은 기존 법령 청크 포맷을 유지한다. `payload`에 `natural_key`, 조문 정보, 경로·계층, `hazard/work_type/cas/year`, `article_sha256`, `law_meta`가 필요하다. `natural_key.law_key`는 `source.external_id`와 같아야 하고, 같은 요청의 `law_meta.source_sha256` 값은 일치해야 한다. 서버는 이 값을 장부의 원본 해시로 사용하고, payload에 `collection`, `embed_text`, `file_path` 등을 기존 형식대로 채운다. `kb_source_id`와 `kb_chunk_id`는 서버가 장부에서 발급한다.

청크가 여러 개면 `chunks` 배열을 늘리고 `chunk_index`를 서로 다르게 지정한다. 요청당 최대 5,000청크다. 최종 Qdrant payload의 공통 필드(`kb_source_id`, `kb_chunk_id`, `source_type`, `site_id`, `flagged`, `title`, `locator`, `url`, `text`)는 서버가 장부와 요청에서 채운다. 사고 payload는 `severity`, `accident_type`, `hazard`, `work_type`, `cas`, `ghs`, `year`와 SIF 원본 필드(`external_id`, `chunk_index`, `natural_key`, `license`, `publisher`, `content_sha256`, `tier`, `cate`, `disaster_type`, `situation`)로 구성된다. SIF 요청에서는 `natural_key`, `tier`, `cate`, `disaster_type`, `situation`을 제공해야 한다. LAW payload는 앞의 법령 형식처럼 조문 메타데이터와 `law_meta`를 보존한다. 다른 지침 유형은 `hazard`, `work_type`, `cas`, `year`, 선택 필드 `guide_no`를 사용한다. 배열 값은 문자열 배열이어야 하고, 분류 코드는 기존 태그 코드(`ELECTRIC`, `ELECTRICAL`, `HOT` 등)에 맞춰야 한다. 값이 없으면 빈 배열을 사용하고 실제 검색에 활용할 분류값은 원자료에 근거해 넣는다.

각 청크의 `text`는 원문, `embedding_text`는 선택 입력이다. `embedding_text`를 생략하면 `text`를 임베딩한다. 검색 결과에 표시할 원문은 `payload.text`로 서버가 구성하므로 `text`에 정확한 청크 본문을 넣는다.

## 5. 기존 청크 JSONL을 벡터화하고 업로드하기

이 절차는 기존 청크 파이프라인 산출물을 단계별로 확인할 때 사용한다. Swagger가 로컬 PC 파일을 직접 업로드하는 방식이 아니다. 서버 컨테이너에서 접근 가능한 위치에 파일이 있어야 한다. Compose에서는 호스트의 `main-ai-model/out`이 컨테이너 `/data/out`으로 읽기 전용 마운트된다.

### 5-1. 파일과 검증 산출물 준비

- 법령 청크 기본 위치 예: `main-ai-model/out/law/05_chunks.jsonl`
- 법령은 같은 폴더의 `06_validation.json`이 `ok: true`여야 하며 `chunk_index_map.json`도 필요하다.
- SIF 사고 청크 예: `main-ai-model/out/accidents/law/05_chunks.jsonl`. 검증 결과가 있으면 `06_validation.json`도 같은 폴더에 둔다.
- 각 레코드의 필수 형식과 자연 키가 올바른지 먼저 확인한다. 경로 입력에는 파일명만 지정할 수 있으며, 임의 경로는 지정할 수 없다.

### 5-2. 벡터 파일 생성

Swagger에서 `POST /ingest/vectors/local/file`을 실행한다.

기존 `out/law/05_chunks.jsonl` 법령 파일 예:

```json
{
  "dataset_id": "law-demo-2026",
  "chunk_file": "05_chunks.jsonl"
}
```

`data_type`을 생략하면 기본 입력 경로는 컨테이너의 `out/law`다. `LOCAL_CHUNK_ROOT=/data/out`인 Compose 설정에서는 위 호스트 폴더가 이 경로에 연결된다.

SIF 사고 파일 예:

```json
{
  "dataset_id": "sif-demo-2026",
  "data_type": "accidents",
  "subdir": "law",
  "chunk_file": "05_chunks.jsonl",
  "validation_file": "06_validation.json"
}
```

사고 데이터의 경로는 `/data/out/accidents/law/`다. `validation_file`을 생략하거나 파일이 없으면 구조 검사만 수행될 수 있다. 응답의 `validation`이 `pipeline`인지 `structural`인지 확인한다. 운영 적재 전에는 가능하면 pipeline 검증 산출물과 `pipeline` 결과를 사용한다.

HTTP `200`에서 `saved`, `dataset_id`, `file`, `collection`, `validation`을 확인한다. Compose에서는 출력 파일이 호스트 `main-ai-model/out/vectors/<dataset_id>.jsonl`에도 나타난다. 이 단계에서 법령·SIF의 원본 및 청크 장부 등록은 수행되지만, 아직 Qdrant에 upsert된 것은 아니다.

### 5-3. Qdrant에 벡터 파일 올리기

벡터 파일과 collection이 일치하는지 확인한 다음 Swagger의 `POST /ingest/vectors/local/load`를 실행한다.

법령:

```json
{
  "dataset_id": "law-demo-2026",
  "collection": "kb_guides"
}
```

사고:

```json
{
  "dataset_id": "sif-demo-2026",
  "collection": "kb_accidents"
}
```

HTTP `200`에서 `indexed`가 파일 청크 수와 같은지, `collection`과 `dataset_id`가 의도한 값인지, `unlinked_payloads`가 `0`인지 확인한다. `finalized_sources`는 Qdrant 적재 후 `agent_kb`에서 `EMBEDDED`로 완료 처리한 원본 수다. 완료 처리는 파일 내 원본·청크 ID가 장부와 일치하고, 파일 청크 수가 해당 원본의 장부 전체 청크 수와 같을 때 수행된다.

`unlinked_payloads`가 0보다 크면 ID가 장부에 연결되지 않은 포인트가 있다는 뜻이다. 연결되지 않은 포인트는 Qdrant에는 적재되지만 장부 완료 처리는 할 수 없다. 장부 연결이 필요한 경우 `unlinked_payloads: 0` 및 `finalized_sources` 값을 확인한다. HTTP 502로 장부 완료 처리에 실패한 경우 Qdrant upsert는 이미 끝났을 수 있으므로 장부와 Qdrant 상태를 확인한 후 같은 요청으로 재시도한다.

### 5-4. MySQL `agent_kb` 장부에서 확인하기

Compose의 MySQL 컨테이너에서 `agent_kb`에 접속한다. 비밀번호 프롬프트에는 `.env`의 `KB_INGEST_PASSWORD` 값을 입력한다. 비밀번호를 명령행에 직접 적지 않는다.

```bash
docker exec -it database-mysql-container mysql -u kb_ingest -p agent_kb
```

원본 상태와 장부 청크 수를 확인한다. `1`은 이번 예시의 `kb_source_id`이므로 다른 자료를 확인할 때는 해당 원본 ID로 바꾼다.

```sql
SELECT
    s.id AS kb_source_id,
    s.source_type,
    s.external_id,
    s.status,
    s.chunk_count,
    s.embedding_model,
    COUNT(c.id) AS actual_chunk_count
FROM kb_sources AS s
LEFT JOIN kb_chunks AS c ON c.source_id = s.id
WHERE s.id = 1
GROUP BY s.id;
```

정상 완료된 예시는 `status`가 `EMBEDDED`이고 `chunk_count`와 `actual_chunk_count`가 모두 `695`다. 개별 청크 ID와 Qdrant point ID를 확인하려면 다음 쿼리를 실행한다.

```sql
SELECT
    id AS kb_chunk_id,
    source_id AS kb_source_id,
    chunk_index,
    collection_name,
    qdrant_point_id,
    locator
FROM kb_chunks
WHERE source_id = 1
ORDER BY chunk_index
LIMIT 20;
```

`LIMIT`을 조정하거나 제거하면 더 많은 청크를 조회할 수 있다. 확인을 마치면 `exit;`로 MySQL 클라이언트를 종료한다.

#### SIF 사고 원본 적재 건수 확인

이번 SIF 벡터 파일 예시의 `dataset_id`는 `accidents-KHE_TQC`다. `dataset_id`는 벡터 파일 이름이며 MySQL 장부 필드가 아니므로, 장부에서는 `source_type`과 상태를 기준으로 조회한다.

상태별 SIF 원본 수:

```sql
SELECT status, COUNT(*) AS source_count
FROM kb_sources
WHERE source_type = 'SIF'
GROUP BY status;
```

원본별 장부 청크 수를 확인한다. `LIMIT`을 생략하면 전체 SIF 원본을 볼 수 있다.

```sql
SELECT
    s.id AS kb_source_id,
    s.external_id,
    s.status,
    s.chunk_count,
    COUNT(c.id) AS actual_chunks
FROM kb_sources AS s
LEFT JOIN kb_chunks AS c ON c.source_id = s.id
WHERE s.source_type = 'SIF'
GROUP BY s.id
ORDER BY s.id
LIMIT 20;
```

확인한 `accidents-KHE_TQC` 적재 결과는 SIF 원본 3,069건이며 모두 `EMBEDDED` 상태다. 각 원본은 1개 청크를 가져 장부의 총 청크 수도 3,069건이다. 컬렉션 인덱스 정의를 확인하려면 다음 명령을 사용한다.

```sql
SHOW INDEX FROM kb_sources;
SHOW INDEX FROM kb_chunks;
```

## 6. Swagger 응답으로 확인할 것

| 확인 단계 | 성공 기준 |
| --- | --- |
| 인증 | API 실행 결과가 401이 아님 |
| 요청 검증 | 422가 아닌 성공 응답 |
| 장부 연결 | 권장 API 응답의 `source_id`, `chunk_id`, `point_id`가 반환됨 |
| 벡터화 | `indexed` 또는 `saved`가 요청 청크 수와 같음 |
| 파일 기반 벡터 적재 | `collection`, `unlinked_payloads`, `finalized_sources` 확인 |
| 완료 처리 | 권장 API가 성공하고 서버가 장부 완료 상태를 갱신함 |

Swagger의 성공 응답만으로 실제 검색 순위나 검색 품질까지 검증되는 것은 아니다. 별도의 검색 평가가 필요하면 프로젝트의 `scripts/try_rag_tools.py`와 `scripts/eval_retrieval.py`를 사용한다. Qdrant 저장소를 직접 확인할 때는 `http://localhost:60011`의 Qdrant API를 쓰되, 쓰기 키를 브라우저 주소나 문서에 넣지 않는다.

## 7. 자주 보는 오류

| HTTP 응답 | 먼저 확인할 사항 |
| --- | --- |
| 401 | Swagger Authorize에 `KB_INGEST_API_TOKEN`을 입력했는지, 값이 현재 Compose 설정과 같은지 확인 |
| 422 | `source_type`과 API 경로, 중복 `chunk_index`, 빈 본문, 태그 배열 형식, 데이터 파일 경로·검증 파일을 확인 |
| 404 | `/ready` 및 파일 기반 경로의 청크 파일 존재 여부 확인 |
| 502 | 임베딩 서버/Qdrant 오류. Docker에서 `kb-ingest`, `python-embedding-model`, `qdrant-server` 로그와 상태 확인 |
| 503 | 필수 키/URL, DB 연결, `agent_kb` 장부 등록·검증 오류 확인 |

테스트 목적이라도 실제 운영 중인 `external_id`나 기존 자료와 같은 ID를 사용하지 않는다. `docker compose down -v`는 MySQL·Qdrant 데이터를 포함한 볼륨을 지우므로 실행하지 않는다.

## 8. 관련 경로

- API 진입점: `app/ingest_main.py`
- 요청 스키마: `app/ingest/schemas.py`
- 원본·청크 적재: `app/ingest/documents.py`
- 파일 벡터화·Qdrant 업로드: `app/ingest/local_vectors.py`
- Compose 포트·환경 변수·마운트: `../../main-infra-manage/docker/docker-compose.yml`
