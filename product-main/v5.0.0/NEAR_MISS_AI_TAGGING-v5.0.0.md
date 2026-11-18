# 사내 아차사고 AI 태깅 및 현장 RAG 연동

## 목적

이 문서는 사내 아차사고의 AI 5축 태깅 API, 규칙 및 시나리오 C 보강 동작, 적재 계약, 현장 RAG 검색 범위를 설명합니다.

기본 원칙은 다음과 같습니다.

- AI 결과는 제안이며 사람이 확정하기 전에는 RAG 적재와 긴급 경고에 사용하지 않습니다.
- 태그 규칙이 먼저 실행되고, 규칙이 판단하지 못한 축만 시나리오 C 모델이 보강합니다.
- 규칙 결과를 시나리오 C 모델이 덮어쓰지 않습니다.
- 태그 값은 `tag_codes_v5.json`과 서버가 사용하는 영문 태그 코드만 허용합니다.
- 하나의 응답을 규칙 제안과 모델 보강 제안으로 구분해 저장할 수 있어야 합니다.

## 구현 위치

| 위치 | 역할 |
| --- | --- |
| `app/near_miss/schemas.py` | 아차사고 태깅 요청·응답 Pydantic 스키마 |
| `app/near_miss/tagging.py` | 규칙 기반 5축 태깅 |
| `app/near_miss/llm.py` | 시나리오 C를 이용한 누락 축 보강 |
| `app/near_miss/service.py` | 규칙 우선 태깅 흐름과 출처·버전 조합 |
| `app/api/v1/agent/near_miss_tag.py` | `POST /api/v1/agent/near-miss-tag` 라우터 |
| `app/rag/tags/metadata.py` | CAS 체크섬 등 공통 메타데이터 검증 재사용 |
| `app/chunking/accident_ingest/profiles/sif.py` | 사고형태·작업유형 규칙 재사용 |
| `app/ingest/schemas.py` | `NEAR_MISS` 적재 payload 검증 |
| `app/rag/filters.py` | 공공 자료와 실행 현장 아차사고 검색 범위 |

`app/rag/tags/`에는 공공 자료 태깅과 태그 메타데이터 검증만 둡니다. 아차사고 제안 API는 도메인 패키지인 `app/near_miss/`에 둡니다.

## Docker 스테이지

아차사고 태깅 기능은 Dockerfile의 일반 AI API를 만드는 `ai` 스테이지에 포함됩니다.

- 빌드 대상: `ai`
- 컨테이너 포트: `60100`
- 실행 앱: `app.main:app`
- 실행 명령: `uvicorn app.main:app --host 0.0.0.0 --port 60100`
- 소스 포함 방식: `COPY app ./app`

아차사고 관련 라우터와 도메인 패키지도 `COPY app ./app`으로 이미지에 함께 포함됩니다.

- `app/near_miss/llm.py`
- `app/near_miss/service.py`
- `app/near_miss/tagging.py`
- `app/api/v1/agent/near_miss_tag.py`

빌드와 실행 예시는 다음과 같습니다.

```powershell
docker build --target ai -t main-ai-model:api .
docker run -d --name ai-api `
  -p 60100:60100 `
  --env-file .env `
  main-ai-model:api
```

다른 스테이지는 다음 역할만 담당하므로 아차사고 태깅 API를 제공하지 않습니다.

- `embedding`: 임베딩 및 RAG 검색 보조 API
- `ingest`: 아차사고 적재 및 공공 자료 적재 계약
- `qdrant-init`: Qdrant 초기화

## 태깅 API

### 요청

```http
POST /api/v1/agent/near-miss-tag
Content-Type: application/json
```

```json
{
  "runId": 51,
  "siteId": 3,
  "nearMissId": 9,
  "title": "비계 위 공구 낙하",
  "description": "2단 비계에서 작업 중 망치가 떨어짐",
  "immediateAction": "하부 통제",
  "zoneName": "A동 외벽",
  "workType": "HEIGHT"
}
```

필드 규칙은 다음과 같습니다.

- `runId`, `siteId`, `nearMissId`는 양의 정수여야 합니다.
- `title`은 필수이며 최대 300자입니다.
- `description`은 최대 4000자입니다.
- `immediateAction`, `zoneName`, `workType`은 없을 수 있습니다.
- `workType`은 서버가 작업 유형을 태그 코드로 변환해 전달합니다.

### 응답

```json
{
  "success": true,
  "data": {
    "tags": [
      {
        "axis": "SEVERITY",
        "value": "NEAR_MISS",
        "confidence": 1.0
      },
      {
        "axis": "ACCIDENT_TYPE",
        "value": "STRUCK",
        "confidence": 0.8
      },
      {
        "axis": "WORK_TYPE",
        "value": "HEIGHT",
        "confidence": 0.9
      }
    ],
    "taggingSource": "near_miss_rules",
    "taggingVersion": "nm-1",
    "model": null
  }
}
```

응답의 축은 다음 다섯 개만 허용합니다.

- `SEVERITY`
- `ACCIDENT_TYPE`
- `HAZARD`
- `WORK_TYPE`
- `SUBSTANCE`

`tagStatus`는 응답에 포함하지 않습니다. 아차사고의 다섯 축은 모두 적용 대상이며, 제안 태그가 있으면 확인됨, 없으면 미확인으로 화면에 표시합니다.

다음은 실제 요청과 시나리오 C 모델 보강이 결합된 응답 예시입니다.

```json
{
  "success": true,
  "data": {
    "tags": [
      {
        "axis": "SEVERITY",
        "value": "NEAR_MISS",
        "confidence": 1.0
      },
      {
        "axis": "WORK_TYPE",
        "value": "HEIGHT",
        "confidence": 0.9
      },
      {
        "axis": "SUBSTANCE",
        "value": "108-88-3",
        "confidence": 0.5
      }
    ],
    "taggingSource": "near_miss_rules+llm",
    "taggingVersion": "nm-1",
    "model": "Qwen/Qwen2.5-1.5B-Instruct"
  }
}
```

이 예시의 규칙 제안은 `SEVERITY=NEAR_MISS`와 요청의 `workType=HEIGHT`이며,
누락된 축에 대해 시나리오 C가 유효한 CAS 번호 `108-88-3`를 추가 제안했습니다.
모델 보강 제안의 신뢰도는 `0.5`입니다.

## 규칙 기반 태깅

태깅 버전은 `nm-1`입니다.

| 축 | 동작 |
| --- | --- |
| `SEVERITY` | 기본으로 `NEAR_MISS`를 제안합니다. 신뢰도는 `1.0`입니다. |
| `ACCIDENT_TYPE` | 제목, 설명, 즉시 조치의 키워드에 공공 SIF의 `FACTOR_RULES`를 적용합니다. 신뢰도는 `0.8`입니다. |
| `HAZARD` | 사고형태에서 파생합니다. 감전은 `ELECTRICAL`, 누출은 `CHEMICAL`, 맞음·끼임은 `MECHANICAL`로 제안합니다. |
| `WORK_TYPE` | 요청의 `workType`이 유효한 태그 코드이면 우선 사용합니다. 없으면 `CONSTRUCTION_WORK_TYPE_RULES` 키워드를 사용합니다. |
| `SUBSTANCE` | 본문에서 CAS 형식을 찾은 뒤 체크섬을 검증한 경우에만 제안합니다. 신뢰도는 `1.0`입니다. |

규칙 제안 신뢰도와 모델 제안 신뢰도는 분리합니다.

- 요청에서 온 작업 유형: `0.9`
- 규칙 키워드: `0.8`
- CAS 번호 또는 기본 심각도: `1.0`
- 시나리오 C 모델 보강: `0.5`

## 시나리오 C LLM 보강

시나리오 C의 엔진 키는 `near_miss_warning`이고, 영문 별칭 `C`도 provider가 허용합니다.

현재 시나리오 설정은 다음 모델 바인딩을 사용합니다.

```python
"C": {
    "engine": "NEAR_MISS_WARNING",
    "model_id": "Qwen/Qwen2.5-1.5B-Instruct",
}
```

동작 순서는 다음과 같습니다.

1. 규칙 태깅을 실행합니다.
2. 이미 제안된 축은 누락 축에서 제외합니다.
3. 누락된 축이 없으면 시나리오 C를 호출하지 않습니다.
4. 누락 축이 있으면 시나리오 C에 축 목록과 허용 코드를 전달합니다.
5. JSON에서 허용 코드 외의 값은 버립니다.
6. 응답에 유효한 모델 제안이 있으면 `model`을 기록합니다.
7. 모델 호출 결과가 모델 설정 오류 등 `LLMError`이면 기존 공통 에러 형식으로 반환합니다.

모델은 규칙 결과를 덮어쓰지 않습니다.

출처 값은 다음과 같습니다.

- 규칙만 제안: `near_miss_rules`
- 규칙과 유효한 모델 보강: `near_miss_rules+llm`

## 적재 계약

아차사고 적재 payload의 `severity`는 문자열 배열입니다.

```json
{
  "severity": ["NEAR_MISS"],
  "accident_type": ["FALL"],
  "hazard": ["HUMAN"],
  "work_type": ["HEIGHT"],
  "cas": [],
  "ghs": []
}
```

현재 검증 규칙은 다음과 같습니다.

- `severity`는 문자열 배열이어야 합니다.
- 빈 배열은 허용됩니다.
- `accident_type`, `hazard`, `work_type`, `cas`, `ghs`도 문자열 배열이어야 합니다.
- 문자열 배열의 각 원소는 공백이 아니어야 합니다.
- `ghs`는 현재 아차사고 계약상 빈 배열이어야 합니다.
- `zone_id`가 없으면 서버 payload의 값은 `null`이어야 합니다.
- `source_type=NEAR_MISS`는 업무 DB의 양의 `site_id`와 `flagged=false`가 필요합니다.

이 검증은 `tests/test_ingest_api.py`의 아차사고 요청을 기준으로 검증합니다.

## 현장 RAG 검색 계약

검색 현장 범위는 다음과 같습니다.

- `site_id`가 없으면 공공 자료만 검색합니다: `[0]`
- `site_id=17`이면 공공 자료와 실행 현장 자료를 검색합니다: `[0, 17]`
- 둘 이상의 현장은 받지 않습니다.
- 점이 다른 현장에 속한 `NEAR_MISS`는 검색 결과에서 제외합니다.
- `kb_guides`는 현재 사내 자료를 포함하지 않으므로 현장 필터와 무관하게 공공 자료만 반환합니다.

실행 현장 번호는 `RunContext.site_id`에서만 유도해야 합니다.

- 검색 도구 호출 인자의 `site_id`는 예약 키로 제거합니다.
- 고정 필터를 덮어쓰지 않도록 보호합니다.
- 스프링이 확인한 실행 현장만 검색 인자로 전달합니다.
- 클라이언트가 임의의 현장 번호를 검색 입력으로 지정할 수 없게 합니다.

관련 검증은 `tests/test_rag_configuration.py`의 `test_search_site_filter_includes_public_and_current_site()`에 있습니다.

## 서버 연동 기대 동작

이 문서의 API는 Spring이 다음 흐름으로 사용합니다.

1. 현장 권한을 확인한 뒤 `POST /near-misses/{id}/auto-tags`를 호출합니다.
2. AI 태그는 `agent_runs.output_json`에 저장합니다.
3. 받은 태그를 `record_tags`에 `source=AI`와 confidence와 함께 제안으로 저장합니다.
4. 사람이 그대로 확정하면 기존 또는 확정용 제안 태그가 유지됩니다.
5. 사람이 수정하면 `source=USER`로 저장합니다.
6. `TAGGED` 이상일 때만 RAG에 적재합니다.
7. 긴급 경고는 사람이 확정한 태그만 사용합니다.

이 동작은 이 저장소에서 완전히 구현되지 않으며, 서버와 웹 쪽에서 각각 연동해야 합니다.

## 검증 명령

workspace 가상환경으로 다음 테스트를 실행합니다.

```powershell
.\.venv\Scripts\python.exe -m pytest -q `
  tests/test_near_miss_tagging.py `
  tests/test_ingest_api.py `
  tests/test_rag_configuration.py
```

전체 관련 RAG 및 태깅 검증은 다음으로 실행할 수 있습니다.

```powershell
.\.venv\Scripts\python.exe -m pytest -q `
  tests/test_near_miss_tagging.py `
  tests/test_ingest_api.py `
  tests/test_rag_configuration.py `
  tests/test_rag_tags.py `
  tests/test_verify_sif_search.py
```

## 남은 결정

다음 항목은 서버 연동 시 별도 확정이 필요합니다.

- 아차사고 payload에 `tag_status`, `tagging_source`, `tagging_version`을 포함할지 여부
- 시나리오 C의 실제 운영 모델과 로컬 캐시 경로
- 재색인 방식이 컬렉션 재생성인지 같은 point ID 덮어쓰기인지
- 잠재 심각도를 `SEVERITY`의 `SERIOUS`·`FATAL`로 둘지 별도 SIF 가능성 축으로 둘지
- 사내 아차사고 근거에 표시할 `kb_source_id` 또는 아차사고 상세 연결 값
