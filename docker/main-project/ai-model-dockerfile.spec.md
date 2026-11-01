# AI 모델 Dockerfile 개발 명세서

## 목적 및 범위

이 명세서는 실험용 `main-ai-model/Dockerfile.dev`와 `main-infra-manage/docker/docker-compose.dev.yml`을 통한 검증 방법을 설명한다. 운영용 Compose 설정은 이 문서의 수정 대상이나 범위에 포함하지 않는다. 개발 설정의 검증이 끝나면 필요한 타깃 및 실행 설정을 운영 Compose 파일에 반영할 수 있다.

## 설계 목표

- API, 임베딩 API, KB 수집 서비스를 각각 명시적인 Docker 타깃으로 빌드한다.
- 용량이 큰 PyTorch 및 Transformers 의존성 레이어를 API와 임베딩 타깃에서 공유한다.
- KB 수집 이미지에는 ML 의존성이 포함되지 않도록 한다.
- 다운로드한 Python wheel 파일은 최종 런타임 이미지에 포함하지 않는다.
- Hugging Face 모델 파일은 이미지 레이어가 아니라 영속 개발 볼륨에 저장한다.
- 이미지별 기본 실행 명령과 컨테이너 포트를 명확히 지정해 Compose에서 Dockerfile 진입점을 직접 검증할 수 있게 한다.

## 스테이지 구성

| 스테이지 | 역할 | 최종 이미지 포함 여부 |
| --- | --- | --- |
| `base` | Python 3.12 실행 환경, 공통 환경 변수, `/app` 작업 디렉터리를 제공한다. | 부모 레이어로 포함 |
| `ml-wheelhouse` | `requirements.release.txt`에 필요한 바이너리 wheel을 다운로드한다. BuildKit이 pip 다운로드 캐시를 관리한다. | 미포함 |
| `ml-dependencies` | `/opt/venv`를 만들고 wheelhouse에서 ML 의존성을 오프라인 설치한 뒤 `pip check`를 실행한다. | `ai`, `embedding`에 포함 |
| `ingest-dependencies` | `requirements.ingest.txt`에서 FastAPI와 Uvicorn만 설치한다. | `ingest`에 포함 |
| `ai` | 애플리케이션을 복사하고 `app.main:app`을 포트 60100에서 실행한다. | 포함 |
| `embedding` | 애플리케이션을 복사하고 팩토리로 `app.rag.embedding_api:create_app`을 포트 8000에서 실행한다. | 포함 |
| `ingest` | 애플리케이션을 복사하고 `app.ingest_main:app`을 포트 8001에서 실행한다. | 포함 |

현재 API와 임베딩 서비스 모두 Torch와 Transformers가 필요하다. 임베딩 인코더가 두 패키지를 직접 가져오며, API의 기본 `CATEGORIZATOR` 엔진도 로컬 Transformers 모델을 불러온다. 따라서 용량이 큰 의존성 레이어를 두 이미지에 따로 만들지 않고 공유한다. 경량 ingest 서비스는 두 패키지를 가져오지 않으므로 별도의 의존성 스테이지를 사용한다.

## Wheel 및 의존성 캐시

`ml-wheelhouse`는 설치 전에 wheel의 의존성을 확인하고 파일을 다운로드한다. BuildKit 캐시 마운트에 연결한 pip 캐시는 이미지 레이어에 저장되지 않는다. `ml-dependencies`는 `--no-index` 옵션으로 `/wheelhouse`에서 패키지를 설치하고, 설치 환경을 검사한 뒤 복사된 wheelhouse를 삭제한다. 최종 API 및 임베딩 이미지에는 가상환경만 포함되고 다운로드한 wheel 파일은 포함되지 않는다.

ML requirements 파일 중 하나가 바뀌면 wheelhouse 레이어가 다시 생성된다. 소스만 바뀐 경우에는 wheel 다운로드 및 설치 레이어를 재사용할 수 있다. ingest 의존성 레이어는 ML requirements 변경과 무관하다.

pip 캐시 마운트를 사용하려면 BuildKit이 필요하다. Docker Desktop의 일반 `docker build`는 BuildKit을 사용한다. CI가 매번 새 빌더를 사용한다면 BuildKit/buildx를 활성화하고 원격 캐시를 구성해야 한다.

## 개발 Compose 연결

`docker-compose.dev.yml`은 다음 타깃을 선택한다.

| 서비스 | Docker 타깃 | 컨테이너 포트 | Hugging Face 캐시 |
| --- | --- | --- | --- |
| `python-ai-model` | `ai` | 60100 | `ai-model-cache` |
| `python-embedding-model` | `embedding` | 8000 | `embedding-model-cache` |
| `kb-ingest` | `ingest` | 8001 | 불필요 |

임베딩 서비스는 Dockerfile의 `CMD`를 직접 사용하며 Compose의 `command`로 덮어쓰지 않는다. 호스트 포트 기본값은 60110이며 이미지 내부의 8000번 포트에 연결한다. Hugging Face 볼륨은 컨테이너를 재생성해도 모델 다운로드를 보존하면서 이미지 크기를 늘리지 않는다.

## 검증

저장소 루트에서 Compose 설정을 검증하고 세 타깃을 각각 빌드한다.

```powershell
docker compose -f main-infra-manage/docker/docker-compose.dev.yml config
docker compose -f main-infra-manage/docker/docker-compose.dev.yml build python-ai-model
docker compose -f main-infra-manage/docker/docker-compose.dev.yml build python-embedding-model
docker compose -f main-infra-manage/docker/docker-compose.dev.yml build kb-ingest
```

빌드가 통과하면 서비스를 실행해 API 상태 확인 엔드포인트, 모델 로딩이 끝난 뒤 임베딩 `/ready` 엔드포인트, ingest `/ready` 엔드포인트를 확인한다. 타깃별 이미지 크기와 빌드 로그도 비교한다. API와 임베딩 이미지는 의도적으로 ML 런타임을 공유하므로 계속 큰 편이고, ingest 이미지는 해당 비용을 피하는 것이 목표다.

2026-11-01 최초 개발 검증 결과:

- Compose에서 `ai`와 `embedding` 타깃을 함께 빌드하는 데 101초가 걸렸다. ML wheel 다운로드 및 설치 레이어를 공유했다.
- `ingest` 타깃은 최초 단독 빌드에 10.6초가 걸렸고, 이후 Compose 빌드에는 2초가 걸렸다.
- 각 이미지에서 API, 임베딩, ingest Python 진입점 모듈을 정상적으로 가져오는 것을 확인했다.
- 로컬 이미지 목록에서 ML 이미지 각각은 2.24 GB, ingest 이미지는 252 MB로 표시됐다. Docker는 동일한 의존성 레이어를 로컬에서 공유하므로 두 ML 이미지의 크기를 단순 합산해 중복 저장량으로 보면 안 된다. 같은 레지스트리에 푸시하는 경우 전송 및 저장에서도 레이어를 공유할 수 있다.
- 이 빌드 및 import 검증에서는 HTTP 서버 기동, API 상태 확인, 모델 로딩 후 임베딩 준비 상태를 확인하지 않았다.

개발 빌드 및 런타임 검증이 끝나기 전에는 이 실험용 Dockerfile을 운영 경로에 복사하지 않는다. 검증된 타깃 선택, 포트, 모델 캐시 볼륨 설정만 운영 Compose에 반영한다.