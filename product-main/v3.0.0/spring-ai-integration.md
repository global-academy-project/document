# Spring Boot와 AI 서비스 연동 현황

## 목적 및 범위

이 문서는 Spring Boot, AI API, 임베딩 API, Qdrant 사이의 실제 요청 경로와 개발 Compose에서 남아 있는 연결 작업을 기록한다. Dockerfile 및 개발 빌드 변경 기록은 [docker-update.md](docker-update.md)를 참고한다.

## 예상 요청 경로

```text
Spring Boot
  -> AI API (agent work-register / record-query)
      -> RAG 검색 도구
          -> Embedding API (/v1/embeddings)
          -> Qdrant
```

Spring의 `FastApiAgentClient`는 `APP_AGENT_BASE_URL`을 기준으로 다음 경로를 호출하고, `X-Internal-Token` 헤더를 보낸다.

- `POST /api/v1/agent/work-register`
- `POST /api/v1/agent/record-query`

AI 측 RAG 검색 백엔드는 `EMBEDDING_URL`, `EMBEDDING_MODEL`, `EMBEDDING_DIM`과 Qdrant 연결 설정을 읽는다. 임베딩 호출은 JSON의 `model`과 `input`을 `/v1/embeddings`로 전송하고, 설정된 API 키가 있으면 Bearer 인증을 사용한다.

## 현재 개발 Compose 상태

현재 개발 Compose의 서비스명과 컨테이너 포트는 다음과 같다.

| 역할 | Compose 서비스명 | 컨테이너 포트 |
| --- | --- | ---: |
| Spring Boot | `spring-boot-application` | 60001 |
| AI API | `python-ai-model` | 60100 |
| 임베딩 API | `python-embedding-model` | 60110 |
| Qdrant | `qdrant-server` | 6333 |

Compose 네트워크 안에서 호출할 때는 호스트에 공개한 포트가 아니라 Compose 서비스명과 컨테이너 포트를 사용한다.

## 아직 해결해야 할 연결 불일치

1. `spring-boot-application`의 `APP_AGENT_BASE_URL`은 현재 `http://ai-model-server:60100`을 가리키지만, 개발 Compose에 등록된 실제 AI 서비스명은 `python-ai-model`이다. 개발 네트워크에서는 `http://python-ai-model:60100`으로 맞춰야 한다.
2. `python-ai-model` 컨테이너 환경에는 현재 RAG 검색에 필요한 `EMBEDDING_URL`, `EMBEDDING_MODEL`, `EMBEDDING_DIM`, `QDRANT_URL`, Qdrant 인증 정보가 전달되지 않는다. 개발 Compose에서 이를 AI 서비스에 전달해야 한다.
3. AI 컨테이너의 임베딩 주소는 `http://python-embedding-model:60110/v1/embeddings`여야 한다. 호스트에서는 `${EMBEDDING_MODEL_PORT}`로 노출되더라도 컨테이너 간 호출에는 내부 포트 60110을 쓴다.
4. Spring 클라이언트가 호출하는 두 agent 경로와 AI API의 실제 라우트 및 요청/응답 JSON 계약이 일치하는지 통합 테스트로 확인해야 한다.
5. Spring에서 전달하는 `INTERNAL_API_TOKEN`과 AI API가 기대하는 인증 설정이 같은 값으로 연결되는지 확인해야 한다.

현재 `kb-migrator`의 임베딩 주소는 개발 Compose의 실제 서비스명 `python-embedding-model`과 포트 60110을 사용하도록 수정되어 있다. 이는 migrator와 임베딩 서버 간 설정이며, Spring→AI API→RAG 전체 경로가 검증됐다는 뜻은 아니다.

## 검증 계획

1. 개발 Compose에서 Spring의 AI API base URL과 AI 서비스의 RAG 환경 변수를 실제 서비스명에 맞춘다.
2. `docker compose config`로 설정을 확인하고 관련 서비스만 빌드·재생성한다.
3. AI API health endpoint, 임베딩 `/ready`, Qdrant readiness를 확인한다.
4. Spring API에 실제 agent 요청을 보내 `FastApiAgentClient`가 AI API에서 정상 응답을 받는지 확인한다.
5. 해당 요청에서 RAG 검색이 실행되도록 테스트해 AI API가 임베딩 API와 Qdrant를 실제 호출하는지 확인한다.
6. 실패 시 컨테이너별 DNS 이름, 내부 포트, 환경 변수 전달, 인증 토큰, API payload 계약을 순서대로 확인한다.
7. 개발 통합 테스트가 통과한 뒤 검증된 연결 설정만 운영 Compose에 반영한다.

## 현재 검증 상태

- 임베딩 API 단독 테스트: 완료. BGE-M3 응답에서 4개 문장 벡터, 1024차원, 정규화 값과 코사인 유사도를 확인했다.
- AI API/임베딩/ingest 이미지 빌드와 모듈 import: 완료.
- Spring Boot → AI API 호출: 미검증.
- AI API RAG → 임베딩 API 및 Qdrant 호출: 미검증.
- 운영 Compose 반영: 미수행.