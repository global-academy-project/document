# 지식 데이터 및 RAG 요구사항 구현 현황

- 확인일: 2026-11-12
- 기준 문서: `/home/geonwoo/Downloads/rag.md`
- 확인 범위: 임베딩 모델 파인튜닝을 제외한 기준 문서의 기능 및 구성 요구사항
- 확인 대상: 현재 작업 트리의 AI 모델, Spring 서버, 인프라 Compose 코드

> [!IMPORTANT]
> 이 문서는 코드와 설정을 기준으로 작성한 점검 결과다. 2026-11-12에 로컬 Compose에서 `guide-KHE_TQC` 법령 벡터 695건과 `accidents-KHE_TQC` SIF 벡터 3,069건의 Qdrant 적재 및 `agent_kb` 완료 상태를 확인했다. 해당 확인에서는 LAW 695청크와 SIF 원본 3,069건·청크 3,069건이 모두 `EMBEDDED`였다. 이는 두 적재 사례의 검증이며, 전체 서비스의 종단 간 동작이나 전체 데이터의 품질을 검증한 것은 아니다.

## 상태 기준

- **구현**: 요구사항에 해당하는 코드가 확인됨. 실제 운영 데이터와 통합 환경의 검증 완료를 의미하지는 않는다.
- **부분 구현**: 관련 코드가 있으나 기본 실행 경로, 데이터 처리 또는 연결이 요구사항 전체를 충족하지 않는다.
- **미구현·미확인**: 요구사항에 해당하는 동작을 확인하지 못했거나 기본 경로에 연결되어 있지 않다.

## 1. 시스템 경계와 데이터 처리

| 요구사항 | 상태 | 확인 내용 |
| --- | --- | --- |
| 웹 요청은 Spring을 거쳐 AI와 통신 | 구현 | 웹 API 클라이언트는 상대 경로를 사용하고 Compose는 웹 프록시를 Spring에 연결한다. 다만 AI 포트가 호스트에 공개되어 있어 Spring을 통한 접근만 네트워크 수준에서 강제하지는 않는다. |
| AI는 업무 기록을 Spring `search_records` API로 조회 | 구현 | 내부 API 클라이언트와 기록 조회 어댑터가 있다. 현재 Compose의 AI 서비스에는 내부 API 주소와 토큰이 전달되지 않아, 이 설정 그대로는 해당 기록 조회 경로를 사용할 수 없다. |
| 규칙 판정은 Spring `check_rules`가 수행 | 미구현·미확인 | 작업 등록 도구 레지스트리는 `check_rules`에 mock을 등록한다. 실제 Spring 도구로 교체하는 연결은 확인되지 않았으며 Compose 기본값도 `MOCK`이다. `SPRING` 설정값만 변경해서 연결된다고 판단할 수 없다. |
| AI는 업무 DB에 직접 연결하지 않음 | 구현 | 확인한 업무 기록 조회 경로는 Spring API를 사용하고, 지식 장부 코드는 `agent_kb`에 연결한다. 다만 Compose에서 서비스들이 같은 기본 네트워크를 사용하므로 DB 접근 자체가 네트워크 수준에서 차단된 것은 아니다. 실제 DB 계정 권한은 별도 검증이 필요하다. |
| 업무 기록과 사내 아차사고의 RAG 제외 | 구현 | RAG 검색은 `site_id=0`인 공공 데이터만 조회한다. 양의 현장 ID를 요청해도 현재 검색 대상은 공공 데이터로 제한된다. 사내 기록의 별도 Spring 조회 기능과 구분해야 한다. |
| 지식 문서의 청크·벡터와 지식 장부 저장 | 부분 구현 | Qdrant에 청크와 벡터를 저장하고 `agent_kb`에 원본 메타데이터와 청크 텍스트를 기록하는 경로가 있다. DB에는 원본 파일 자체가 아니라 원본의 URL·파일 경로 등과 청크가 저장된다. |
| MSDS 공단 API 조회와 캐시 | 부분 구현 | 실제 공단 API 클라이언트와 프로세스 메모리 캐시는 구현되어 있다. 그러나 기본 작업 등록 도구 레지스트리에 해당 클라이언트를 등록하지 않아 현재 경로는 mock을 사용한다. 키 설정만으로 실제 연동이 활성화되지는 않는다. |

**근거:** [웹 API 클라이언트](../../main-vite-web/src/api/client.js), [Compose 설정](../../main-infra-manage/docker/docker-compose.yml), [기록 조회 의존성](../app/api/v1/agent/record_query_deps.py), [도구 등록](../app/tools/tool_setup.py), [mock 도구](../app/tools/mock/__init__.py), [MSDS 클라이언트](../app/publicdata/msds.py), [지식 DB 스키마](../db/migration/V1__kb_init.sql), [RAG 필터](../app/rag/filters.py).

## 2. 적재, 태그 및 검색

| 요구사항 | 상태 | 확인 내용 |
| --- | --- | --- |
| 원본을 청크로 분할하고 bge-m3로 임베딩·적재 | 부분 구현 | 법령·사고 문서별 청킹 도구와 임베딩·적재 API가 있다. `/ingest/accidents`, `/ingest/guides`는 청크 입력을 장부에 등록하고 임베딩·Qdrant 적재·완료 처리를 한 요청에서 수행한다. 파일 기반 경로는 `/ingest/vectors/local/file`로 벡터 JSONL을 만든 뒤 `/ingest/vectors/local/load`로 Qdrant에 적재하며 연결된 장부도 완료 처리한다. 다만 임의의 원본 텍스트를 받아 자동 청킹하는 단일 API는 확인하지 못했다. |
| `kb_accidents`, `kb_guides` 컬렉션 분리 | 구현 | 컬렉션 초기화 스크립트와 사고·지침별 검색 도구가 있다. 포인트는 청크 단위로 생성된다. |
| Dense + Sparse 하이브리드 검색과 RRF | 구현 | 임베딩 결과에 dense·sparse 벡터를 포함하고, sparse 값이 있는 검색에서 Qdrant prefetch와 RRF를 사용한다. sparse 값이 비어 있으면 dense만 검색한다. |
| `agent_kb`의 `kb_sources`, `kb_chunks` 기록 | 부분 구현 | `/ingest/accidents`, `/ingest/guides`와 장부 연계 벡터 파일 생성 경로는 원본·청크 ID와 UUID v5 point ID를 연결한다. `/ingest/vectors/local/load`는 연결된 포인트를 장부와 대조한 뒤 Qdrant에 적재하고, 청크 수가 일치하면 원본을 `EMBEDDED`로 완료 처리한다. 다만 호환용 `/ingest/document`와 장부 ID 없는 포인트를 허용하는 경로도 유지되므로 모든 적재 경로에서 장부 일관성이 강제되지는 않는다. |
| 지식 문서와 업무 기록의 5축 태깅 | 부분 구현 | 업무 DB에는 태그 코드·기록 태그 구조가 있고, Qdrant에도 5축 필드와 인덱스가 있다. 모든 데이터의 모든 축이 채워지는 것은 아니다. 예를 들어 사고 청킹의 CAS/GHS는 빈 배열이고 법령의 hazard/work_type 자동 태깅은 기본적으로 꺼져 있다. |
| 에이전트의 5축 필터 검색 | 부분 구현 | 검색 백엔드는 5축 필터를 지원하지만, 기본 근거 수집기는 작업 맥락에 맞는 태그 필터를 검색 도구에 전달하지 않는다. 기본 조건으로는 컬렉션, 출처 유형, 공공 데이터 및 `flagged=false`가 적용된다. |
| 동의어 사전 기반 검색어 확장 | 부분 구현 | 작업 유형 정규화를 위한 동의어는 있지만 검색 질의 자체를 동의어 사전으로 확장하는 로직은 확인하지 못했다. |
| 적재 시 청크에 태그 키워드 삽입 | 부분 구현 | 사고·법령 청크의 임베딩 입력에 분류·문서 계층 등의 문맥 머리말을 추가한다. 다만 5축 태그 값을 공통적으로 본문에 삽입하는 구현은 확인하지 못했다. |

**하이브리드 검색 주의:** 현재 sparse 벡터의 값은 BGE-M3의 학습된 lexical weight가 아니라 토큰 출현 횟수다. RRF 경로가 구현된 사실만으로 검색 품질이 요구 수준에 도달했다고 볼 수 없다.

**근거:** [공개 문서 적재 API](../app/ingest/documents.py), [벡터 파일 적재](../app/ingest/local_vectors.py), [장부 코드](../app/rag/catalog.py), [Qdrant 검색·적재](../app/rag/qdrant.py), [임베딩 모델](../app/rag/embedding_model.py), [컬렉션 초기화](../scripts/init_qdrant.py), [사고 태그](../app/chunking/accident_ingest/tags.py), [법령 청킹 설정](../app/chunking/law_ingest/config.py), [근거 검색 설정](../app/agent/evidence_collector/vocab.py).

## 3. 에이전틱 RAG 흐름

| 요구사항 | 상태 | 확인 내용 |
| --- | --- | --- |
| 작업 등록의 근거 수집 단계에서 검색 계획 수립 | 구현 | 작업 등록 에이전트가 근거 수집기를 호출하고 LLM 기반 검색 계획을 수립한다. |
| `search_accidents`, `search_guides` 호출 | 구현 | 사고 및 지침별 도구를 호출하고 Compose의 `RAG_BACKEND` 설정으로 실제 Qdrant 검색 백엔드를 연결한다. 5축 필터 전달은 위의 미완성 항목과 별개다. |
| 작업 유형별 필수 근거 충분성 검사 | 구현 | 작업 유형별 필수 근거와 키워드를 비교한다. 다만 기본 기준은 코드에 임시값으로 명시되어 있어 안전 전문가와 정답 데이터로 검증해야 한다. |
| 부족한 근거에 대한 최대 3회 재질의 | 구현 | 기본값은 최초 검색 후 재질의 최대 3회다. 부족 항목이 없거나 새로운 근거를 찾지 못하면 조기 종료할 수 있다. |
| 인용을 Spring에 전달하고 UUID v5와 함께 저장 | 부분 구현 | AI가 청크 ID와 출처 정보를 전달하고 Spring이 유효한 KB 인용을 저장한다. 출처 종류·컬렉션 등 필수 정보가 없거나 유효하지 않은 인용은 인용 테이블 저장을 건너뛰고 응답 JSON에만 남는다. |

**근거:** [작업 등록 흐름](../app/agent/work_register.py), [근거 수집기](../app/agent/evidence_collector/collector.py), [필수 근거 기준](../app/agent/evidence_collector/vocab.py), [인용 출처 구성](../app/agent/kb_source.py), [Spring 인용 저장](../../main-server/src/main/java/com/example/backend/agent/service/AgentResultWriter.java).

## 4. 우선 조치

1. **실제 규칙 판정 연결**: 작업 등록의 mock `check_rules`를 Spring 내부 API와 연결하고, 인증·오류·차단 결과가 전달되는지 통합 검증한다.
2. **실제 MSDS 연결**: 공단 클라이언트를 작업 등록 도구 레지스트리에 등록하고, 키 누락·API 오류·캐시 동작을 검증한다.
3. **내부 API 설정 및 경계 확인**: AI 서비스에 필요한 Spring 내부 API 주소·인증 설정을 전달하고, DB 권한과 네트워크 노출을 요구사항에 맞게 검증한다.
4. **5축 태그 품질과 검색 조건 연결**: 데이터별 태그 생성 범위와 빈 값 정책을 정한 뒤 검색 계획에서 필터를 전달하고, 정확도와 누락률을 평가한다.
5. **적재 경로 및 장부 일관성 정리**: 권장 API와 파일 기반 적재 경로는 장부 검증·완료 처리를 연결했다. 호환 우회 경로의 사용 조건을 제한하고 실패 후 복구를 추가 검증한다.
6. **검색 품질 개선과 검증**: 동의어 확장 및 태그 키워드 삽입의 적용 범위를 정하고, 하이브리드 검색과 임시 필수 근거 기준을 평가 데이터 및 전문가 검토로 검증한다.
7. **인용 저장 종단 간 검증**: 출처 정보가 있는 경우와 없는 경우를 포함해 AI 응답에서 Spring 인용 테이블까지 확인한다.

## 5. 수행한 검증

2026-11-12에 AI 모델 저장소에서 ingest API 테스트를 실행했다.

```bash
.venv/bin/python -m pytest -q tests/test_ingest_api.py
```

결과: **36개 통과**. 테스트에는 모의 응답을 사용하는 경우가 있으며, 이 결과만으로 실제 공단 API, Spring, MySQL, Qdrant 및 임베딩 서버를 연결한 전체 시나리오가 검증된 것은 아니다. 별도로 로컬 Compose에서 `guide-KHE_TQC`는 `indexed=695`, `unlinked_payloads=0`, `finalized_sources=1` 및 MySQL `kb_source_id=1`의 `EMBEDDED`, 청크 695건을 확인했다. `accidents-KHE_TQC`는 `indexed=3069`, `unlinked_payloads=0`, `finalized_sources=3069`를 반환했고, 파일의 고유 원본 ID 3,069개 모두 MySQL에서 `EMBEDDED`이며 원본별 청크 1건씩 총 3,069건임을 확인했다. 두 경우의 임베딩 모델은 `BAAI/bge-m3`다.

## 6. 지식 데이터·RAG 체크리스트

확인일: 2026-11-12. 체크 표시는 현재 코드의 기능 구현을 뜻하며, 실제 배포 환경과 전체 데이터의 검증 완료를 뜻하지 않는다. 부분 구현 및 확인이 필요한 항목은 체크하지 않았다.

- [ ] **공공 문서는 `agent_kb` 장부와 Qdrant에 동일한 ID 체계로 적재한다.**
  - `/ingest/accidents`, `/ingest/guides`와 `/ingest/vectors/local/file`은 원본·청크 장부와 UUID v5 point ID를 연결한다. `/ingest/vectors/local/load`는 장부 ID가 있는 청크의 메타데이터와 ID를 검증하고 Qdrant 적재 후 청크 수가 일치하면 `EMBEDDED`로 완료 처리한다. 2026-11-12에 LAW 벡터 695건과 SIF 벡터 3,069건으로 이 경로를 확인했다.
  - 호환용 `/ingest/document` 및 장부 ID 없는 포인트를 허용하는 벡터 적재는 우회로로 남아 있어 장부 연결을 전 경로에서 강제하지는 않는다.
  - 근거: [공개 문서 적재 API](../app/ingest/documents.py), [벡터 파일 적재](../app/ingest/local_vectors.py), [장부 검증·완료 처리](../app/rag/catalog.py).

- [ ] **payload 태그 값은 업무 DB `tag_codes` 코드와 같다.**
  - 일부 태깅 규칙은 코드표에 맞춰 작성되어 있으나, 적재 API가 모든 태그 값을 업무 DB 코드표와 대조해 강제하지는 않는다. 데이터별 빈 태그도 존재하며 전체 적재 데이터의 일치 여부는 검증하지 않았다.
  - 근거: [업무 DB 코드표](../../main-server/src/main/resources/db/migration/V5__near_miss_tags.sql), [공공 문서 적재 요청](../app/ingest_main.py).

- [ ] **공공 데이터의 `site_id`는 Qdrant에서 0, `agent_kb`에서 NULL이다.**
  - `/ingest/document` 경로는 해당 규칙을 적용하고 검증한다. 별도 JSONL 업서트는 공공 데이터의 `site_id=0`과 장부 일치를 강제하지 않는다.
  - 근거: [공공 문서 payload 구성](../app/ingest_main.py), [장부 등록](../app/rag/catalog.py), [JSONL 검증](../scripts/upsert_qdrant_jsonl.py).

- [ ] **사내 아차사고는 RAG에 넣지 않고 Spring SQL로 처리한다.**
  - 기본 설정에서는 Spring의 Qdrant 적재가 꺼져 있고 RAG 검색은 공공 데이터인 `site_id=0`만 조회한다. 다만 사내 아차사고를 Qdrant에 적재하는 코드와 이를 허용하는 API가 남아 있어 적재 자체를 구조적으로 금지하지는 않는다.
  - 근거: [Spring 설정](../../main-server/src/main/resources/application.yaml), [아차사고 적재 리스너](../../main-server/src/main/java/com/example/backend/nearmiss/kb/NearMissKbListener.java), [RAG 필터](../app/rag/filters.py).

- [ ] **MSDS는 RAG가 아니라 공단 API로 조회한다.**
  - 공단 API 클라이언트와 메모리 캐시는 구현되어 있지만 기본 작업 등록 도구 레지스트리에 연결되지 않아 현재는 mock을 사용한다.
  - 근거: [MSDS 클라이언트](../app/publicdata/msds.py), [도구 등록](../app/tools/tool_setup.py).

- [ ] **규칙 판정은 RAG가 아니라 Spring `check_rules`가 수행한다.**
  - 작업 등록은 별도 `check_rules` 도구를 호출하지만 기본 레지스트리는 mock을 등록한다. 실제 Spring 도구 연결은 확인되지 않았고 Compose 기본값도 `MOCK`이다.
  - 근거: [mock 도구 등록](../app/tools/mock/__init__.py), [Compose 설정](../../main-infra-manage/docker/docker-compose.yml).

- [ ] **에이전트는 Qdrant 읽기 전용 키와 `agent_kb`의 `kb_reader`만 사용한다.**
  - Compose는 에이전트에 Qdrant 읽기 전용 키를 전달하고 검색 코드도 그 키를 사용한다. `kb_reader` 계정은 정의되어 있으나 현재 검색 경로는 장부에 직접 접속하지 않으므로 해당 계정의 실제 사용은 확인되지 않는다. 배포 환경의 실제 권한은 별도 검증이 필요하다.
  - 근거: [RAG 설정](../app/rag/config.py), [DB 계정 생성](../../main-infra-manage/db/init/10-agent-kb.sh), [검색 백엔드](../app/rag/tool_backend.py).

- [x] **하이브리드 검색: dense + sparse, RRF 연결 여부를 확인한다.**
  - 적재와 검색에 dense·sparse 벡터가 연결되어 있으며 검색 시 RRF로 결합한다. sparse가 비어 있으면 dense만 사용한다. 구현 확인과 실제 검색 품질 검증은 구분한다.
  - 근거: [Qdrant 검색·적재](../app/rag/qdrant.py), [하이브리드 테스트](../tests/test_rag_hybrid.py).

- [x] **장부 쓰기 코드 위치를 확인한다.**
  - [장부 코드](../app/rag/catalog.py)의 `register_source_chunks`, `register_sources_chunks`에서 원본과 청크를 기록한다. [사고·지침 API](../app/ingest/documents.py) 및 [파일 벡터화](../app/ingest/local_vectors.py)가 이를 사용한다. 호환 우회 경로까지 장부 연결이 보장된다는 뜻은 아니다.

- [ ] **동의어가 잘 잡히지 않을 때 시도할 개선 방안을 확인한다.**
  - 작업 유형 정규화 동의어와 임베딩용 문맥 머리말은 있다. 동의어 사전 기반 질의 확장과 5축 태그 키워드의 공통 삽입은 확인하지 못했다. 검색 평가 결과를 토대로 적용 여부와 오검색 영향을 검증한다.
  - 근거: [작업 유형 동의어](../app/agent/evidence_collector/vocab.py), [사고 청크 머리말](../app/chunking/accident_ingest/fields.py).

---

## 7. 별도 문서 점검: A님 지식 데이터 저장·검색 구조

- 확인일: 2026-11-12
- 기준 문서: `/home/geonwoo/Downloads/A님_지식데이터_저장검색_구조정리 (1).md`
- 확인 범위: 현재 작업 트리의 구현 및 Compose 설정에 대한 정적 점검. 2026-11-12에는 별도로 로컬 Compose에서 LAW 벡터 695건과 장부 완료 상태를 확인했으며, 실제 공용 환경의 데이터와 전체 종단 간 동작은 확인하지 않았다.

### 7-1. 구현·미구현 현황

| 기준 문서의 내용 | 상태 | 현재 구현 및 제한 사항 |
| --- | --- | --- |
| `kb_accidents`·`kb_guides`와 청크 단위 포인트 | 구현 | 컬렉션과 dense·sparse 벡터, payload 인덱스 생성 코드가 있다. [컬렉션 초기화](../scripts/init_qdrant.py) |
| Dense + sparse 하이브리드 검색(RRF) | 구현 | 두 벡터를 적재하고 검색 시 RRF로 결합한다. sparse가 비어 있으면 dense만 사용한다. 원문 8장의 “dense만 사용”이라는 설명은 현재 코드와 다르다. [Qdrant 검색](../app/rag/qdrant.py) |
| `agent_kb` 장부 기록 및 UUID v5 ID | 부분 구현 | 전용 사고·지침 API와 파일 벡터화 경로는 원본·청크 장부 ID 및 UUID v5 point ID를 연결한다. 파일 업로드는 장부 대조 및 Qdrant 적재 후 청크 수가 일치하면 `EMBEDDED`로 완료 처리한다. 다만 장부 ID 없는 포인트를 허용하는 우회 경로가 있어 모든 적재에 일관성을 강제하지 않는다. 2026-11-12 로컬 Compose에서 LAW 695건과 SIF 3,069건 적재 및 장부 완료를 확인했다. 원문 8장의 “장부 쓰기 코드 없음”은 현재 코드와 다르다. [문서 적재 API](../app/ingest/documents.py), [벡터 파일 업로드](../app/ingest/local_vectors.py), [장부 코드](../app/rag/catalog.py) |
| 공공 데이터 `site_id`: 장부 NULL, Qdrant 0 | 부분 구현 | 장부 연계 적재에서는 적용·검증하나 모든 별도 업서트 경로에서 강제하지는 않는다. [장부 검증](../app/rag/catalog.py), [JSONL 업서트](../scripts/upsert_qdrant_jsonl.py) |
| 5축 태깅 및 `tag_codes` 일치 | 부분 구현 | 코드표와 태그 필드는 있으나 태그가 비어 있는 데이터가 있고 모든 payload 값을 업무 DB 코드표와 대조해 강제하지 않는다. [사고 태그](../app/chunking/accident_ingest/tags.py), [적재 요청](../app/ingest_main.py) |
| 에이전트의 5축 필터 검색 | 부분 구현 | 백엔드는 필터를 지원하지만 기본 근거 수집기는 작업 맥락에 맞는 5축 값을 전달하지 않는다. [RAG 필터](../app/rag/filters.py), [근거 검색 설정](../app/agent/evidence_collector/vocab.py) |
| 사내 아차사고의 RAG 제외 | 부분 구현 | 기본 Spring 설정은 적재를 끄고 검색은 `site_id=0`인 공공 데이터만 조회한다. 다만 사내 아차사고 적재 코드 자체는 남아 있다. [Spring 설정](../../main-server/src/main/resources/application.yaml), [아차사고 적재 리스너](../../main-server/src/main/java/com/example/backend/nearmiss/kb/NearMissKbListener.java) |
| 근거 수집의 계획·검색·충분성·재질의 | 구현 | 최초 검색 후 기본 최대 3회 재질의한다. 필수 근거 기준은 임시값으로 안전 전문가 검증이 필요하다. [근거 수집기](../app/agent/evidence_collector/collector.py), [필수 근거 기준](../app/agent/evidence_collector/vocab.py) |
| KB 인용을 Spring에 저장 | 부분 구현 | 유효한 출처·컬렉션·UUID·원문이 있으면 저장한다. 필수 출처 정보가 없거나 유효하지 않으면 인용 테이블 저장을 건너뛰고 응답 JSON에만 남긴다. [인용 저장](../../main-server/src/main/java/com/example/backend/agent/service/AgentResultWriter.java) |
| Qdrant 검색·적재 권한 분리 | 부분 구현 | 검색에는 읽기 전용 키, 적재에는 쓰기 키를 사용하도록 구성한다. `kb_reader`는 정의되어 있으나 에이전트 검색은 현재 장부를 직접 읽지 않는다. 실제 배포 권한은 검증하지 않았다. [RAG 설정](../app/rag/config.py), [DB 계정 생성](../../main-infra-manage/db/init/10-agent-kb.sh) |
| MSDS 공단 API 조회 | 기본 경로 미연결 | 실제 API 클라이언트와 메모리 캐시는 있으나 작업 등록 도구에 등록되지 않아 기본 경로는 mock이다. `msds_cache` 테이블은 있지만 해당 클라이언트는 메모리 캐시를 사용한다. [MSDS 클라이언트](../app/publicdata/msds.py), [도구 등록](../app/tools/tool_setup.py) |
| Spring `check_rules` 판정 | 기본 경로 미연결 | Spring 엔드포인트는 있으나 AI 작업 등록의 도구 레지스트리는 mock을 사용한다. [Spring 내부 도구](../../main-server/src/main/java/com/example/backend/rule/controller/InternalRuleToolController.java), [도구 등록](../app/tools/tool_setup.py) |
| Spring `search_records` 조회 | 구현 | 양쪽 코드가 있으나 현재 Compose는 AI 서비스에 내부 API 주소와 토큰을 전달하지 않는다. [기록 조회 의존성](../app/api/v1/agent/record_query_deps.py), [Compose 설정](../../main-infra-manage/docker/docker-compose.yml) |
| Qdrant 포인트 삭제 방식 | 구현 | 현재 코드는 `POST /collections/{collection}/points/delete`를 사용한다. 실제 서버에 대한 삭제 동작은 이번 점검에서 실행하지 않았다. [Qdrant 삭제](../app/rag/qdrant.py) |
| 동일 원본의 재적재 건너뛰기 | 일부 경로 구현 | 법령 적재 스크립트는 해시·상태·청크 수를 비교해 건너뛴다. 모든 적재 API의 공통 동작은 아니다. [법령 적재](../scripts/ingest_law.py) |
| 검색 품질 측정 | 구현 | JSONL 정답 세트로 hit@k·MRR을 측정한다. 원문에서 제안한 `agent_kb.eval_cases`를 사용하는 방식과는 다르다. [검색 평가](../scripts/eval_retrieval.py) |
| 동의어 질의 확장·리랭커 | 부분 구현 | 작업 유형 정규화 동의어는 있지만 검색어 확장과 검색 결과 리랭커는 확인하지 못했다. [작업 유형 동의어](../app/agent/evidence_collector/vocab.py) |
| 공용 환경 적재 건수 | 확인 불가 | 코드만으로는 현재 공용 Qdrant의 데이터 건수를 판단할 수 없다. 대상 환경에서 별도 조회가 필요하다. |

### 7-2. 추가 차이 및 운영 확인 사항

- **RAG 사용 위치:** 원문은 작업 등록의 근거 수집 단계에서만 RAG를 쓴다고 설명하지만, 시나리오 분류 경로에서도 선택적으로 참고 자료 검색을 사용한다. [시나리오 분류](../app/agent/scenario/scenario_classifier.py)
- **Compose healthcheck:** 현재 Compose의 임베딩·적재 서비스 healthcheck는 `/actuator/health`를 호출하지만, 현재 두 앱의 준비 확인 경로는 `/ready`다. 임베딩 서비스는 적재 서비스의 healthy 상태에 의존하므로 새로 기동할 때 확인이 필요하다. [Compose 설정](../../main-infra-manage/docker/docker-compose.yml), [임베딩 준비 확인](../app/rag/embedding_api.py), [적재 준비 확인](../app/ingest_main.py)
- **인용 보장:** 근거 없는 위험요인은 삭제되는 대신 검토 필요로 표시된다. 따라서 모든 위험요인에 출처가 반드시 있다는 보장은 없다. [자기검증](../app/agent/judgment.py)
- **검증 범위:** 앞서 실행한 관련 테스트 88개는 통과했다. 이번 문서 점검에서는 실제 공용 데이터와 전체 서비스의 종단 간 동작을 재검증하지 않았다.