# AI 모델 Dockerfile 및 개발 Compose 변경 기록

## 범위

이 문서는 `main-ai-model/Dockerfile`과 개발용 `main-infra-manage/docker/docker-compose.dev.yml`의 변경 사항 및 검증 결과를 기록한다. 현재 개발 Compose는 `Dockerfile.dev`가 아니라 `Dockerfile`을 사용한다. 운영용 `docker-compose.yml`에는 아직 변경을 적용하지 않았다.

## 베이스 이미지 버전 선택

현재 프로젝트의 Dockerfile 세 개는 다음 베이스 이미지를 사용한다.

| 서비스 | Dockerfile 베이스 이미지 | 의미 및 선택 이유 |
| --- | --- | --- |
| 웹 프론트엔드 | `node:24.22.2-alpine3.24` | Node.js 24.22.2와 Alpine Linux 3.24를 명시한다. 앱 빌드에는 Node 환경이 필요하므로 그대로 사용하며, 실제 이미지 크기는 빌드 결과로 확인한다. |
| Spring 서버 빌드 단계 | `eclipse-temurin:17-jdk-resolute` | Temurin Java 17 JDK와 Ubuntu Resolute 기반 빌드 환경을 사용한다. Gradle 실행 및 JAR 생성에 사용되며 멀티스테이지 빌드의 최종 런타임 이미지에는 들어가지 않는다. |
| Spring 서버 런타임 단계 | `eclipse-temurin:17-jre-resolute` | 같은 Java 및 Ubuntu 계열의 JRE만 최종 이미지에 포함한다. JDK가 아닌 JRE를 사용해 실행 이미지를 구성한다. |
| AI 모델 서비스 | `python:3.12.15-slim` | Python 3.12.15와 slim 변형을 명시한다. ML wheelhouse, AI/임베딩, 경량 ingest stage가 이 공통 베이스를 상속한다. |

### Noble과 Resolute 비교

Spring 서버 런타임 JRE를 `noble`과 `resolute`로 바꾸어 로컬 이미지 크기를 비교했다.

| 비교 대상 | Noble | Resolute | 차이 |
| --- | ---: | ---: | ---: |
| Temurin 17 JRE 베이스 이미지 | 376.76 MB | 432.11 MB | Noble이 55.35 MB 작음 |
| 빌드된 Spring 서버 이미지 | 507.74 MB | 561.23 MB | Noble이 53.49 MB 작음 |

측정값상 Noble 이미지가 작았지만, 최종 선택은 지원 기간을 우선해 `resolute`로 했다. Noble은 Ubuntu 24.04 LTS, Resolute는 Ubuntu 26.04 LTS 기반이다. 따라서 이 변경은 용량 최소화보다는 더 긴 지원 기간을 우선한 선택이다. Noble 빌드는 61.5초, Resolute 빌드는 28.1초였지만 베이스 이미지 다운로드와 캐시 상태가 달라 빌드 속도 비교 자료로는 사용할 수 없다.

태그에 버전과 배포판 코드명이 지정되어 있어도 태그 자체는 digest처럼 불변 식별자가 아니다. 같은 태그가 보안 및 상위 이미지 업데이트로 새 이미지를 가리킬 수 있다. 바이트 단위 재현성이 필요하면 digest 고정을 검토하고, 고정한 경우 보안 업데이트에 맞춰 digest를 갱신하는 절차를 둔다.

## Dockerfile 변경

### 변경 전 문제

- `requirements.release.txt`만 복사했지만 파일 안에서 `requirements.txt`를 추가로 참조했다.
- `--no-index --find-links=/tmp/wheels`로 설치하면서 wheelhouse를 생성하는 단계가 없어 빌드가 실패했다.
- 가상환경을 중복 생성하는 명령이 있었다.
- 임베딩 타깃의 Uvicorn 경로가 `tool_backend:create_app`을 가리켜 실제 앱 팩토리를 찾지 못했다.
- 개발 Compose가 `ingest` 타깃을 선택했지만 Dockerfile에는 해당 타깃이 없거나, 존재하지 않는 앱 경로를 참조하는 구성이 있었다.

### 현재 stage 구성

```text
base
├── ml-wheelhouse
└── ml-dependencies
  ├── ai
  └── embedding

base
└── ingest-dependencies
  └── ingest
```

| Stage | 구체적인 동작 | 최종 이미지 포함 |
| --- | --- | --- |
| `base` | `python:3.12.15-slim`, `/opt/venv/bin` 우선 PATH, Python 로그 즉시 출력, `.pyc` 생성 억제, `/app` 작업 경로 설정 | 부모 레이어 |
| `ml-wheelhouse` | `requirements.txt`와 `requirements.release.txt`를 복사하고 `pip download --only-binary=:all:`로 wheel 다운로드. `/root/.cache/pip`를 BuildKit 캐시 마운트로 사용 | 미포함 |
| `ml-dependencies` | wheelhouse를 복사해 `/opt/venv`에 `--no-index`로 오프라인 설치, `pip check` 실행 후 복사한 wheel 삭제 | `ai`, `embedding`에서 공유 |
| `ai` | `app/` 전체 복사, `uvicorn app.main:app`, 포트 60100 | 포함 |
| `embedding` | `app/` 전체 복사, `uvicorn app.rag.embedding_api:create_app --factory`, 포트 60110 | 포함 |
| `ingest-dependencies` | `requirements.ingest.txt`만 설치하고 `pip check` 실행. PyTorch/Transformers는 설치하지 않음 | `ingest`에서 사용 |
| `ingest` | `app/` 복사, `uvicorn app.ingest_main:app`, 포트 8001 | 포함 |

Stage 별칭을 소문자로 통일해 Compose의 `target: ai`, `target: embedding`, `target: ingest`와 일치시켰다. API와 임베딩은 현재 둘 다 Torch 및 Transformers를 사용하므로 `ml-dependencies`를 공유한다. Wheelhouse와 pip 다운로드 캐시는 최종 런타임 이미지에 남지 않는다.

## 개발 Compose 변경

### 서비스와 Docker 타깃

| Compose 서비스 | Dockerfile 타깃 | 컨테이너 포트 | 호스트 포트 |
| --- | --- | ---: | --- |
| `python-ai-model` | `ai` | 60100 | `${AI_MODEL_PORT}` |
| `python-embedding-model` | `embedding` | 60110 | `${EMBEDDING_MODEL_PORT}` |
| `kb-ingest` | `ingest` | 8001 | `${KB_INGEST_PORT}` |

### 추가 및 수정한 설정

- `python-ai-model`의 `NETWORK_ENV` 기본값은 `OPEN`, `CHECK_RULES_TOOL` 기본값은 애플리케이션 허용값인 `MOCK`이다.
- AI 모델과 임베딩 모델의 다운로드 캐시를 각각 `ai-model-cache:/models/huggingface`, `embedding-model-cache:/models/huggingface`에 보존한다.
- 임베딩 컨테이너는 `EMBEDDING_LOCAL_FILES_ONLY=false`로 실행해 캐시가 비어 있으면 Hugging Face Hub에서 모델을 받을 수 있게 한다.
- `kb-migrator`의 URL은 실제 개발 Compose 서비스명과 내부 포트를 사용한다: `http://python-embedding-model:60110/v1/embeddings`.
- Spring 서비스의 `APP_AGENT_BASE_URL`과 AI 서비스의 RAG 환경 변수 연결은 아직 완료되지 않았다. 상세 내용과 남은 작업은 [Spring-AI 연동 문서](spring-ai-integration.md)에 분리했다.

Compose 네트워크 안에서 컨테이너 간 요청은 호스트 공개 포트가 아닌 서비스명과 컨테이너 포트를 사용한다. 예를 들어 migrator는 `python-embedding-model:60110`으로 연결한다.

## 검증 결과

2026-11-01 개발 환경 기준:

- `docker compose -f main-infra-manage/docker/docker-compose.dev.yml config --quiet` 통과.
- `python-ai-model`, `python-embedding-model`, `kb-ingest` 세 타깃 빌드 통과. 최초 세 타깃 동시 빌드는 약 81초였다.
- 각 이미지에서 AI API, 임베딩 앱 팩토리, ingest 앱의 진입점 import를 확인했다.
- 임베딩 API에 한국어 문장 4개를 전송해 HTTP 200을 확인했다. 응답 모델은 `BAAI/bge-m3`, 벡터는 4개이며 각 벡터 차원은 1024였다. 전체 `prompt_tokens`/`total_tokens`는 106이었다.
- 각 임베딩 벡터의 L2 norm은 1.000000이었다.
- 화학 안전 문장끼리 코사인 유사도는 `0.799271`, `0.753345`, `0.751789`였다. 도서관 문장과 화학 안전 문장 간 유사도는 `0.418287`, `0.402556`, `0.378848`이었다. 시험 문장 세트에서 관련 문장이 상대적으로 더 가까웠다.
- 공개 모델의 첫 다운로드 때 Hugging Face 인증 안내가 출력됐지만 다운로드와 추론은 성공했다. HF 토큰은 필수로 설정하지 않았다.

이 결과는 각 이미지 빌드, 모듈 import, 임베딩 API 단독 요청까지 검증한 것이다. Spring Boot에서 시작해 AI API, 임베딩 API, Qdrant까지 이어지는 통합 검증은 아직 수행하지 않았다. 운영 Compose에도 변경을 적용하지 않았다.