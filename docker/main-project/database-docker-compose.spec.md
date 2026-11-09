# MySQL · Qdrant Docker Compose 명세서

## 목적 및 범위

이 문서는 `main-infra-manage/docker/docker-compose.yml`에서 운영하는 MySQL과 Qdrant의 역할, 네트워크 주소, 영속 저장소, 인증, 초기화 및 검증 절차를 설명합니다. Compose 파일의 실제 서비스 이름과 환경 변수 구성을 기준으로 합니다.

## 구성 개요

Compose는 업무 데이터와 지식 데이터에 동일한 MySQL 서버를 사용하되 데이터베이스와 계정을 분리합니다. Qdrant는 별도 서비스와 영속 볼륨을 사용합니다.

| 서비스 | 이미지/실행 방식 | 컨테이너 주소 | 호스트 포트 | 데이터 볼륨 |
| --- | --- | --- | --- | --- |
| `database-mysql-server` | `mysql:8-oraclelinux9` | `database-mysql-server:3306` | `60010` | `database-data` |
| `qdrant-server` | `qdrant/qdrant:v1.20` | `qdrant-server:6333` | `60011` | `qdrant-data` |
| `kb-migrator` | Flyway 13.12 | MySQL `agent_kb` | 없음 | `db/migration` 읽기 마운트 |
| `qdrant-init` | `main-ai-model` Dockerfile의 `qdrant-init` 타깃 | Qdrant `qdrant-server:6333` | 없음 | 없음 |

Compose 내부 서비스는 위의 service DNS 이름과 컨테이너 포트를 사용합니다. 호스트에서 접속할 때는 `localhost:60010` 또는 `localhost:60011`을 사용합니다.

## MySQL

### 데이터베이스 및 초기화

`database-mysql-server`는 `MYSQL_DATABASE`로 지정한 업무 데이터베이스를 만듭니다. 같은 서버의 `agent_kb`는 `../db/init/10-agent-kb.sh`가 준비합니다.

- `../db/init`은 `/docker-entrypoint-initdb.d`에 읽기 전용으로 마운트됩니다.
- `10-agent-kb.sh`는 MySQL 데이터 디렉터리가 비어 있는 최초 기동에서 자동 실행됩니다.
- `database-data` 볼륨에 이미 데이터가 있으면 초기화 스크립트가 다시 실행되지 않습니다.
- `kb-migrator`는 `../../main-ai-model/db/migration`을 사용해 Flyway로 `agent_kb` 스키마를 적용합니다.
- Compose에서 `kb-migrator`는 MySQL readiness dependency로 연결되어 있지 않습니다. MySQL이 준비된 후 마이그레이션을 실행해야 합니다.

### 계정 권한

`10-agent-kb.sh`는 `agent_kb` 데이터베이스와 다음 최소 권한 계정을 준비합니다.

| 계정 | 권한 | 용도 |
| --- | --- | --- |
| `kb_owner` | DDL 및 스키마 변경 | `kb-migrator` 전용 |
| `kb_ingest` | SELECT, INSERT, UPDATE, DELETE | `kb-ingest` 장부·적재 작업 |
| `kb_reader` | SELECT | AI 검색에서 장부 읽기가 필요할 때 사용 |

업무 데이터베이스 계정과 `kb_*` 계정은 분리합니다. `kb-ingest` 및 `kb_reader`는 업무 DB 접근 권한을 갖지 않습니다. Spring은 업무 DB만 사용하며 `agent_kb`에 직접 연결하지 않습니다.

### 영속화 및 재초기화

MySQL 데이터는 `database-data` named volume에 저장되므로 컨테이너 재생성 후에도 유지됩니다. 기존 볼륨에서 agent_kb 계정/DB를 다시 맞춰야 하는 경우 MySQL이 준비된 후 다음처럼 초기화 스크립트를 수동 실행할 수 있습니다.

```bash
docker compose --project-directory main-infra-manage/docker \
  -f main-infra-manage/docker/docker-compose.yml \
  exec database-mysql-server bash /docker-entrypoint-initdb.d/10-agent-kb.sh
```

이 스크립트는 여러 번 실행해도 데이터베이스와 계정을 맞추도록 작성되어 있지만, 스키마 migration은 별도로 실행해야 합니다.

## Qdrant

### 인증 및 포트

`qdrant-server`는 HTTP API를 컨테이너 내부 6333 포트에서 제공하고 호스트 60011 포트에 공개합니다.

- 쓰기 키: `QDRANT__SERVICE__API_KEY`에 `QDRANT_API_KEY` 값을 전달합니다.
- 읽기 전용 키: `QDRANT__SERVICE__READ_ONLY_API_KEY`에 `QDRANT_READ_ONLY_API_KEY` 값을 전달합니다.
- `qdrant-init`과 `kb-ingest`는 쓰기 키를 사용합니다.
- AI 검색 서비스는 읽기 전용 키를 사용합니다.
- API key 값은 `.env` 또는 배포 secret으로 제공하고 문서·이미지에 기록하지 않습니다.

### 컬렉션 초기화

`qdrant-init`은 컬렉션과 payload index를 생성하는 일회성 작업입니다. 기본 동작은 컬렉션이 없으면 만들고, 있으면 기존 컬렉션을 유지하면서 index를 준비합니다.

1. `qdrant-server` healthcheck가 TCP 6333 준비를 확인합니다.
2. `qdrant-init`이 healthcheck 성공 후 실행됩니다.
3. `python-ai-model`과 `kb-ingest`는 `qdrant-init`의 성공적인 종료를 기다립니다.

기본 실행은 `scripts.init_qdrant`를 사용합니다. 필터 검증은 다음과 같이 실행할 수 있습니다.

```bash
docker compose --project-directory main-infra-manage/docker \
  -f main-infra-manage/docker/docker-compose.yml \
  run --rm qdrant-init python -m scripts.init_qdrant --check
```

`--recreate`는 기존 컬렉션을 삭제하고 다시 만듭니다. 컬렉션 데이터가 삭제되므로 임베딩 차원을 변경하는 경우에만 사용하고, 실행 전 데이터 보존 필요성을 확인합니다.

### 영속화

Qdrant 저장소는 `qdrant-data` named volume에 유지됩니다. 일반적인 `docker compose down`은 named volume을 보존합니다. 반면 `docker compose down -v`는 MySQL과 Qdrant 데이터를 함께 삭제하므로 테스트 환경에서도 주의해야 합니다.

## 환경 변수

Compose를 적용하기 전에 `.env` 또는 실행 환경에 다음 변수들을 설정해야 합니다. 비밀 값은 저장소에 커밋하지 않습니다.

| 변수 | 사용처 |
| --- | --- |
| `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`, `MYSQL_ROOT_PASSWORD` | 업무 DB와 업무 계정 |
| `KB_OWNER_PASSWORD`, `KB_INGEST_PASSWORD`, `KB_READER_PASSWORD` | `agent_kb` 권한 분리 계정 |
| `QDRANT_API_KEY`, `QDRANT_READ_ONLY_API_KEY` | Qdrant 쓰기·읽기 인증 |
| `KB_INGEST_API_TOKEN` | ingest HTTP API 인증 |

누락 시 Compose의 필수 변수 검사 또는 애플리케이션 연결 단계가 실패합니다. Compose 설정은 저장소 루트에서 다음처럼 확인합니다.

```bash
docker compose --project-directory main-infra-manage/docker \
  -f main-infra-manage/docker/docker-compose.yml config --quiet
```

## 기동 및 확인

저장소 루트에서 전체 서비스 및 필요한 초기화 작업을 실행합니다.

```bash
docker compose --project-directory main-infra-manage/docker \
  -f main-infra-manage/docker/docker-compose.yml up --build -d

docker compose --project-directory main-infra-manage/docker \
  -f main-infra-manage/docker/docker-compose.yml ps
```

MySQL은 다음 조건을 확인합니다.

- `database-mysql-server`가 실행 중이고 로그에 MySQL ready 메시지가 있습니다.
- 최초 초기화에서 업무 DB 및 `agent_kb`가 준비됐습니다.
- `kb-migrator`가 성공적으로 완료되어 `kb_sources`, `kb_chunks` 등의 migration table이 적용됐습니다.
- 애플리케이션 컨테이너에서 MySQL 주소는 `database-mysql-server:3306`입니다.

Qdrant는 다음 조건을 확인합니다.

- `qdrant-server` healthcheck가 healthy입니다.
- `qdrant-init` 로그에서 `kb_accidents`, `kb_guides` 생성 또는 기존 컬렉션 skip과 payload index 구성을 확인합니다.
- 호스트 접속 주소는 `http://localhost:60011`, Compose 내부 주소는 `http://qdrant-server:6333`입니다.
- `qdrant-data` volume이 연결되어 있어 컨테이너 재생성 후에도 point와 collection이 유지됩니다.

로그 조회 예시:

```bash
docker compose --project-directory main-infra-manage/docker \
  -f main-infra-manage/docker/docker-compose.yml logs --tail=100 \
  database-mysql-server qdrant-server qdrant-init kb-migrator
```

## 운영 주의 사항

- MySQL init script는 빈 데이터 volume의 최초 기동에만 자동 실행됩니다. 계정 설정 파일을 수정했다고 기존 volume에 자동 재적용되지는 않습니다.
- Qdrant healthcheck는 포트가 열렸는지 확인하는 수준이며 인증이나 컬렉션 schema 검증은 `qdrant-init`이 담당합니다.
- Compose의 host port mapping은 로컬 개발용입니다. 외부에 서비스를 공개할 때는 방화벽, TLS, secret 보관 및 접근 제어를 별도로 구성해야 합니다.
- MySQL과 Qdrant는 분리된 데이터 저장소입니다. point 색인 성공과 MySQL 장부 상태 사이에는 분산 트랜잭션이 없으므로, 백업·복구 및 재시도 절차를 마련해야 합니다.
