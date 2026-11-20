# Docker Compose 환경 변수와 Kubernetes 설정 사용 영역

## 목적

이 문서는 `docker/docker-compose.yml`에 정의된 환경 변수를 서비스별로 정리하고,
쿠버네티스 ConfigMap과 Secret으로 이관할 때의 사용 영역을 문서화한다.

기준 원칙은 다음과 같다.

- ConfigMap: 비밀이 아닌 설정값
- Secret: 비밀번호, 토큰, API 키
- 공통 ConfigMap/Secret: 실제로 여러 독립 서비스가 소비하는 값만 포함
- 서비스 전용 ConfigMap/Secret: 단일 서비스 또는 단일 도메인에서만 소비하는 값
- Kubernetes 내부 주소는 Compose 주소가 아니라 실제 Service 이름과 포트로 정의

## 1. Public과 Private 분류

### Public 영역

Public은 비밀번호나 토큰이 아닌 설정값이다.

- ConfigMap에 저장한다.
- 환경 변수명과 값을 Git에서 관리할 수 있다.
- 내부 주소, 포트, 모델명, 차원, 로케일, 인코딩, 배치 크기 등이 포함된다.
- 여러 서비스가 소비하면 공통 ConfigMap에 둔다.
- 특정 서비스만 소비하면 해당 서비스 ConfigMap에 둔다.

Public 설정에 포함되는 환경 변수는 다음과 같다.

| 분류 | 환경 변수 |
|---|---|
| 시스템 공통 | `TZ`, `LANG`, `LC_ALL` |
| 서버 주소 | `VITE_API_PROXY_TARGET`, `MYSQL_HOST`, `MYSQL_PORT`, `APP_OCR_BASE_URL`, `APP_AGENT_BASE_URL`, `INTERNAL_API_BASE_URL`, `EMBEDDING_URL`, `QDRANT_URL`, `KB_DB_HOST`, `KB_DB_PORT` |
| 데이터베이스 | `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_ENCODING_TYPE`, `MYSQL_COLLATION`, `KB_OWNER_USER`, `KB_DATABASE` |
| 임베딩 | `EMBEDDING_MODEL`, `EMBEDDING_DIM`, `EMBEDDING_MAX_TOKENS`, `EMBEDDING_MAX_BATCH_SIZE`, `EMBEDDING_LOCAL_FILES_ONLY` |
| 모델 | `STT_MODEL_NAME`, `APP_AGENT_CLIENT`, `LLM_ADAPTERS`, `RAG_BACKEND`, `RAG_SEARCH_TIMEOUT_SECONDS`, `NETWORK_ENV`, `CHECK_RULES_TOOL`, `MSDS_TOOL` |
| 모델 경로 | `HF_HOME`, `VISION_MODEL_PATH`, `VISION_HELMET_MODEL_PATH`, `STT_MODEL_PATH` |
| 로컬 작업 경로 | `LOCAL_CHUNK_ROOT`, `LOCAL_VECTOR_DIR` |
| 애플리케이션 설정 | `JAVA_TOOL_OPTIONS`, `DEBUG` |

### Private 영역

Private은 외부 노출 또는 Git 저장을 피해야 하는 비밀 값이다.

- Kubernetes Secret에 저장한다.
- 운영 환경에서 별도로 생성하거나 외부 Secret 관리 도구로 주입한다.
- Secret 템플릿에는 실제 값 대신 placeholder만 둔다.
- 비밀번호가 여러 리소스에 동일하게 사용되더라도 실제 Secret은 용도별로 분리한다.

Private 설정에 포함되는 환경 변수는 다음과 같다.

| 분류 | 환경 변수 |
|---|---|
| Spring 및 내부 API | `JWT_SECRET`, `INTERNAL_API_TOKEN`, `ADMIN_EMAIL`, `ADMIN_PASSWORD` |
| MySQL | `MYSQL_PASSWORD`, `MYSQL_ROOT_PASSWORD` |
| Knowledge Base | `KB_OWNER_PASSWORD`, `KB_INGEST_PASSWORD`, `KB_READER_PASSWORD`, `KB_INGEST_API_TOKEN`, `FLYWAY_PASSWORD` |
| 임베딩 | `EMBEDDING_API_KEY` |
| Qdrant | `QDRANT_API_KEY`, `QDRANT_READ_ONLY_API_KEY`, `QDRANT__SERVICE__API_KEY`, `QDRANT__SERVICE__READ_ONLY_API_KEY` |
| 외부 서비스 | `KOSHA_MSDS_SERVICE_KEY` |

### Private과 Public 키 매핑

같은 비밀번호를 여러 이름으로 사용할 때는 Canonical key를 기준으로 Secret을 만들고,
애플리케이션이 요구하는 이름으로 Pod 환경에서 매핑한다.

| Canonical Private key | 사용처 환경 변수 |
|---|---|
| `KB_OWNER_PASSWORD` | `FLYWAY_PASSWORD`, `KB_OWNER_PASSWORD` |
| `MYSQL_PASSWORD` | `MYSQL_PASSWORD` |
| `MYSQL_ROOT_PASSWORD` | `MYSQL_ROOT_PASSWORD` |
| `QDRANT_API_KEY` | `QDRANT_API_KEY`, `QDRANT__SERVICE__API_KEY` |
| `QDRANT_READ_ONLY_API_KEY` | `QDRANT_READ_ONLY_API_KEY`, `QDRANT__SERVICE__READ_ONLY_API_KEY` |

## 2. 서비스별 분류

### Public 환경 변수를 사용하는 서비스

| 서비스 | 주요 Public 환경 변수 |
|---|---|
| Vite | `VITE_API_PROXY_TARGET` |
| Spring | `LANG`, `LC_ALL`, `JAVA_TOOL_OPTIONS`, `MYSQL_HOST`, `MYSQL_PORT`, `MYSQL_DATABASE`, `VISION_MODEL_PATH`, `VISION_HELMET_MODEL_PATH`, `STT_MODEL_PATH`, `STT_MODEL_NAME`, `APP_OCR_BASE_URL`, `APP_AGENT_BASE_URL`, `APP_AGENT_CLIENT`, `LLM_ADAPTERS` |
| Python AI | `RAG_BACKEND`, `RAG_SEARCH_TIMEOUT_SECONDS`, `EMBEDDING_MODEL`, `EMBEDDING_DIM`, `STT_MODEL_NAME`, `LLM_ADAPTERS`, `NETWORK_ENV`, `CHECK_RULES_TOOL`, `MSDS_TOOL`, `EMBEDDING_URL`, `INTERNAL_API_BASE_URL`, `QDRANT_URL`, `HF_HOME`, `VISION_MODEL_PATH`, `VISION_HELMET_MODEL_PATH`, `STT_MODEL_PATH` |
| Embedding | `HF_HOME`, `EMBEDDING_MODEL`, `EMBEDDING_DIM`, `EMBEDDING_MAX_TOKENS`, `EMBEDDING_MAX_BATCH_SIZE`, `EMBEDDING_LOCAL_FILES_ONLY`, `TZ` |
| MySQL | `TZ`, `LANG`, `LC_ALL`, `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_ENCODING_TYPE`, `MYSQL_COLLATION` |
| Flyway/KB Migrator | `KB_OWNER_USER`, `EMBEDDING_URL`, `EMBEDDING_MODEL`, `EMBEDDING_DIM`, `EMBEDDING_MAX_TOKENS`, `EMBEDDING_MAX_BATCH_SIZE` |
| Qdrant | `DEBUG` |
| Qdrant Init | `QDRANT_URL`, `EMBEDDING_DIM` |
| KB Ingest | `QDRANT_URL`, `EMBEDDING_URL`, `EMBEDDING_MODEL`, `EMBEDDING_DIM`, `RAG_SEARCH_TIMEOUT_SECONDS`, `KB_DB_HOST`, `KB_DATABASE`, `KB_DB_PORT`, `LOCAL_CHUNK_ROOT`, `LOCAL_VECTOR_DIR` |

### Private 환경 변수를 사용하는 서비스

| 서비스 | 주요 Private 환경 변수 |
|---|---|
| Spring | `JWT_SECRET`, `INTERNAL_API_TOKEN`, `ADMIN_EMAIL`, `ADMIN_PASSWORD`, `MYSQL_PASSWORD` |
| Python AI | `INTERNAL_API_TOKEN`, `EMBEDDING_API_KEY`, `QDRANT_READ_ONLY_API_KEY`, `KOSHA_MSDS_SERVICE_KEY` |
| Embedding | `EMBEDDING_API_KEY` |
| MySQL | `MYSQL_PASSWORD`, `MYSQL_ROOT_PASSWORD`, `KB_OWNER_PASSWORD`, `KB_INGEST_PASSWORD`, `KB_READER_PASSWORD` |
| Flyway/KB Migrator | `FLYWAY_PASSWORD`, `EMBEDDING_API_KEY` |
| Qdrant | `QDRANT_API_KEY`, `QDRANT_READ_ONLY_API_KEY` |
| Qdrant Init | `QDRANT_API_KEY` |
| KB Ingest | `QDRANT_API_KEY`, `EMBEDDING_API_KEY`, `KB_INGEST_PASSWORD`, `KB_INGEST_API_TOKEN` |

## 1. 전역 공통 설정

다음 값은 여러 Compose 서비스 또는 운영 기반 작업에서 공통으로 사용되는 설정이다.

| 환경 변수 | 값 | 사용하는 영역 | 권장 Kubernetes 리소스 |
|---|---|---|---|
| `TZ` | `Asia/Seoul` | MySQL, 임베딩 | `common-system-config` |
| `LANG` | `C.UTF-8` | Spring, MySQL 등 UTF-8 필요 서비스 | `common-system-config` |
| `LC_ALL` | `C.UTF-8` | Spring, MySQL 등 UTF-8 필요 서비스 | `common-system-config` |
| `JAVA_TOOL_OPTIONS` | Java 파일 인코딩 옵션 | Spring | Spring 전용 ConfigMap |

여러 서비스에 직접 주입할 수 있는 시스템 기본값은 `envFrom`으로 묶는 것이 적합하다.

다만 전역 ConfigMap에 모든 도메인 설정을 합치면 불필요한 환경 변수까지 주입될 수 있으므로,
`LANG`, `LC_ALL`, `TZ`처럼 진짜 공통 기본값만 남기는 것을 권장한다.

## 2. Vite 웹

| 환경 변수 | Compose 값 | 구분 | Kubernetes 이관 |
|---|---|---|---|
| `VITE_API_PROXY_TARGET` | `http://spring-boot-application:60001` | 공개 설정 | Vite 전용 ConfigMap 또는 Deployment env |

쿠버네티스에서는 Spring Service 이름과 포트를 사용한다.

```yaml
VITE_API_PROXY_TARGET: "http://spring-service:60001"
```

## 3. Spring Boot

| 환경 변수 | Compose 값 | 구분 | Kubernetes 이관 |
|---|---|---|---|
| `LANG` | `C.UTF-8` | 공통 | `common-system-config` |
| `LC_ALL` | `C.UTF-8` | 공통 | `common-system-config` |
| `JAVA_TOOL_OPTIONS` | UTF-8 JVM 옵션 | 공개 설정 | Spring 전용 ConfigMap |
| `MYSQL_HOST` | `database-mysql-server` | 공개 설정 | Spring 전용 ConfigMap |
| `MYSQL_PORT` | `3306` | 공개 설정 | MySQL 또는 Spring 전용 ConfigMap |
| `VISION_MODEL_PATH` | `/models/huggingface/yolox/yolox_nano.onnx` | 경로 설정 | 모델 PVC 또는 ConfigMap |
| `VISION_HELMET_MODEL_PATH` | `/models/huggingface/yolox/helmet_nano.onnx` | 경로 설정 | 모델 PVC 또는 ConfigMap |
| `STT_MODEL_PATH` | `/models/huggingface/whisper-small-ko` | 경로 설정 | 모델 PVC 또는 ConfigMap |
| `STT_MODEL_NAME` | `whisper-small-ko` | 공개 설정 | 모델 전용 ConfigMap |
| `APP_OCR_BASE_URL` | `http://python-ai-model:60100` | 공개 설정 | Spring 전용 ConfigMap |
| `APP_AGENT_BASE_URL` | `http://python-ai-model:60100` | 공개 설정 | Spring 전용 ConfigMap |
| `APP_AGENT_CLIENT` | `fastapi` | 공개 설정 | 모델 전용 ConfigMap |
| `LLM_ADAPTERS` | Compose 기본값 또는 외부 지정 | 공개 설정 | 환경별 ConfigMap |
| `MYSQL_DATABASE` | 외부 `.env` 값 | 공개 설정 | MySQL 또는 KB ConfigMap |
| `MYSQL_PASSWORD` | 외부 `.env` 값 | 비밀 | MySQL Secret |
| `INTERNAL_API_TOKEN` | 외부 `.env` 값 | 비밀 | Spring 또는 공통 API Secret |
| `JWT_SECRET` | 외부 `.env` 값 | 비밀 | Spring Secret |
| `ADMIN_EMAIL` | 외부 `.env` 값 | 비밀 | Spring Secret |
| `ADMIN_PASSWORD` | 외부 `.env` 값 | 비밀 | Spring Secret |

Spring은 쿠버네티스에서도 다음과 같이 필요한 리소스만 주입하는 것이 적합하다.

```yaml
envFrom:
  - configMapRef:
      name: common-system-config
  - configMapRef:
      name: common-spring-config
  - secretRef:
      name: common-spring-secret
  - secretRef:
      name: common-mysql-secret
```

## 4. Python AI 모델

| 환경 변수 | Compose 값 | 구분 | Kubernetes 이관 |
|---|---|---|---|
| `RAG_BACKEND` | `app.rag.tool_backend:build_search_backend` | 공개 설정 | AI 전용 ConfigMap |
| `RAG_SEARCH_TIMEOUT_SECONDS` | `60` | 공개 설정 | RAG/AI 또는 Ingest 공통 ConfigMap |
| `EMBEDDING_MODEL` | `BAAI/bge-m3` | 공개 설정 | 임베딩 공통 ConfigMap |
| `EMBEDDING_DIM` | `1024` | 공개 설정 | 임베딩 공통 ConfigMap |
| `STT_MODEL_NAME` | `whisper-small-ko` | 공개 설정 | 모델 공통 ConfigMap |
| `LLM_ADAPTERS` | 어댑터 두 개 경로가 포함된 JSON | 공개 설정 | AI 전용 ConfigMap |
| `NETWORK_ENV` | `OPEN` | 공개 설정 | AI 전용 ConfigMap |
| `CHECK_RULES_TOOL` | 기본값 `MOCK` | 공개 설정 | AI 전용 ConfigMap |
| `MSDS_TOOL` | 기본값 `KOSHA` | 공개 설정 | AI 전용 ConfigMap |
| `EMBEDDING_URL` | `http://python-embedding-model:60110/v1/embeddings` | 공개 설정 | 임베딩 ConfigMap 또는 Service 주소 key |
| `INTERNAL_API_BASE_URL` | `http://spring-boot-application:60001` | 공개 설정 | AI 전용 ConfigMap |
| `QDRANT_URL` | `http://qdrant-server:6333` | 공개 설정 | Qdrant ConfigMap |
| `HF_HOME` | `/models/huggingface` | 경로 설정 | Volume mount 또는 환경 설정 |
| `VISION_MODEL_PATH` | `/models/yolox/yolox_nano.onnx` | 경로 설정 | Volume mount 또는 ConfigMap |
| `VISION_HELMET_MODEL_PATH` | `/models/yolox/helmet_nano.onnx` | 경로 설정 | Volume mount 또는 ConfigMap |
| `STT_MODEL_PATH` | `/models/huggingface/whisper-small-ko` | 경로 설정 | Volume mount 또는 ConfigMap |
| `INTERNAL_API_TOKEN` | 외부 `.env` 값 | 비밀 | Spring 또는 공통 API Secret |
| `EMBEDDING_API_KEY` | 외부 `.env` 값 | 비밀 | Embedding Secret |
| `QDRANT_READ_ONLY_API_KEY` | 외부 `.env` 값 | 비밀 | Qdrant Secret |
| `KOSHA_MSDS_SERVICE_KEY` | 외부 `.env` 값 | 비밀 | KOSHA Secret |

`LLM_ADAPTERS`는 JSON 문자열을 직접 ConfigMap에 넣기보다 어댑터 마운트 경로와 관련된 도메인 설정으로 분리하는 것을 권장한다.

## 5. 임베딩 모델

| 환경 변수 | Compose 값 | 구분 | Kubernetes 이관 |
|---|---|---|---|
| `HF_HOME` | `/models/huggingface` | 경로 설정 | 임베딩 Volume 또는 ConfigMap |
| `EMBEDDING_MODEL` | `BAAI/bge-m3` | 공개 설정 | 임베딩 ConfigMap |
| `EMBEDDING_DIM` | `1024` | 공개 설정 | 임베딩 ConfigMap |
| `EMBEDDING_MAX_TOKENS` | `8192` | 공개 설정 | 임베딩 ConfigMap |
| `EMBEDDING_MAX_BATCH_SIZE` | `8` | 공개 설정 | 임베딩 ConfigMap |
| `EMBEDDING_LOCAL_FILES_ONLY` | `false` | 공개 설정 | 임베딩 ConfigMap |
| `TZ` | `Asia/Seoul` | 공통 | `common-system-config` |
| `EMBEDDING_API_KEY` | 외부 `.env` 값 | 비밀 | Embedding Secret |

임베딩을 소비하는 서비스가 일부 값만 사용할 경우 `envFrom` 전체 주입보다
필요한 key를 명시하는 방식을 고려할 수 있다.

```yaml
env:
  - name: EMBEDDING_MODEL
    valueFrom:
      configMapKeyRef:
        name: common-embedding-config
        key: EMBEDDING_MODEL
```

변수를 많이 나열해야 한다면 도메인 ConfigMap을 유지하고 `envFrom`을 사용한다.
이때도 다른 도메인의 설정은 포함하지 않는다.

## 6. MySQL

| 환경 변수 | Compose 값 | 구분 | Kubernetes 이관 |
|---|---|---|---|
| `TZ` | `Asia/Seoul` | 공통 | `common-system-config` |
| `LANG` | `C.UTF-8` | 공통 | `common-system-config` |
| `LC_ALL` | `C.UTF-8` | 공통 | `common-system-config` |
| `MYSQL_DATABASE` | 외부 `.env` 값 | 공개 설정 | MySQL ConfigMap |
| `MYSQL_USER` | 외부 `.env` 값 | 공개 설정 | MySQL ConfigMap |
| `MYSQL_PASSWORD` | 외부 `.env` 값 | 비밀 | MySQL Secret |
| `MYSQL_ROOT_PASSWORD` | 외부 `.env` 값 | 비밀 | MySQL Secret |
| `KB_OWNER_PASSWORD` | 외부 `.env` 값 | 비밀 | KB Secret |
| `KB_INGEST_PASSWORD` | 외부 `.env` 값 | 비밀 | KB Secret |
| `KB_READER_PASSWORD` | 외부 `.env` 값 | 비밀 | KB Secret |

MySQL 초기화 스크립트와 StatefulSet이 공유하는 Secret은 도메인별로 분리한다.

## 7. Flyway / KB Migrator

| 환경 변수 | Compose 값 | 구분 | Kubernetes 이관 |
|---|---|---|---|
| `KB_OWNER_USER` | `kb_owner` | 공개 설정 | KB ConfigMap |
| `EMBEDDING_URL` | `http://python-embedding-model:60110/v1/embeddings` | 공개 설정 | Embedding ConfigMap |
| `EMBEDDING_MODEL` | `BAAI/bge-m3` | 공개 설정 | Embedding ConfigMap |
| `EMBEDDING_DIM` | `1024` | 공개 설정 | Embedding ConfigMap |
| `EMBEDDING_MAX_TOKENS` | `8192` | 공개 설정 | Embedding ConfigMap |
| `EMBEDDING_MAX_BATCH_SIZE` | `8` | 공개 설정 | Embedding ConfigMap |
| `FLYWAY_PASSWORD` | `KB_OWNER_PASSWORD`와 동일 | 비밀 | Flyway 또는 KB Secret |
| `EMBEDDING_API_KEY` | 외부 `.env` 값 | 비밀 | Embedding Secret |

## 8. Qdrant

| 환경 변수 | Compose 값 | 구분 | Kubernetes 이관 |
|---|---|---|---|
| `QDRANT__SERVICE__API_KEY` | 외부 `.env` 값 | 비밀 | Qdrant Secret |
| `QDRANT__SERVICE__READ_ONLY_API_KEY` | 외부 `.env` 값 | 비밀 | Qdrant Secret |
| `DEBUG` | `True` | 공개 설정 | Qdrant ConfigMap |

Compose의 읽기 전용 키가 두 형태로 사용될 수 있으므로
Kubernetes에서는 Canonical key인 `QDRANT_READ_ONLY_API_KEY`를 유지하고,
Qdrant 서버가 요구하는 이중 언더스코어 이름은 Pod 환경에서 매핑한다.

## 9. Qdrant Init

| 환경 변수 | Compose 값 | 구분 | Kubernetes 이관 |
|---|---|---|---|
| `QDRANT_URL` | `http://qdrant-server:6333` | 공개 설정 | Qdrant ConfigMap |
| `QDRANT_API_KEY` | 외부 `.env` 값 | 비밀 | Qdrant Secret |
| `EMBEDDING_DIM` | `1024` | 공개 설정 | Embedding ConfigMap |

## 10. KB Ingest

| 환경 변수 | Compose 값 | 구분 | Kubernetes 이관 |
|---|---|---|---|
| `QDRANT_URL` | `http://qdrant-server:6333` | 공개 설정 | Qdrant ConfigMap |
| `EMBEDDING_URL` | `http://python-embedding-model:60110/v1/embeddings` | 공개 설정 | Embedding ConfigMap |
| `EMBEDDING_MODEL` | `BAAI/bge-m3` | 공개 설정 | Embedding ConfigMap |
| `EMBEDDING_DIM` | `1024` | 공개 설정 | Embedding ConfigMap |
| `RAG_SEARCH_TIMEOUT_SECONDS` | `60` | 공개 설정 | RAG/AI 공통 ConfigMap |
| `KB_DB_HOST` | `database-mysql-server` | 공개 설정 | KB ConfigMap |
| `KB_DATABASE` | `agent_kb` | 공개 설정 | KB ConfigMap |
| `KB_DB_PORT` | `3306` | 공개 설정 | MySQL 또는 KB ConfigMap |
| `LOCAL_CHUNK_ROOT` | `/data/out` | 경로 설정 | Volume mount 또는 ConfigMap |
| `LOCAL_VECTOR_DIR` | `/data/vectors` | 경로 설정 | Volume mount 또는 ConfigMap |
| `QDRANT_API_KEY` | 외부 `.env` 값 | 비밀 | Qdrant Secret |
| `EMBEDDING_API_KEY` | 외부 `.env` 값 | 비밀 | Embedding Secret |
| `KB_INGEST_PASSWORD` | 외부 `.env` 값 | 비밀 | KB Secret |
| `KB_INGEST_API_TOKEN` | 외부 `.env` 값 | 비밀 | KB Secret |

## 권장 리소스 구성

| 리소스 | 포함할 설정 |
|---|---|
| `common-system-config` | `TZ`, `LANG`, `LC_ALL` |
| `common-mysql-config` | MySQL 주소, 포트, 데이터베이스명, 사용자명, 문자셋 |
| `common-kb-config` | KB 데이터베이스명, 소유자 계정 |
| `common-embedding-config` | 임베딩 모델명, 차원, 토큰 제한, 배치 크기, 임베딩 URL |
| `common-model-config` | 모델 클라이언트 종류와 공통 모델 설정 |
| `common-spring-config` | Java 및 Spring 전용 공개 설정 |
| `common-python-config` | Python AI 전용 공개 설정 |
| `common-spring-secret` | JWT 및 관리자 계정, 내부 API 토큰 |
| `common-mysql-secret` | MySQL 사용자 및 root 비밀번호 |
| `common-kb-secret` | KB 계정 비밀번호와 ingest 토큰 |
| `common-embedding-secret` | 임베딩 API 키 |
| `common-qdrant-secret` | Qdrant 관리 및 읽기 전용 API 키 |
| `common-kosha-secret` | KOSHA 서비스 키 |

## 주입 규칙

- 시스템 공통값만 `common-system-config`에 둔다.
- 도메인별로 독립적으로 소비하는 설정은 임베딩, MySQL, KB, Qdrant, Spring 등의 전용 리소스로 분리한다.
- 여러 키를 매번 나열해야 하면 도메인 ConfigMap을 `envFrom`으로 주입한다.
- 키의 수가 적고 소비 서비스가 제한적이면 `configMapKeyRef` 또는 `secretKeyRef`를 사용한다.
- Secret은 ConfigMap과 분리하고 Secret 리소스를 직접 Git에 저장하지 않는다.
- Kubernetes 내부 주소는 Compose 컨테이너 이름이 아니라 Service 이름과 Service port를 사용한다.
