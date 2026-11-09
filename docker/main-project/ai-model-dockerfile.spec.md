# AI Model Dockerfile 및 Compose 명세서

## 목적 및 범위

이 명세서는 `main-ai-model/Dockerfile`의 `ai`, `embedding`, `ingest`, `qdrant-init` 빌드 타깃과 `main-infra-manage/docker/docker-compose.yml`의 서비스·포트·환경 변수 연결을 설명한다.

## 설계 목표

- AI API, 임베딩 API, KB ingest, Qdrant schema 초기화 작업을 각기 적절한 Docker 타깃으로 빌드한다.
- 용량이 큰 ML 의존성은 AI API와 임베딩 타깃에서 공유한다.
- ingest와 Qdrant 초기화 이미지는 각자의 경량 의존성만 포함한다.
- 다운로드한 Python wheel 파일은 최종 런타임 이미지에 포함하지 않는다.
- Hugging Face 모델 파일은 이미지 레이어가 아니라 영속 named volume에 저장한다.
- 장시간 실행 서비스와 일회성 초기화 작업의 실행 명령 및 포트를 명확히 분리한다.

## 스테이지 구성

| 스테이지 | 역할 | 최종 이미지 포함 여부 |
| --- | --- | --- |
| `base` | Python 3.12 실행 환경, 공통 환경 변수, `/app` 작업 디렉터리를 제공한다. | 부모 레이어로 포함 |
| `ml-wheelhouse` | `requirements.release.txt`의 바이너리 wheel을 다운로드한다. BuildKit pip 캐시를 사용한다. | 미포함 |
| `ml-dependencies` | `/opt/venv`에 ML 의존성을 설치하고 `pip check`를 실행한 뒤 wheel을 삭제한다. | `ai`, `embedding`에 포함 |
| `ai` | 애플리케이션을 복사하고 `app.main:app`을 포트 60100에서 실행한다. | 포함 |
| `embedding` | 애플리케이션을 복사하고 factory로 `app.rag.embedding_api:create_app`을 포트 60110에서 실행한다. | 포함 |
| `ingest-dependencies` | `requirements.ingest.txt`에서 FastAPI, Uvicorn, PyMySQL RSA 인증 의존성을 설치한다. | `ingest`에 포함 |
| `ingest` | 애플리케이션을 복사하고 `app.ingest_main:app`을 포트 8001에서 실행한다. | 포함 |
| `qdrant-init` | `requirements.qdrant-init.txt`에서 `qdrant-client`를 설치하고 `scripts/`를 복사한다. 기본 명령은 `python -m scripts.init_qdrant`다. | 포함 |

AI API와 임베딩 서비스는 `requirements.txt`의 FastAPI/Pydantic 계층과 `requirements.release.txt`의 CPU PyTorch 계층을 포함한다. 두 타깃은 같은 `ml-dependencies` 스테이지를 공유한다. `ingest`는 `requirements.ingest.txt`만 사용하며 ML 런타임을 설치하지 않는다. `qdrant-init`은 `qdrant-client`만 별도 설치하고 API 애플리케이션 코드는 복사하지 않는다.

## Requirements 구성

| 파일 | 사용 타깃 | 주요 역할 |
| --- | --- | --- |
| `requirements.txt` | `ai`, `embedding`의 공통 release dependency | FastAPI, Uvicorn, Pydantic, Transformers 등 운영 Python 패키지를 지정한다. |
| `requirements.release.txt` | `ai`, `embedding` | `requirements.txt`를 포함하고 CPU 전용 Torch, Torchvision, Torchaudio wheel을 선택한다. |
| `requirements.ingest.txt` | `ingest` | FastAPI, Pydantic Settings, Uvicorn, `PyMySQL[rsa]`를 설치한다. RSA extra는 MySQL `caching_sha2_password` 인증에 필요하다. |
| `requirements.qdrant-init.txt` | `qdrant-init` | Qdrant API 호출용 `qdrant-client`를 설치한다. |
| `requirements.dev.txt` | 개발·테스트 환경 | 운영 requirements를 포함하고 pytest 및 개발용 ML 패키지를 추가한다. |

`ml-wheelhouse`는 `requirements.release.txt`의 바이너리 wheel을 다운로드한다. BuildKit 캐시 마운트의 pip cache는 이미지 레이어에 포함되지 않는다. `ml-dependencies`는 wheelhouse에서 `--no-index`로 설치하고 `pip check`를 실행한 뒤 복사된 wheel을 삭제한다. 최종 AI 및 embedding 이미지에는 가상환경이 포함되고 다운로드 wheel은 포함되지 않는다.

운영 requirements를 바꾸면 해당 dependency 설치 레이어가 다시 빌드된다. 애플리케이션 소스만 바뀌면 dependency 레이어는 재사용할 수 있다. `ingest`와 `qdrant-init`은 독립 requirements를 사용하므로 ML wheelhouse 변경과 분리되어 있다. pip cache mount를 사용하려면 BuildKit이 필요하다.

## Compose 연결

운영 Compose는 `main-infra-manage/docker/docker-compose.yml`이며, 저장소 루트 기준 서비스와 포트는 다음과 같다.

| 서비스 | Docker 타깃 | 컨테이너 포트 | Host port | 영속 볼륨 / 역할 |
| --- | --- | --- | --- | --- |
| `python-ai-model` | `ai` | 60100 | 60100 | `ai-model-cache`; RAG 검색은 Qdrant read-only key 사용 |
| `python-embedding-model` | `embedding` | 60110 | 60110 | `embedding-model-cache`; bge-m3 모델 파일 캐시 |
| `kb-ingest` | `ingest` | 8001 | 8001 | MySQL agent_kb와 Qdrant에 쓰기 |
| `qdrant-init` | `qdrant-init` | 없음 | 없음 | Qdrant 컬렉션과 payload index를 준비하고 종료 |

Compose 내부 통신은 host port가 아니라 `python-embedding-model:60110`, `qdrant-server:6333`, `database-mysql-server:3306` service 주소를 사용한다. AI와 ingest는 `qdrant-init`이 성공적으로 종료된 뒤 시작한다. `qdrant-init`은 Qdrant healthcheck 성공 뒤 실행된다.

임베딩 서비스의 `EXPOSE`와 CMD 포트는 60110이다. 운영 Compose에서 별도 `command`로 덮어쓰지 않는다. Hugging Face 모델 파일은 named volume에 두어 이미지 크기를 늘리지 않고 컨테이너 재생성 후에도 재사용한다.

`qdrant-init`의 기본 CMD는 컬렉션과 payload index를 없으면 생성하고 있으면 유지한다. `--check`는 테스트 point로 필터를 확인하고 삭제한다. `--recreate`는 기존 컬렉션 데이터를 삭제하므로 재임베딩/차원 변경 시에만 신중히 실행한다.

## 환경 변수 설정

Compose는 비밀 값을 이미지에 넣지 않고 `.env` 또는 실행 환경에서 주입한다. `.env`는 저장소에 커밋하지 않는다.

| 변수 | 사용처 | 비고 |
| --- | --- | --- |
| `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`, `MYSQL_ROOT_PASSWORD` | MySQL 업무 DB | `database-mysql-server` 및 Spring |
| `KB_OWNER_PASSWORD`, `KB_INGEST_PASSWORD`, `KB_READER_PASSWORD` | `agent_kb` | Flyway, ingest, 선택적 reader |
| `QDRANT_API_KEY`, `QDRANT_READ_ONLY_API_KEY` | Qdrant | 초기화/ingest는 쓰기 키, AI 검색은 읽기 키 |
| `KB_INGEST_API_TOKEN` | ingest API | Compose에서 필수 값으로 검사 |
| `EMBEDDING_API_KEY` | 임베딩 API | ingest와 호출 서비스에 전달 |
| `INTERNAL_API_TOKEN`, `JWT_SECRET` | Spring 내부 API 및 인증 | 비밀 값으로 관리 |

`database-mysql-server`는 최초 빈 `database-data` 볼륨에서 `../db/init`을 실행한다. 기존 볼륨에서는 init script가 자동 재실행되지 않는다. `kb-migrator`는 MySQL health dependency를 선언하지 않았으므로 MySQL ready 이후 별도로 실행한다.

## 빌드 및 검증

명령은 저장소 루트에서 실행한다.

```bash
docker compose --project-directory main-infra-manage/docker \
	-f main-infra-manage/docker/docker-compose.yml config --quiet

docker compose --project-directory main-infra-manage/docker \
	-f main-infra-manage/docker/docker-compose.yml build \
	python-ai-model python-embedding-model kb-ingest qdrant-init
```

데이터베이스 초기화·마이그레이션과 서비스 기동은 다음 순서로 확인한다.

```bash
docker compose --project-directory main-infra-manage/docker \
	-f main-infra-manage/docker/docker-compose.yml up -d database-mysql-server qdrant-server

docker compose --project-directory main-infra-manage/docker \
	-f main-infra-manage/docker/docker-compose.yml run --rm kb-migrator

docker compose --project-directory main-infra-manage/docker \
	-f main-infra-manage/docker/docker-compose.yml up --build -d \
	qdrant-init python-embedding-model python-ai-model kb-ingest
```

확인 항목:

- `docker compose ps`에서 MySQL과 Qdrant가 실행 중이고 Qdrant healthcheck가 healthy인지 확인한다.
- `qdrant-init` 로그에서 `kb_accidents`, `kb_guides`와 payload index가 생성/확인됐는지 확인한다. 이 서비스는 성공 후 종료되는 것이 정상이다.
- AI `/health`, embedding `/ready`, ingest `/ready` 응답을 확인한다.
- Compose 내부 연결은 `database-mysql-server:3306`, `qdrant-server:6333`, `python-embedding-model:60110`을 사용한다.
- 이미지 재생성은 named volume 데이터를 삭제하지 않는다. `docker compose down -v`는 MySQL/Qdrant 데이터까지 삭제하므로 사용 전 반드시 확인한다.

2026-11-09 확인 결과:

- Compose 설정 검증과 `kb-ingest`, `python-embedding-model`, `qdrant-init` 이미지 빌드가 통과했다.
- ingest/RAG/config 테스트 47개와 embedding API 테스트 22개가 통과했다.
- 로컬 MySQL·임베딩 API·Qdrant 연동으로 문서 등록, point 색인, `EMBEDDED` 상태를 확인한 후 테스트 데이터를 정리했다.