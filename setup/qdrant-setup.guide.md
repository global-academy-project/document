# Qdrant 설정 가이드

## 1. 개요

Qdrant는 벡터와 메타데이터(payload)를 저장하고 유사도 검색과 조건 필터링을 제공하는 벡터 데이터베이스다. 임베딩 모델이 텍스트 등을 벡터로 변환하면 Qdrant에 저장하고, 검색할 때는 질의 벡터와 유사한 데이터를 찾는다. Qdrant 자체가 이 예제에서 텍스트를 임베딩하지는 않는다.

| 용어 | 설명 |
| --- | --- |
| Collection | 같은 벡터 설정을 사용하는 데이터의 묶음 |
| Point | ID, 벡터, 선택적인 payload로 구성된 데이터 |
| Vector | 임베딩 모델이 생성한 숫자 배열 |
| Payload | 필터링과 결과 표시 등에 사용하는 메타데이터 |

이 문서는 Docker를 이용한 **로컬 개발용 단일 인스턴스** 설정을 다룬다. Dockerfile 방식과 Docker Compose 방식은 대안이므로 **하나만 선택**한다. 두 예제는 동일한 컨테이너 이름과 포트를 사용한다.

## 2. 사전 준비

- Docker Engine 또는 Docker Desktop을 설치하고 실행한다.
- Compose 방식을 사용한다면 Docker Compose 플러그인도 필요하다.
- 아래 명령은 Bash 기준이다. Windows에서는 WSL 또는 Git Bash 등을 사용할 수 있다.
- 호스트의 `6333`, `6334` 포트가 비어 있어야 한다.

```bash
docker --version
docker compose version
```

예제의 이미지 `qdrant/qdrant:v1.20`은 현재 프로젝트의 인프라 Compose 설정과 같은 태그다. 운영 환경에서는 호환성을 검증한 버전으로 고정하고 업그레이드 전에 백업 및 마이그레이션 절차를 확인한다.

### 접근 주소

| 용도 | 주소 |
| --- | --- |
| REST API | `http://127.0.0.1:6333` |
| 웹 대시보드 | `http://127.0.0.1:6333/dashboard` |
| gRPC API | `127.0.0.1:6334` |

예제는 포트를 루프백 주소에만 바인딩하므로 호스트에서만 직접 접근할 수 있다. 다른 Compose 컨테이너에서는 호스트 포트가 아니라 서비스 이름과 컨테이너 포트(예: `http://qdrant:6333`)를 사용한다.

## 3. Getting Started: Dockerfile

### 3.1 이미지 생성

작업 디렉터리를 만들고 그 안에 다음 `Dockerfile`을 작성한다.

```dockerfile
FROM qdrant/qdrant:v1.20
```

공식 이미지를 그대로 기반으로 하므로 별도의 시작 명령을 지정할 필요가 없다. 이미지에 추가 설정이나 파일이 필요할 때 이 방식을 확장할 수 있다.

```bash
docker build -t local-qdrant:v1.20 .
```

### 3.2 컨테이너 실행

인증 키를 화면에 표시하지 않고 입력한 뒤 실행한다. 충분히 길고 무작위인 키를 사용하고, 터미널 환경변수와 Docker 컨테이너 설정을 볼 수 있는 사용자에게 키가 노출될 수 있으므로 로컬 개발용으로만 사용한다.

```bash
read -r -s -p "Qdrant API key: " QDRANT_API_KEY; echo
export QDRANT_API_KEY

docker run -d \
  --name qdrant-local \
  --restart unless-stopped \
  -p 127.0.0.1:6333:6333 \
  -p 127.0.0.1:6334:6334 \
  -v qdrant-local-data:/qdrant/storage \
  -e QDRANT__SERVICE__API_KEY="${QDRANT_API_KEY:?API key is required}" \
  local-qdrant:v1.20
```

기동 후 **5. 동작 확인 및 기본 사용법**으로 이동한다. 실행 중지와 재시작은 다음 명령을 사용한다.

```bash
docker stop qdrant-local
docker start qdrant-local
docker logs --tail 100 qdrant-local
```

## 4. Getting Started: Docker Compose

### 4.1 Compose 파일 작성

별도의 작업 디렉터리에 `compose.yaml`을 작성한다.

```yaml
services:
  qdrant:
    image: qdrant/qdrant:v1.20
    container_name: qdrant-local
    restart: unless-stopped
    ports:
      - "127.0.0.1:6333:6333"
      - "127.0.0.1:6334:6334"
    environment:
      QDRANT__SERVICE__API_KEY: ${QDRANT_API_KEY:?QDRANT_API_KEY is required}
    volumes:
      - qdrant-data:/qdrant/storage

volumes:
  qdrant-data:
    name: qdrant-local-data
```

### 4.2 실행

Dockerfile 방식과 마찬가지로 충분히 긴 무작위 키를 입력한다. Compose가 환경변수를 읽어 컨테이너에 전달한다.

```bash
read -r -s -p "Qdrant API key: " QDRANT_API_KEY; echo
export QDRANT_API_KEY

docker compose up -d
docker compose ps
docker compose logs --tail 100 qdrant
```

기동 후 **5. 동작 확인 및 기본 사용법**으로 이동한다. 중지 또는 재시작은 다음과 같다.

```bash
docker compose stop
docker compose start
```

`docker compose down`은 컨테이너와 네트워크를 제거하지만 기본적으로 데이터 볼륨은 유지한다. `docker compose down -v`는 볼륨을 함께 제거하므로 데이터가 필요한 경우 사용하지 않는다.

## 5. 동작 확인 및 기본 사용법

선택한 방식으로 서버를 기동하고 같은 셸에 `QDRANT_API_KEY`가 설정된 상태에서 실행한다. 서버가 준비될 때까지 잠시 기다린 뒤 확인한다.

```bash
curl --fail-with-body -sS \
  -H "api-key: $QDRANT_API_KEY" \
  http://127.0.0.1:6333/collections
```

JSON 응답이 반환되면 REST API와 인증을 사용할 수 있다. 대시보드에 접속할 때도 설정한 API 키가 필요하다. 아래 예제는 학습용 컬렉션과 임의의 4차원 벡터를 사용한다.

### 5.1 컬렉션 생성

실제 서비스에서는 임베딩 모델의 출력 차원과 컬렉션의 벡터 차원이 일치해야 한다. `Cosine`은 유사도 비교 방식이다.

```bash
curl --fail-with-body -sS -X PUT \
  -H "api-key: $QDRANT_API_KEY" \
  -H "Content-Type: application/json" \
  http://127.0.0.1:6333/collections/quickstart_demo \
  -d '{"vectors":{"size":4,"distance":"Cosine"}}'
```

### 5.2 데이터 저장 및 검색

```bash
curl --fail-with-body -sS -X PUT \
  -H "api-key: $QDRANT_API_KEY" \
  -H "Content-Type: application/json" \
  'http://127.0.0.1:6333/collections/quickstart_demo/points?wait=true' \
  -d '{"points":[
    {"id":1,"vector":[1.0,0.0,0.0,0.0],"payload":{"title":"첫 번째 문서"}},
    {"id":2,"vector":[0.0,1.0,0.0,0.0],"payload":{"title":"두 번째 문서"}}
  ]}'

curl --fail-with-body -sS -X POST \
  -H "api-key: $QDRANT_API_KEY" \
  -H "Content-Type: application/json" \
  http://127.0.0.1:6333/collections/quickstart_demo/points/query \
  -d '{"query":[1.0,0.0,0.0,0.0],"limit":2,"with_payload":true}'
```

응답의 `result.points`에서 ID `1`이 가장 높은 유사도 점수로 반환되는지 확인한다. 예제는 실제 임베딩 모델의 결과가 아니다.

### 5.3 예제 컬렉션 삭제

다음 명령은 **`quickstart_demo` 컬렉션과 그 데이터만 영구 삭제**한다.

```bash
curl --fail-with-body -sS -X DELETE \
  -H "api-key: $QDRANT_API_KEY" \
  http://127.0.0.1:6333/collections/quickstart_demo
```

## 6. 설정 및 운영 시 주의사항

- **데이터 영속성**: 두 예제는 `qdrant-local-data` Docker 볼륨을 `/qdrant/storage`에 연결한다. 컨테이너를 삭제하거나 재생성해도 볼륨을 삭제하지 않으면 데이터가 유지된다. Dockerfile 방식에서 Compose 방식으로 전환할 때는 기존 컨테이너를 먼저 제거해야 한다.
- **백업**: 볼륨은 백업을 대체하지 않는다. 운영 데이터는 Qdrant 스냅샷을 별도 저장소에 보관하고 복원 절차를 정기적으로 검증한다.
- **인증**: 기본 Qdrant 설정에는 인증이 없으므로 예제에서는 `QDRANT__SERVICE__API_KEY`를 설정한다. 요청 시 `api-key` 헤더를 사용한다. 키는 소스 코드나 문서에 저장하지 말고 안전하게 관리한다.
- **네트워크와 TLS**: 예제의 HTTP 연결은 암호화되지 않는다. 외부에 공개할 때는 API 키만으로 충분하지 않으며 TLS, 방화벽 및 접근 제어를 적용해야 한다. 특히 인터넷에 무인증으로 노출하지 않는다.
- **설정 변경**: Qdrant 환경변수는 `QDRANT__` 접두사와 중첩 설정을 구분하는 이중 밑줄을 사용한다. Compose에서 키를 변경한 뒤에는 새 키를 export하고 `docker compose up -d`로 컨테이너를 재생성한다. Dockerfile 방식은 기존 컨테이너를 제거하고 새 환경변수로 다시 실행한다. 볼륨은 삭제하지 않는다.
- **프로젝트 연동**: 기존 인프라 Compose는 `qdrant-server`라는 서비스명, 호스트 REST 포트 `60011`, 읽기 전용 키와 쓰기 키를 사용한다. 다른 컨테이너는 `http://qdrant-server:6333`, 호스트는 `http://127.0.0.1:60011`로 접속한다. 이 문서의 독립 예제와 포트 및 서비스명이 다르므로 혼동하지 않는다.

## 7. 문제 해결

| 증상 | 확인 및 조치 |
| --- | --- |
| 포트가 이미 사용 중 | 다른 Qdrant 인스턴스 또는 해당 포트를 사용하는 프로세스를 확인한다. 필요하면 호스트 포트만 변경하고 접속 주소도 함께 변경한다. |
| `401` 또는 `403` 응답 | 요청의 `api-key`가 서버에 설정된 키와 일치하는지, 읽기 전용 키로 쓰기 작업을 시도하지 않았는지 확인한다. |
| 연결 실패 | Docker 실행 여부, 컨테이너 상태 및 로그, 포트 매핑을 확인한다. 컨테이너 내부에서 `localhost`는 해당 컨테이너 자신을 가리킨다. |
| 벡터 차원 오류 | 컬렉션의 `size`와 저장·검색 벡터의 차원, 사용 중인 임베딩 모델을 확인한다. |
| 데이터가 보이지 않음 | 같은 볼륨이 `/qdrant/storage`에 연결되어 있는지, 다른 인스턴스나 컬렉션에 접속하지 않았는지 확인한다. |

## 8. 참고 자료

- [Qdrant 로컬 빠른 시작](https://qdrant.tech/documentation/quickstart/)
- [Qdrant 설정](https://qdrant.tech/documentation/ops-configuration/configuration/)
- [Qdrant 보안](https://qdrant.tech/documentation/security/)
- [Qdrant 스냅샷](https://qdrant.tech/documentation/concepts/snapshots/)
- [Qdrant API 참조](https://api.qdrant.tech/)
