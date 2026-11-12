# Docker Compose 구성 안내

이 문서는 2026-11-12 기준 [docker-compose.yml](./docker-compose.yml)의 활성 설정을 설명한다.
현재 선언된 동작과 개선이 필요한 사항을 구분하며, 실제 서비스 기동을 검증한 문서는 아니다.
환경 변수의 실제 값과 비밀 정보는 포함하지 않는다.

## 1. 전체 구성

활성 서비스는 9개이며, 모두 Compose의 `default` bridge 네트워크에 연결된다.
Jenkins 설정은 주석 처리되어 있어 실행되지 않는다.

| 서비스 | 역할 | 이미지 또는 빌드 | 재시작 정책 |
|---|---|---|---|
| `vite-web-application` | 웹 프런트엔드와 `/api` 프록시 | `../../main-vite-web/Dockerfile` | `always` |
| `spring-boot-application` | 업무 API, MySQL 접근, AI API 호출 | `../../main-server/Dockerfile` | `always` |
| `python-ai-model` | AI API, RAG 검색 | `../../main-ai-model/Dockerfile`, `ai` target | `always` |
| `python-embedding-model` | 임베딩 API | 같은 Dockerfile, `embedding` target | `always` |
| `database-mysql-server` | 업무 DB와 `agent_kb` 저장 | `mysql:8-oraclelinux9` | `always` |
| `kb-migrator` | Flyway로 `agent_kb` 스키마 마이그레이션 | `flyway/flyway:13.12-alpine` | `no` |
| `qdrant-server` | 검색용 벡터 DB | `qdrant/qdrant:v1.20` | `always` |
| `qdrant-init` | Qdrant 컬렉션과 payload 인덱스 초기화 | AI Dockerfile, `qdrant-init` target | `no` |
| `kb-ingest` | 지식 데이터 적재 API | AI Dockerfile, `ingest` target | `always` |

장기 실행 서비스 7개와 일회성 작업 2개로 구성된다.
`restart: "no"`는 자동 재시작을 하지 않는다는 뜻이며, 수동 실행이나 재생성을 금지하는 설정은 아니다.

### 런타임 통신

아래 화살표는 API 및 DB 접근 방향이며, 시작 순서가 아니다.

```text
웹 → Spring → MySQL
          ↔ AI API → 임베딩 API
                  → Qdrant (읽기 전용 키)

적재 API → 임베딩 API
         → MySQL agent_kb
         → Qdrant (쓰기 키)

Flyway → MySQL agent_kb
Qdrant 초기화 → Qdrant
```

AI API에는 Spring을 호출하는 `INTERNAL_API_BASE_URL`도 설정되어 있다.
임베딩 모델은 `BAAI/bge-m3`, 벡터 차원은 `1024`로 설정되어 있으며,
AI API, 임베딩 API, 적재 API, Qdrant 초기화 작업 사이에서 일치해야 한다.

## 2. 네트워크와 포트

컨테이너 간 통신에는 서비스 이름과 **컨테이너 포트**를 사용한다.
예를 들어 Spring의 MySQL 접속 주소는 `database-mysql-server:3306`이며,
호스트에서 접근할 때의 `60010`을 사용하지 않는다.
컨테이너 내부의 `localhost`는 해당 컨테이너 자신이다.

| 대상 | 컨테이너 간 주소 | 호스트 공개 포트 |
|---|---|---|
| 웹 | `vite-web-application:50001` | `50001:50001` |
| Spring | `spring-boot-application:60001` | 없음 |
| AI API | `python-ai-model:60100` | `60100:60100` |
| 임베딩 API | `python-embedding-model:60110` | 없음, `ports`가 주석 처리됨 |
| MySQL | `database-mysql-server:3306` | `60010:3306` |
| Qdrant HTTP API | `qdrant-server:6333` | `60011:6333` |
| 적재 API | `kb-ingest:8001` | `8001:8001` |

웹의 `VITE_API_PROXY_TARGET`은 `http://spring-boot-application:60001`이다.
임베딩 호출 경로는 `/v1/embeddings`이다.
Qdrant gRPC 포트는 현재 Compose에 공개되어 있지 않다.

공개 포트에 호스트 IP를 명시하지 않았으므로 localhost 전용 노출이라고 가정하면 안 된다.
실제 접근 범위는 Docker와 호스트 방화벽 설정을 함께 확인해야 한다.

## 3. 데이터와 파일 마운트

상대 경로는 [docker-compose.yml](./docker-compose.yml)이 위치한 디렉터리를 기준으로 해석한다.

### Named volume

| 볼륨 | 서비스 | 컨테이너 경로 | 용도 |
|---|---|---|---|
| `database-data` | MySQL | `/var/lib/mysql` | MySQL 데이터 |
| `qdrant-data` | Qdrant | `/qdrant/storage` | 벡터 및 컬렉션 데이터 |
| `ai-model-cache` | AI API | `/models/huggingface` | Hugging Face 캐시 |
| `embedding-model-cache` | 임베딩 API | `/models/huggingface` | Hugging Face 캐시 |
| `default` | 현재 사용 없음 | 없음 | 선언만 존재 |

모든 named volume은 `local` 드라이버를 사용한다.
실제 Docker 볼륨 이름에는 일반적으로 Compose 프로젝트 이름이 붙는다.
MySQL과 Qdrant는 서로 다른 볼륨을 사용하며, 같은 데이터 볼륨을 공유하면 안 된다.

### Bind mount

| 호스트 경로 | 컨테이너 경로 | 서비스 | 모드 |
|---|---|---|---|
| `../db/init` | `/docker-entrypoint-initdb.d` | MySQL | 읽기 전용 |
| `../db/verify` | `/opt/verify` | MySQL | 읽기 전용 |
| `../../main-ai-model/db/migration` | `/flyway/sql` | Flyway | 읽기·쓰기 |
| `../../main-ai-model/out` | `/data/out` | 적재 API | 읽기 전용 |
| `../../main-ai-model/out/vectors` | `/data/vectors` | 적재 API | 읽기·쓰기 |

`/opt/verify` 마운트만으로 검증 스크립트가 자동 실행되지는 않는다.
적재 API는 `LOCAL_CHUNK_ROOT=/data/out`, `LOCAL_VECTOR_DIR=/data/vectors`를 사용한다.
`out/vectors`는 별도 쓰기 가능한 마운트이므로 실제 출력 데이터도 백업 대상으로 고려한다.

## 4. 초기화와 마이그레이션의 차이

### MySQL 최초 초기화

MySQL 이미지가 빈 데이터 디렉터리를 초기화할 때 `MYSQL_DATABASE`, `MYSQL_USER` 등의 설정과
`/docker-entrypoint-initdb.d`의 스크립트를 적용한다.
기존 데이터가 있는 볼륨에는 최초 초기화 스크립트가 자동으로 다시 실행되지 않는다.

[10-agent-kb.sh](../db/init/10-agent-kb.sh)는 기본적으로 `agent_kb` DB와 다음 계정을 준비한다.

| 계정 | 역할 |
|---|---|
| `kb_owner` | `agent_kb` DDL 및 데이터 변경, 마이그레이션 전용 |
| `kb_ingest` | `agent_kb` 조회·삽입·수정·삭제 |
| `kb_reader` | `agent_kb` 조회 전용 |

이 스크립트는 기존 계정의 비밀번호와 권한도 다시 설정한다.
수동 재실행은 단순 확인 작업이 아니라 계정 설정 변경이라는 점에 주의한다.
MySQL 실행 옵션에는 `--log-bin-trust-function-creators=1`과 `--default-time-zone=+09:00`이 포함된다.

### Flyway 마이그레이션

`kb-migrator`는 다음 대상으로 `migrate` 명령을 실행한다.

- JDBC URL: `jdbc:mysql://database-mysql-server:3306/agent_kb?allowPublicKeyRetrieval=true`
- 사용자: `kb_owner`
- 비밀번호: `KB_OWNER_PASSWORD`를 `FLYWAY_PASSWORD`로 전달
- SQL 디렉터리: `/flyway/sql`
- `-validateMigrationNaming=true` 적용

최초 DB·계정 생성과 스키마 마이그레이션은 별도 단계이다.
Flyway는 이미 적용된 마이그레이션 이력을 관리하지만, 실행 전에 MySQL과 계정이 준비되어 있어야 한다.
Compose에 선언된 임베딩 관련 환경 변수는 Flyway 명령 자체가 임베딩 API를 호출한다는 의미는 아니다.

### Qdrant 초기화

`qdrant-init` 이미지는 `python -m scripts.init_qdrant`를 실행한다.
[초기화 스크립트](../../main-ai-model/scripts/init_qdrant.py)는 다음을 수행한다.

- `kb_accidents`, `kb_guides` 컬렉션이 없으면 생성한다.
- `dense` 벡터는 1024차원 cosine, `sparse` 벡터는 IDF 설정을 사용한다.
- 컬렉션별 payload 인덱스를 생성한다.
- 기존 컬렉션은 유지하지만, 기존 벡터 차원을 자동 변경하거나 호환성을 검증하지는 않는다.

현재 Compose 명령에는 `--recreate`와 `--check`가 없다.
`--recreate`는 기존 컬렉션을 삭제하므로 데이터 보존 목적의 일반 초기화에 사용하면 안 된다.
초기화 작업은 쓰기 가능한 `QDRANT_API_KEY`를 사용한다.

## 5. 현재 시작 의존성과 주의사항

`service_healthy`는 healthcheck 성공을, `service_completed_successfully`는 작업의 정상 종료를 기다린다.
이는 시작 시의 조건이지, 실행 중 의존 서비스 장애를 계속 감시하는 보장은 아니다.

| 서비스 | 현재 `depends_on` 조건 |
|---|---|
| 웹 | 없음 |
| Spring | MySQL healthy |
| AI API | Qdrant 초기화 성공, Qdrant healthy, 임베딩 API healthy |
| 임베딩 API | 없음 |
| MySQL | **Flyway 성공** |
| Flyway | 없음 |
| Qdrant | **Qdrant 초기화 성공** |
| Qdrant 초기화 | 없음 |
| 적재 API | Qdrant 초기화 성공, Qdrant healthy, MySQL healthy |

### DB와 선행 작업의 순서가 반대인 부분

현재 MySQL은 Flyway의 완료를 기다리지만 Flyway는 MySQL에 접속해야 한다.
Qdrant도 초기화 작업의 완료를 기다리지만 초기화 작업은 Qdrant에 접속해야 한다.
YAML 의존성 그래프 자체의 순환이라기보다는, 실제 통신에 필요한 시작 순서가 반대로 선언된 상태이다.
따라서 DB가 실행 중이지 않은 새 환경에서는 정상 기동을 기대할 수 없다.

권장 순서는 다음과 같으며, **현재 Compose에는 아직 반영되어 있지 않다.**

```text
MySQL 시작 → 최초 초기화 → MySQL 준비 확인 → Flyway 성공
Qdrant 시작 → Qdrant 준비 확인 → Qdrant 초기화 성공
임베딩 API 시작 → 준비 확인
위 단계 완료 → 해당 기능을 사용하는 API 및 적재 서비스 시작
```

적재 API에는 Flyway 성공과 임베딩 API healthy를 기다리는 조건이 직접 선언되어 있지 않다.
웹은 Spring의 준비를 기다리지 않으며, Spring은 AI API의 준비를 기다리지 않는다.
앱 내부의 재시도 여부는 이 Compose 파일만으로 판단할 수 없다.

### Healthcheck

| 서비스 | 검사 | 간격 / timeout / retries |
|---|---|---|
| Spring | `curl -f http://localhost:60001/actuator/health` | 30초 / 10초 / 3회 |
| AI API | `curl -f http://localhost:60100/actuator/health` | 30초 / 10초 / 3회 |
| 임베딩 API | `curl -f http://localhost:60110/ready` | 30초 / 10초 / 3회 |
| MySQL | root 계정으로 `SELECT 1` | 30초 / 10초 / 3회 |
| Qdrant | bash `/dev/tcp/127.0.0.1/6333` 연결 | 5초 / 3초 / 12회, 시작 유예 5초 |
| 적재 API | `curl -f http://localhost:8001/ready` | 30초 / 10초 / 3회 |

- Qdrant 검사는 TCP 연결만 확인한다. 인증, 컬렉션 초기화, 실제 검색 성공을 검증하지 않는다.
- MySQL 검사에 사용된 `$MYSQL_ROOT_PASSWORD`는 Compose가 호스트 환경이나 env 파일에서 보간한다.
  컨테이너 내부 변수 확장이 목적이라면 `$$` 이스케이프 여부를 검토해야 한다.
- [AI Dockerfile](../../main-ai-model/Dockerfile)은 `curl` 설치를 명시하지 않는다.
  Python 서비스의 healthcheck 명령을 이미지에서 실제 실행할 수 있는지 확인해야 한다.
- 모델 다운로드와 초기 로딩이 길어질 수 있지만 임베딩 API에 별도 `start_period`는 없다.

## 6. 환경 변수

`.env`의 실제 값은 소스나 문서에 복사하지 않는다.
Compose 보간의 필수 여부와 애플리케이션 실행 시 필수 여부는 서로 다르다.

### Compose가 비어 있으면 오류로 처리하는 변수

`${VAR:?메시지}`로 선언된 변수는 다음과 같다.

| 변수 | 사용처 |
|---|---|
| `INTERNAL_API_TOKEN` | Spring과 AI API의 내부 API 인증 |
| `QDRANT_API_KEY` | Qdrant 쓰기 키와 초기화 작업, 적재 API |
| `QDRANT_READ_ONLY_API_KEY` | Qdrant 읽기 전용 키와 AI API |
| `KB_OWNER_PASSWORD` | MySQL 초기화와 Flyway |
| `KB_INGEST_API_TOKEN` | 적재 API 인증 |

일부 서비스에서는 같은 변수를 단순 `${VAR}`로 참조하지만,
전체 파일의 다른 서비스에 필수 검사가 존재한다.

### 기본값 없이 참조하는 나머지 변수

- DB: `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`, `MYSQL_ROOT_PASSWORD`,
  `KB_INGEST_PASSWORD`, `KB_READER_PASSWORD`
- 인증: `JWT_SECRET`, `EMBEDDING_API_KEY`

이 변수들은 미설정 시 Compose가 경고하거나 빈 값으로 처리할 수 있다.
그렇다고 정상 실행이 보장되지는 않는다. 특히 MySQL 초기 스크립트는 KB 계정 비밀번호를 별도로 검사한다.
`EMBEDDING_API_KEY`는 임베딩 서비스에서는 기본값 없는 참조,
AI API·Flyway·적재 API에서는 빈 문자열 기본값으로 선언되어 있다.

### 기본값이 있는 변수

| 변수 | 기본값 |
|---|---|
| `ADMIN_EMAIL`, `ADMIN_PASSWORD` | 빈 문자열 |
| `APP_AGENT_CLIENT` | `fastapi` |
| `CHECK_RULES_TOOL` | `MOCK` |
| `KOSHA_MSDS_SERVICE_KEY` | 빈 문자열 |

Qdrant 서버의 환경 변수 이름은 `QDRANT__SERVICE__API_KEY`와
`QDRANT__SERVICE__READ_ONLY_API_KEY`이며, 클라이언트의 변수 이름과 구분한다.
`HF_HOME=/models/huggingface`, `RAG_SEARCH_TIMEOUT_SECONDS=60`,
`EMBEDDING_MAX_TOKENS=8192`, `EMBEDDING_MAX_BATCH_SIZE=8`,
`EMBEDDING_LOCAL_FILES_ONLY="false"` 등은 Compose에 고정되어 있다.
AI API에는 `NETWORK_ENV=OPEN`, Qdrant에는 `DEBUG: True`도 선언되어 있다.

## 7. 운영 시 확인 명령

다음 명령은 `main-infra-manage` 저장소 루트에서 실행한다.
`.env`에 필요한 값을 별도로 준비해야 한다.

```bash
# 보간 및 Compose 구성 검사: 비밀 값이 포함된 전체 설정은 출력하지 않는다.
docker compose --env-file docker/.env -f docker/docker-compose.yml config --quiet

# 현재 컨테이너와 일회성 작업 상태 확인
docker compose --env-file docker/.env -f docker/docker-compose.yml ps -a

# DB, 초기화, 마이그레이션 로그 확인
docker compose --env-file docker/.env -f docker/docker-compose.yml logs --tail=100 database-mysql-server kb-migrator qdrant-server qdrant-init
```

`config --quiet` 성공은 서비스 접속, 이미지 내 healthcheck 도구, 초기화 성공을 검증하지 않는다.
시작 순서 문제가 있으므로 전체 `up`을 정상 기동 절차로 안내하지 않는다.
로그에도 민감한 정보가 있을 수 있으므로 외부 공유 전에 확인한다.

데이터 보존 시 `docker compose down --volumes`를 사용하지 않는다.
DB별 백업·복구를 검증하고, 볼륨 이름이나 프로젝트 이름 변경이 기존 데이터의 자동 이전을 뜻한다고 가정하지 않는다.

## 8. Kubernetes 전환 시 대응

아래 내용은 현재 Compose의 설명이 아니라 전환 시 권장 설계이다.

| Compose 구성 | Kubernetes 대응 |
|---|---|
| MySQL, Qdrant | 각각 StatefulSet, 독립 PVC, Service |
| MySQL 최초 초기 스크립트 | MySQL 컨테이너에 마운트하거나 초기화 파일을 포함한 이미지 사용 |
| Flyway | 별도 `batch/v1` Job |
| Qdrant 초기화 | 별도 `batch/v1` Job |
| 웹, Spring, AI API, 임베딩 API, 적재 API | 우선 Deployment와 Service 검토 |
| 비밀 환경 변수 | Secret 또는 기존 비밀 관리 시스템 |
| 비밀이 아닌 설정 | ConfigMap 또는 워크로드 환경 변수 |
| host bind mount | 파일을 포함한 이미지, ConfigMap, PVC 등 용도에 맞게 대체 |
| `depends_on` | 배포 파이프라인의 완료 대기 및 제한된 재시도 |
| healthcheck | 목적에 맞는 startup/readiness/liveness probe |

MySQL 초기화와 Flyway를 동일한 작업으로 합치지 않고,
Qdrant 초기화도 독립 Job으로 두면 실패 원인과 재실행 대상을 구분하기 쉽다.
파일 하나에 여러 리소스를 `---`로 구분할 수 있지만, 각 리소스는 자신의 `apiVersion`, `kind`, `spec`을 가진다.
StatefulSet은 `apps/v1`, Job은 `batch/v1`을 사용한다.
Job 완료를 다음 배포의 자동 재실행으로 오해하지 않도록 배포별 생성·재실행 정책을 정한다.

DB만 먼저 이전할 경우 다음을 함께 계획해야 한다.

1. Compose 앱에서 Kubernetes DB로 접속할 안전한 사설 네트워크 경로.
   일반적인 `ClusterIP`를 Compose에서 바로 사용할 수 있다고 가정하지 않는다.
2. MySQL 백업·복구와 Qdrant snapshot 복구, 데이터 검증.
   현재 Qdrant는 `/qdrant/storage`만 영속 마운트하므로 snapshot 생성·보관 경로도 별도로 확인한다.
3. 수집·쓰기 중단 또는 정합성 검증·재처리 전략.
   두 DB를 서로 다른 시점에 백업하면 참조와 벡터 데이터가 어긋날 수 있다.
4. 초기에는 각각 replica 1로 시작하고, 실제 복제 및 장애 복구 설계 없이 replica만 늘리지 않는다.
5. 새 PVC는 기존 Docker 볼륨 데이터를 자동으로 가져오지 않는다.
