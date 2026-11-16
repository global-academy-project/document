# Docker Compose Configuration

이 문서는 `docker/docker-compose.yml`과 `main-ai-model/Dockerfile`에 반영된 개발 환경 설정을 설명합니다. 비밀값은 저장소에 기록하지 말고 Compose가 읽는 `.env` 파일 또는 배포 환경의 secret으로 주입하세요.

## 최근 변경 사항

- MySQL healthcheck에서 비밀번호를 `--password="$$MYSQL_ROOT_PASSWORD"`로 전달하도록 수정했습니다. `$$`는 Compose 보간을 피하고 컨테이너 셸에서 환경변수를 확장하게 합니다.
- Python AI, embedding, ingest 서비스의 healthcheck를 `curl` 대신 Python 표준 라이브러리 `urllib`로 실행합니다. AI는 `/health`, embedding과 ingest는 `/ready`를 확인합니다.
- Qdrant healthcheck는 컨테이너 안의 TCP 6333 연결을 확인하며, Qdrant는 시작 후 healthy 상태임을 확인했습니다.
- ML 이미지의 `/models/huggingface` 캐시 디렉터리를 만들고 UID/GID `65003:65003`에 쓰기 권한을 줍니다. 이 경로는 비-root AI 및 embedding 컨테이너가 사용합니다.
- `qdrant-init`은 root 권한으로 가상환경과 의존성을 설치한 뒤 UID/GID `65003:65003`으로 실행합니다.
- ingest 이미지에 공유 `app/rag` 코드와 SIF 프로필 코드를 포함해 앱 시작 시 필요한 import를 사용할 수 있게 했습니다.
- 최근 수정은 healthcheck와 이미지 구성에 대한 변경입니다. 이 변경으로 새 Compose 환경변수가 추가된 것은 없습니다.

## 환경변수

아래는 현재 `docker-compose.yml`에 서비스별로 선언된 컨테이너 환경변수입니다. `${NAME}` 형식은 Compose 실행 환경이나 `.env`에서 값을 받습니다. `:?`가 붙은 변수는 누락되거나 빈 값이면 Compose가 오류를 냅니다. `:-값`은 변수가 없거나 비었을 때 기본값을 사용합니다. 기본값 없는 `${NAME}`은 설정이 없으면 빈 값으로 대체될 수 있으므로 표의 필수 시크릿은 반드시 설정하세요.

| 서비스 | 변수 | 설정/용도 |
| --- | --- | --- |
| `vite-web-application` | `VITE_API_PROXY_TARGET` | 내부 Spring 주소 `http://spring-boot-application:60001` |
| `spring-boot-application` | `MYSQL_HOST`, `MYSQL_PORT` | MySQL 서비스 주소와 포트 (`database-mysql-server`, `3306`) |
| `spring-boot-application` | `APP_OCR_BASE_URL`, `APP_AGENT_BASE_URL`, `APP_AGENT_CLIENT` | Python AI API 주소 및 클라이언트 (`fastapi`) |
| `spring-boot-application` | `STT_MODEL_NAME`, `STT_MODEL_PATH` | STT 모델 이름과 캐시 내 경로 |
| `spring-boot-application` | `VISION_MODEL_PATH`, `VISION_HELMET_MODEL_PATH` | 일반/헬멧 검출 ONNX 모델 경로 |
| `spring-boot-application` | `LLM_ADAPTERS` | 선택 설정, 기본값은 빈 문자열 |
| `spring-boot-application` | `ADMIN_EMAIL`, `ADMIN_PASSWORD` | 선택 설정, 기본값은 빈 문자열 |
| `spring-boot-application` | `MYSQL_DATABASE`, `MYSQL_PASSWORD`, `INTERNAL_API_TOKEN`, `JWT_SECRET` | DB 접속 및 내부 API/JWT 설정. `INTERNAL_API_TOKEN`은 Compose에서 필수 검사됨 |
| `python-ai-model` | `HF_HOME` | Hugging Face 캐시 경로 `/models/huggingface` |
| `python-ai-model` | `RAG_BACKEND`, `RAG_SEARCH_TIMEOUT_SECONDS` | RAG backend 경로 및 검색 제한시간 (`60`초) |
| `python-ai-model` | `EMBEDDING_URL`, `EMBEDDING_MODEL`, `EMBEDDING_DIM` | 임베딩 서비스 주소, 모델, 벡터 차원 |
| `python-ai-model` | `QDRANT_URL`, `QDRANT_READ_ONLY_API_KEY` | Qdrant 주소 및 읽기 전용 키 |
| `python-ai-model` | `NETWORK_ENV`, `INTERNAL_API_BASE_URL`, `INTERNAL_API_TOKEN` | 네트워크 모드, Spring 주소, 내부 API 인증. `INTERNAL_API_TOKEN`은 필수 |
| `python-ai-model` | `CHECK_RULES_TOOL`, `EMBEDDING_API_KEY`, `KOSHA_MSDS_SERVICE_KEY` | 규칙 도구 및 외부 서비스 설정. 규칙 도구 기본값은 `MOCK`; 나머지는 선택값 |
| `python-embedding-model` | `HF_HOME`, `EMBEDDING_MODEL`, `EMBEDDING_DIM` | 모델 캐시 경로, 모델 이름, 벡터 차원 |
| `python-embedding-model` | `EMBEDDING_MAX_TOKENS`, `EMBEDDING_MAX_BATCH_SIZE`, `EMBEDDING_LOCAL_FILES_ONLY`, `TZ` | 임베딩 제한 및 실행 환경 (`8192`, `8`, `false`, `Asia/Seoul`) |
| `python-embedding-model` | `EMBEDDING_API_KEY` | 임베딩 API 인증 키 |
| `database-mysql-server` | `TZ`, `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`, `MYSQL_ROOT_PASSWORD` | 기본 DB와 사용자 설정, root 비밀번호 |
| `database-mysql-server` | `KB_OWNER_PASSWORD`, `KB_INGEST_PASSWORD`, `KB_READER_PASSWORD` | Knowledge Base 계정별 비밀번호 |
| `kb-migrator` | `KB_OWNER_USER`, `FLYWAY_PASSWORD` | 마이그레이션 계정 (`kb_owner`)과 비밀번호. `FLYWAY_PASSWORD`는 필수 검사됨 |
| `kb-migrator` | `EMBEDDING_URL`, `EMBEDDING_MODEL`, `EMBEDDING_DIM`, `EMBEDDING_MAX_TOKENS`, `EMBEDDING_MAX_BATCH_SIZE`, `EMBEDDING_API_KEY` | Migration task의 임베딩 서비스 설정 |
| `qdrant-server` | `QDRANT__SERVICE__API_KEY`, `QDRANT__SERVICE__READ_ONLY_API_KEY`, `DEBUG` | Qdrant 쓰기/읽기 키와 디버그 설정. 키는 각각 `QDRANT_API_KEY`, `QDRANT_READ_ONLY_API_KEY`에서 주입되며 Compose가 필수 검사함 |
| `qdrant-init` | `QDRANT_URL`, `QDRANT_API_KEY`, `EMBEDDING_DIM` | Qdrant 초기화 주소, 쓰기 키, 벡터 차원. 키는 필수 검사됨 |
| `kb-ingest` | `QDRANT_URL`, `QDRANT_API_KEY`, `EMBEDDING_URL`, `EMBEDDING_API_KEY` | Qdrant/임베딩 연결 및 인증 |
| `kb-ingest` | `EMBEDDING_MODEL`, `EMBEDDING_DIM`, `RAG_SEARCH_TIMEOUT_SECONDS` | 임베딩 및 검색 설정 |
| `kb-ingest` | `KB_DB_HOST`, `KB_DB_PORT`, `KB_DATABASE`, `KB_INGEST_PASSWORD`, `KB_INGEST_API_TOKEN` | Knowledge Base DB 연결 및 ingest 인증. `KB_INGEST_API_TOKEN`은 필수 검사됨 |
| `kb-ingest` | `LOCAL_CHUNK_ROOT`, `LOCAL_VECTOR_DIR` | 컨테이너 내부 입력/벡터 디렉터리 (`/data/out`, `/data/vectors`) |

## 모델 볼륨

- `ai-model-cache`는 Spring Boot와 `python-ai-model`에서 `/models/huggingface`로 마운트합니다. Spring의 `VISION_MODEL_PATH`, `VISION_HELMET_MODEL_PATH`, `STT_MODEL_PATH`는 이 볼륨 내부의 파일 경로입니다.
- `embedding-model-cache`는 embedding 서비스에서 별도로 `/models/huggingface`에 마운트합니다.
- Compose named volume의 이름은 저장소 식별자이고, 모델 경로는 컨테이너 내부 경로입니다. 볼륨 안에 설정된 상대 경로로 실제 모델 파일이 있어야 합니다.
- 이미 만들어진 볼륨의 소유권은 새 이미지 빌드만으로 변경되지 않습니다. 권한 문제가 발생하면 해당 볼륨을 한 번 UID/GID `65003:65003`으로 조정해야 합니다.

## 확인 명령

Compose 설정을 검사하고 Python 서비스 상태를 확인할 때는 저장소 루트에서 실행합니다.

```powershell
docker compose --env-file .\docker\.env -f .\docker\docker-compose.yml config --quiet
docker compose --env-file .\docker\.env -f .\docker\docker-compose.yml up -d --build --wait python-ai-model python-embedding-model kb-ingest
```
