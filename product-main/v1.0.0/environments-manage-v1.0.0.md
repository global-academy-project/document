# Docker 환경 변수 관리 변경 내역 v1.0.0

- 작성일: 2026-11-01
- 기준 파일: [docker-compose.dev.yml](../../../main-infra-manage/docker/docker-compose.dev.yml)
- 대상 파일: [docker-compose.yml](../../../main-infra-manage/docker/docker-compose.yml), [.env.example](../../../main-infra-manage/docker/.env.example)

> 두 Compose 파일의 활성 설정은 동일합니다. `docker-compose.yml` 끝에는 이전 구성이 주석으로 남아 있으며, 실행에는 영향을 주지 않습니다.

## 1. 변경 목적

- 개발자가 직접 관리해야 하는 환경 변수 수를 줄인다.
- 컨테이너 내부 주소, 고정 계정명, 모델 설정처럼 환경별로 바뀌지 않는 값은 Compose에 고정한다.
- 비밀번호, 토큰, API 키, 호스트 공개 포트처럼 환경별로 달라지는 값만 `.env`에서 관리한다.
- 서비스마다 같은 의미의 값을 다르게 설정해 생기는 불일치를 방지한다.

## 2. 관리 원칙

Compose의 `environment`는 두 그룹으로 구분한다.

| 구분 | 의미 | 관리 위치 |
|---|---|---|
| Public bind environment | 비밀이 아니고 환경별로 바뀌지 않는 값 | Compose에 직접 작성 |
| Private bind environment | 비밀값 또는 환경별로 달라지는 값 | `.env`에서 `${VAR}`로 주입 |

추가 원칙:

- 컨테이너 간 통신은 **서비스 이름 + 컨테이너 내부 포트**를 사용한다.
  - 예: `database-mysql-server:3306`, `qdrant-server:6333`
- `ports:`는 Docker 밖(호스트 PC, DB 도구, 외부 서버)에서 접근할 때만 필요하다.
- 호스트 공개 포트 변수(`MYSQL_PORT` 등)를 컨테이너 간 접속 주소에 사용하지 않는다.
- 용도가 다른 비밀값은 값이 같아 보이더라도 하나로 합치지 않는다.

## 3. 주요 변경 사항

### 3.1 `.env` 관리 대상 축소

현재 Compose에서 참조하는 환경 변수는 **22개**이다.

### 3.2 Compose에 고정한 값

| 서비스 | 고정한 값 | 이유 |
|---|---|---|
| `vite-web-application` | `VITE_API_PROXY_TARGET=http://spring-boot-application:60001` | 웹 컨테이너 내부의 `localhost`는 Spring이 아님 |
| `spring-boot-application` | `MYSQL_HOST=database-mysql-server`, `MYSQL_PORT=3306` | Compose 내부 DB 주소 |
| `python-ai-model`, `python-embedding-model` | `HF_HOME=/models/huggingface` | 모델 캐시 볼륨 경로와 일치 |
| `python-ai-model`, `python-embedding-model`, `kb-migrator` | `EMBEDDING_MODEL=BAAI/bge-m3`, `EMBEDDING_DIM=1024` | 모델과 벡터 차원 불일치 방지 |
| `python-embedding-model`, `kb-migrator` | `EMBEDDING_MAX_TOKENS=8192`, `EMBEDDING_MAX_BATCH_SIZE=8` | 서비스 간 임베딩 설정 통일 |
| `python-ai-model` | `RAG_BACKEND`, `RAG_SEARCH_TIMEOUT_SECONDS=60` | RAG 검색을 수행하는 서비스에만 설정 (3.5 참고) |
| `python-ai-model` | `EMBEDDING_URL=http://python-embedding-model:60110/v1/embeddings`, `QDRANT_URL=http://qdrant-server:6333` | RAG 검색 시 호출할 내부 주소 |
| `kb-migrator` | `kb_owner`, `jdbc:mysql://database-mysql-server:3306/agent_kb` | [초기화 스크립트](../../../main-infra-manage/db/init/10-agent-kb.sh)가 생성하는 계정과 DB에 고정 |
| `kb-ingest` | `QDRANT_URL=http://qdrant-server:6333`, `KB_DB_HOST`, `KB_DATABASE=agent_kb` | Compose 내부 주소 |
| `database-mysql-server`, `python-embedding-model` | `TZ=Asia/Seoul` | 시간대 통일 |

### 3.3 데이터 볼륨 분리

| 볼륨 | 사용 서비스 | 경로 |
|---|---|---|
| `database-data` | MySQL | `/var/lib/mysql` |
| `qdrant-data` | Qdrant | `/qdrant/storage` |
| `ai-model-cache` | AI 모델 서버 | `/models/huggingface` |
| `embedding-model-cache` | 임베딩 서버 | `/models/huggingface` |

MySQL과 Qdrant가 같은 볼륨을 공유하면 MySQL이 데이터 디렉터리를 초기화하지 못하고 재시작을 반복한다. 데이터 볼륨은 서비스별로 분리한다.

### 3.4 MySQL 실행 옵션

```yaml
command:
  - mysqld
  - --log-bin-trust-function-creators=1
  - --default-time-zone=+09:00
```

`--log-bin-trust-function-creators=1`은 Spring Flyway V9 마이그레이션의 트리거 생성에 필요하다.

### 3.5 RAG 설정 위치 정리

임베딩 서버는 벡터를 **만드는 쪽**이고, AI 모델 서버는 임베딩 서버와 Qdrant를 **호출하는 쪽**이다. 호출 주소와 검색 설정은 호출하는 서비스에만 둔다.

| 설정 | 임베딩 서버 | AI 모델 서버 |
|---|---|---|
| `EMBEDDING_MODEL`, `EMBEDDING_DIM`, `EMBEDDING_API_KEY` | 필요 | 필요 |
| `EMBEDDING_MAX_TOKENS`, `EMBEDDING_MAX_BATCH_SIZE`, `EMBEDDING_LOCAL_FILES_ONLY`, `HF_HOME` | 필요 | 불필요 (`HF_HOME`은 자체 모델 캐시용) |
| `EMBEDDING_URL`, `RAG_BACKEND`, `RAG_SEARCH_TIMEOUT_SECONDS` | 불필요 | 필요 |
| `QDRANT_URL`, `QDRANT_READ_ONLY_API_KEY` | 불필요 | 필요 |

근거 (`main-ai-model`):

- 임베딩 서버는 `EmbeddingServerSettings`로 모델 서빙 설정만 읽는다. 클래스 설명에 "임베딩 엔드포인트나 Qdrant 접근이 필요 없다"고 명시되어 있다.
- AI 모델 서버의 `Settings.rag_backend` 기본값은 `None`이며, 이 경우 mock 검색 도구를 사용한다.
- `RagSettings.from_env()`는 `EMBEDDING_URL`이 없으면 오류를 낸다. `QDRANT_URL`이 없으면 코드 기본값 `http://qdrant:6333`을 사용하는데, 이 주소는 실제 서비스 이름 `qdrant-server`와 다르다.

변경 내용:

| 서비스 | 변경 |
|---|---|
| `python-ai-model` | `EMBEDDING_URL`을 존재하지 않는 `embedding-server`에서 `python-embedding-model`로 수정 |
| `python-ai-model` | `RAG_BACKEND`, `RAG_SEARCH_TIMEOUT_SECONDS`, `EMBEDDING_MODEL`, `EMBEDDING_DIM`, `QDRANT_URL`, `EMBEDDING_API_KEY`, `QDRANT_READ_ONLY_API_KEY` 추가 |
| `python-embedding-model` | 읽지 않는 `EMBEDDING_URL`, `RAG_BACKEND`, `RAG_SEARCH_TIMEOUT_SECONDS` 제거 |

> **동작 변경:** 이전에는 AI 모델 서버에 `RAG_BACKEND`가 없어 검색 도구가 mock으로 동작했다. 이제 실제 RAG 검색(임베딩 서버 → Qdrant)을 사용한다. Qdrant에 컬렉션과 데이터가 적재되지 않았거나 임베딩 서버가 준비되지 않았다면 검색 호출 시 오류가 발생한다. mock 검색으로 되돌리려면 AI 모델 서버에서 `RAG_BACKEND`를 제거한다.

## 4. 현재 `.env` 관리 변수 (22개)

### 4.1 호스트 공개 포트

| 변수 | 사용 서비스 | 컨테이너 포트 |
|---|---|---|
| `WEB_FRONTEND_PORT` | `vite-web-application` | `50001` |
| `AI_MODEL_PORT` | `python-ai-model` | `60100` |
| `EMBEDDING_MODEL_PORT` | `python-embedding-model` | `60110` |
| `MYSQL_PORT` | `database-mysql-server` | `3306` |
| `QDRANT_PORT` | `qdrant-server` | `6333` |
| `KB_INGEST_PORT` | `kb-ingest` | `8001` |

호스트에서 직접 접근할 필요가 없는 서비스는 포트를 제거해도 컨테이너 간 통신에는 영향이 없다. 로컬 도구로만 접근한다면 `127.0.0.1:<호스트포트>:<컨테이너포트>` 형식으로 로컬에만 공개한다.

### 4.2 MySQL 및 KB 계정

| 변수 | 사용 서비스 | 비밀 |
|---|---|---|
| `MYSQL_DATABASE` | Spring, MySQL | 아니오 |
| `MYSQL_USER` | Spring, MySQL | 아니오 |
| `MYSQL_PASSWORD` | Spring, MySQL | 예 |
| `MYSQL_ROOT_PASSWORD` | MySQL | 예 |
| `KB_OWNER_PASSWORD` | MySQL, `kb-migrator` | 예 |
| `KB_INGEST_PASSWORD` | MySQL, `kb-ingest` | 예 |
| `KB_READER_PASSWORD` | MySQL | 예 |

### 4.3 인증 및 API 키

| 변수 | 사용 서비스 | 필수 |
|---|---|---|
| `JWT_SECRET` | Spring | 예 |
| `INTERNAL_API_TOKEN` | Spring | 예 |
| `QDRANT_API_KEY` | Qdrant, `kb-ingest` | 예 |
| `QDRANT_READ_ONLY_API_KEY` | Qdrant, AI 모델 서버 | 예 |
| `KB_INGEST_API_TOKEN` | `kb-ingest` | 예 (`:?` 검사) |
| `EMBEDDING_API_KEY` | 임베딩 서버, AI 모델 서버, `kb-migrator` | 아니오 (기본값 빈 문자열) |

### 4.4 기능 선택

| 변수 | 사용 서비스 | 기본값 |
|---|---|---|
| `APP_AGENT_CLIENT` | Spring | `fastapi` |
| `NETWORK_ENV` | AI 모델 서버 | `OPEN` |
| `CHECK_RULES_TOOL` | AI 모델 서버 | `MOCK` |

## 5. `.env.example` 정리 필요 항목

현재 [.env.example](../../../main-infra-manage/docker/.env.example)은 Compose와 일치하지 않는다.

**Compose에서 사용하지만 예시에 없는 변수 (14개)**

```text
AI_MODEL_PORT, APP_AGENT_CLIENT, CHECK_RULES_TOOL, EMBEDDING_API_KEY,
EMBEDDING_MODEL_PORT, INTERNAL_API_TOKEN, KB_INGEST_API_TOKEN,
KB_INGEST_PASSWORD, KB_INGEST_PORT, KB_OWNER_PASSWORD, KB_READER_PASSWORD,
QDRANT_API_KEY, QDRANT_PORT, QDRANT_READ_ONLY_API_KEY
```

**예시에 있지만 Compose에서 사용하지 않는 변수 (21개)**

```text
AI_MAX_OUTPUT_TOKENS, AI_TIMEOUT_SECONDS, APP_OCR_BASE_URL, APP_OCR_CLIENT,
DEFAULT_LLM_ENGINE, GO_SERVER_PORT, MYSQL_HOST, OPENAI_API_KEY, OPENAI_MODEL,
POSTGRES_DB, POSTGRES_HOST, POSTGRES_PASSWORD, POSTGRES_PORT, POSTGRES_USER,
PYTHON_SERVER_PORT, SPRING_MYSQL_DATABASE, SPRING_MYSQL_HOST,
SPRING_MYSQL_PASSWORD, SPRING_MYSQL_PORT, SPRING_MYSQL_USER, SPRING_SERVER_PORT
```

`OPENAI_*`는 실제 LLM 호출이 필요하다면 삭제하지 말고 AI 모델 서버의 Private bind environment에 추가해야 한다.

예시 파일의 `JWT_SECRET`에는 실제 값처럼 보이는 문자열이 들어 있으므로 빈 값이나 자리표시자로 교체한다.

## 6. 남은 확인 사항

현재 설정에서 아래 항목은 의도와 다르게 동작할 수 있다.

| 위치 | 현재 값 | 확인 내용 |
|---|---|---|
| Qdrant의 `QDRANT_API_KEY`, `QDRANT_READ_ONLY_API_KEY` | 일반 환경 변수 이름으로 전달 | Qdrant 서버 설정은 `QDRANT__SERVICE__API_KEY`, `QDRANT__SERVICE__READ_ONLY_API_KEY` 형식으로 읽음. 현재 이름으로는 API 키 인증이 적용되지 않을 수 있음 |
| Spring의 `APP_OCR_BASE_URL`, `APP_AGENT_BASE_URL` | `http://ai-model-server:60100` | Compose에 `ai-model-server` 서비스가 없음. `python-ai-model`이 맞는지 확인 |
| `kb-ingest`의 `KB_DB_PORT` | `${MYSQL_PORT}` | 호스트 공개 포트를 참조함. 컨테이너 간 통신은 `3306`이어야 함 |
| Spring의 `ADMIN_PASSWORD` | Compose에 직접 작성 | 비밀값이므로 `.env`로 이동 필요 |
| Spring `depends_on` | 없음 | MySQL보다 먼저 시작하면 DB 연결 실패 가능. MySQL healthcheck와 함께 구성 권장 |

## 7. 검증 방법

저장소 루트에서 실행한다.

```powershell
docker compose --env-file .\docker\.env -f .\docker\docker-compose.dev.yml config --quiet
```

- 구문 오류와 `:?`로 지정한 필수 변수 누락을 확인한다.
- 값의 유효성이나 서비스 간 실제 연결은 확인하지 않는다.

환경 파일의 변수 참조(`${OTHER_VAR}`)는 Docker Compose가 읽을 때만 해석된다. 애플리케이션의 dotenv 라이브러리나 `docker run --env-file`에서 같은 동작을 가정하지 않는다.
