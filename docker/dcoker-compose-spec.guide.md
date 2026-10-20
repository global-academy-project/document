# 1. Docker Compose Spec Guide

```yml
spring-server-backend:
    container_name: spring-server-container
    shm_size: '1024mb'
    restart: always
    build:
      context: ../../server/backend
      dockerfile: Dockerfile
    environment: 
      MYSQL_HOST: ${SPRING_MYSQL_HOST}
      MYSQL_PORT: ${SPRING_MYSQL_PORT}
      MYSQL_DATABASE: ${SPRING_MYSQL_DATABASE}
      MYSQL_USER: ${SPRING_MYSQL_USER}
      MYSQL_PASSWORD: ${SPRING_MYSQL_PASSWORD}
      JWT_SECRET: ${JWT_SECRET}
    ports:
      - "${SPRING_SERVER_PORT}:${SPRING_SERVER_PORT}"
    depends_on:
      ocr-mysql:
        condition: service_healthy
```

이 Compose 파일은 Spring Boot 서버를 구성하는 핵심 설정을 정의한 구성이다. `spring-server-backend`는 실제로 실행되는 서비스 이름이며, 이 이름은 Docker Compose 내부에서 다른 서비스와 네트워크 연결을 위해 참조된다. 예를 들어 MySQL 서비스와 연동할 때, 애플리케이션 코드에서는 `MYSQL_HOST`를 Docker Compose 내부 네트워크 이름 기준으로 설정해야 하며, 이 값은 서비스 이름과 일치해야 한다.

Docker Compose는 단순히 Container를 띄우는 도구가 아니라, 여러 서비스의 의존 관계, 환경 변수, 포트 노출, 네트워크 설정을 한 번에 정의하는 배포 스펙이다. 따라서 이 파일은 서버 애플리케이션이 어떤 환경에서 실행되는지, 어떤 DB와 연결되는지, 언제 실행되기 시작하는지를 명확하게 표현한다.

#### 1. Service Name

```yml
spring-server-backend:
```

Compose 파일의 최상위 키는 각 서비스의 이름을 의미한다. 여기서는 `spring-server-backend`라는 이름으로 Spring Boot 서버를 실행하고 있다. 서비스 이름은 컨테이너의 논리적 식별자로 사용되며, 같은 Compose 파일 안에서 다른 서비스와 연결할 때 가장 중요하게 쓰인다.

예를 들어 `depends_on`에서 `ocr-mysql`을 참조하거나, 환경 변수에서 DB Host를 설정할 때 서비스 이름이 실제 연결 대상이 된다. 따라서 서비스 이름은 단순한 라벨이 아니라 네트워크 내 통신을 위한 핵심 식별자라고 볼 수 있다.

#### 2. Container Name

```yml
container_name: spring-server-container
```

`container_name`은 생성되는 Container의 이름을 고정시키는 설정이다. Docker Compose는 보통 자동으로 컨테이너 이름을 생성하지만, 이 옵션을 사용하면 특정 이름으로 관리할 수 있다.

개발 환경에서는 컨테이너를 직접 확인하거나 로그를 추적할 때 이름을 고정해 두는 것이 편리하다. 예를 들어 `docker ps` 명령으로 현재 실행 중인 서버를 빠르게 찾을 수 있고, 디버깅 또는 로그 확인 시 혼동을 줄일 수 있다.

다만 `container_name`은 여러 환경에서 충돌이 발생할 수 있으므로, 한 프로젝트에서 여러 Compose 파일을 동시에 실행할 때는 이름 중복을 주의해야 한다. 일반적으로 로컬 개발 환경에서는 관리 편의성을 위해 사용하지만, 다중 인스턴스 환경에서는 유연성이 떨어질 수 있다는 점을 함께 고려해야 한다.

#### 3. Memory Size

```yml
shm_size: '1024mb'
```

`shm_size`는 컨테이너의 shared memory 크기를 설정하는 항목이다. Docker 환경에서는 메모리 공유 영역이 애플리케이션의 병렬 처리, 캐시, IPC 통신과 같은 동작에 영향을 줄 수 있다.

특히 Java 기반 서버처럼 메모리 사용량이 높은 애플리케이션에서는 기본값으로 충분하지 않을 수 있다. `1024mb`를 지정함으로써 애플리케이션이 필요로 하는 메모리 공유 영역을 넉넉하게 확보할 수 있다.

이 설정은 서버 성능과 안정성에 직접적인 영향을 줄 수 있으며, 특히 다수의 스레드 또는 고부하 환경에서 발생할 수 있는 메모리 관련 문제를 완화하는 데 도움이 된다. 단, 실제 서버 환경에서는 전체 호스트 메모리에 비례해 너무 큰 값을 설정하면 오히려 불필요한 자원 소모가 발생할 수 있으므로, 애플리케이션 요구사항에 맞게 조절하는 것이 중요하다.

#### 4. Restart Policy

```yml
restart: always
```

`restart: always`는 컨테이너가 종료되었을 때 자동으로 다시 시작하도록 설정하는 정책이다. 서버 애플리케이션은 예기치 않은 종료가 발생할 수 있으므로, 이 옵션을 사용하면 운영 중 장애를 최소화할 수 있다.

특히 Java 서버 또는 MySQL 같이 백그라운드에서 동작하는 서비스는 일시적인 오류로 인해 종료될 수 있기 때문에, 자동 재시작 정책을 적용하는 것이 안정적인 서비스 운영에 유리하다. 다만 개발 환경이나 테스트 환경에서는 의도적인 중지를 위해 별도로 조절할 필요가 있기 때문에, 서비스 특성에 따라 정책을 적절하게 선택해야 한다.

#### 5. Build Configuration

```yml
build:
  context: ../../server/backend
  dockerfile: Dockerfile
```

`build`는 이미지를 빌드할 때 필요한 경로와 Dockerfile 정보를 정의한다. `context`는 빌드의 기준 경로를 의미하며, 여기서는 `../../server/backend`를 기준으로 빌드가 수행된다. 즉, Spring Boot 서버 코드가 위치한 디렉터리를 Docker 빌드 컨텍스트로 지정하는 것이다.

`dockerfile: Dockerfile`은 해당 경로 안에서 실제 이미지를 생성하는 Dockerfile 파일을 지정한다. 이 설정을 통해 소스 코드와 실행 환경을 함께 패키징한 이미지를 생성할 수 있으며, 개발 또는 배포 환경에서 동일한 실행 환경을 유지하는 데 큰 장점이 있다.

Docker Compose에서 `build`를 사용하면 로컬 소스 코드를 기준으로 이미지를 자동 생성하므로, 별도의 수동 빌드 과정 없이 서버 컨테이너를 실행할 수 있다. 이는 개발 편의성을 높이고, 배포 시에도 일관된 이미지 상태를 유지하는 데 도움이 된다.

#### 6. Environment Variables

```yml
environment: 
  MYSQL_HOST: ${SPRING_MYSQL_HOST}
  MYSQL_PORT: ${SPRING_MYSQL_PORT}
  MYSQL_DATABASE: ${SPRING_MYSQL_DATABASE}
  MYSQL_USER: ${SPRING_MYSQL_USER}
  MYSQL_PASSWORD: ${SPRING_MYSQL_PASSWORD}
  JWT_SECRET: ${JWT_SECRET}
```

`environment`는 컨테이너 내부에서 필요한 변수들을 전달하는 설정이다. Spring Boot 서버가 MySQL에 연결하기 위해 필요한 DB 주소, 포트, 데이터베이스 이름, 사용자 정보, 비밀번호를 환경 변수로 받아 사용한다. 이 값들은 보통 `.env` 파일이나 외부 설정 파일에서 관리되며, 실행 환경에 따라 유연하게 변경할 수 있다.

특히 `JWT_SECRET`은 인증 토큰 서명을 위해 사용되는 민감한 값으로, 코드에 하드코딩하는 대신 환경 변수로 관리하는 것이 보안상 안전하다. 또한 환경 변수를 사용하면 개발 환경과 운영 환경을 분리해서 설정할 수 있어 배포 과정이 훨씬 안정적이 된다.

환경 변수는 애플리케이션 설정을 코드에서 분리해 관리할 수 있게 해주는 핵심 기능이다. 이 설정이 잘 구성되어 있어야 서버는 동일한 코드라도 각 환경에 맞는 동작을 수행할 수 있다.

#### 7. Port Mapping

```yml
ports:
  - "${SPRING_SERVER_PORT}:${SPRING_SERVER_PORT}"
```

`ports`는 호스트와 컨테이너 간의 포트 매핑을 설정한다. 여기서는 서버 애플리케이션의 내부 포트와 외부 포트를 동일하게 연결하거나, 호스트에서 접근 가능한 포트로 매핑해 주고 있다.

예를 들어 `SPRING_SERVER_PORT` 값이 `8080`이라면, 호스트의 `8080` 포트로 접근했을 때 컨테이너의 `8080` 포트로 전달된다. 이를 통해 브라우저 또는 외부 클라이언트가 애플리케이션에 접속할 수 있다.

포트 매핑은 단순히 서버 접속 경로를 열어 주는 기능이 아니라, 애플리케이션을 외부에서 사용할 수 있도록 네트워크 경로를 연결하는 핵심 설정이다. 따라서 포트 값은 서버 설정 파일과 일치해야 하며, 충돌이 발생하지 않도록 미리 확인해야 한다.

#### 8. Dependency Management

```yml
depends_on:
  ocr-mysql:
    condition: service_healthy
```

`depends_on`은 서비스 의존 관계를 정의하는 옵션이다. 여기서는 Spring Boot 서버가 MySQL 서비스보다 먼저 실행되기보다는, MySQL이 정상 상태인 이후에 시작되도록 설정하고 있다.

`condition: service_healthy`는 MySQL 서비스가 Health Check 기준을 통과한 상태일 때만 다음 단계로 진행하도록 보장한다. 이 설정은 DB 초기화가 끝나지 않은 상태에서 애플리케이션이 연결을 시도하는 문제를 줄여 준다.

이러한 의존성 설정은 MSA 구조에서도 중요한 역할을 한다. 다수의 서비스가 함께 동작할 때, 특정 서비스가 준비되지 않은 상태에서 다른 서비스가 실행되면 오류가 발생할 수 있으므로, 의존성 순서를 보장하는 것이 매우 중요하다.

결론적으로 이 Compose 파일은 단순한 실행 설정이 아니라, Spring Boot 서버와 MySQL 데이터베이스 간의 연결, 환경 변수 구성, 외부 접근 포트, 서비스 시작 순서 등을 한 번에 정의하는 배포 스펙이다. 각각의 항목은 애플리케이션이 안정적으로 동작할 수 있도록 설계된 핵심 요소이며, 환경 구성 시 반드시 함께 이해하고 관리해야 한다.
