# main-server Dockerfile 개선 명세

## 범위

이 문서는 `main-server`의 Docker 이미지 빌드와 배포 전 검증만 다룬다. 다른 서비스의 Dockerfile이나 Compose 구성은 범위에 포함하지 않는다.

## 목표

- Java 소스가 바뀌어도 Gradle Wrapper와 의존성 캐시를 재사용해 반복 빌드 시간을 줄인다.
- 개발 환경의 로컬 설정 파일과 불필요한 파일이 Docker 빌드 컨텍스트에 포함되지 않게 한다.
- 빌드 도구가 최종 런타임 이미지에 들어가지 않도록 한다.
- 검증한 이미지와 실제 배포할 이미지를 동일하게 유지한다.

## 현재 설계

`main-server/Dockerfile`은 멀티스테이지 빌드다. 빌드 단계는 Eclipse Temurin 17 JDK의 `resolute` 태그를 사용하고, 최종 단계는 같은 계열의 JRE를 사용한다.

Gradle Wrapper, `gradle/`, `settings.gradle`, `build.gradle`을 먼저 복사한 뒤 `dependencies`를 실행한다. 이후 `src/`만 복사하고 `bootJar`를 생성한다. 두 Gradle 실행 단계는 BuildKit 캐시 마운트(`/root/.gradle`)를 공유하고, JAR만 최종 JRE 이미지로 복사한다.

현재 `.dockerignore`는 `.git`, 로컬 환경 파일, Gradle/build 산출물, IDE 파일, 문서, 테스트 소스를 제외한다. `application-local` 및 확장자가 붙은 로컬 설정 파일도 제외한다.

최초 Dockerfile은 비교를 위해 주석 블록으로 보존되어 있다. 실제 빌드에 사용되는 것은 그 아래의 `Optimized version (active)` 블록이다.

## 캐시 동작과 기대 효과

- 소스 파일만 바뀌면 설정 파일 복사 및 `dependencies` 단계는 Docker 레이어 캐시를 재사용할 수 있다.
- `bootJar` 단계는 소스 변경에 따라 다시 실행되지만, Gradle 캐시 마운트에서 Wrapper 배포본, 의존성, 작업 캐시를 재사용할 수 있다.
- `build.gradle`, `settings.gradle`, `gradle/` 또는 `gradlew`가 바뀌면 의존성 준비 레이어가 다시 실행되는 것이 정상이다.
- 멀티스테이지 구조로 인해 최종 이미지에는 JRE와 실행 JAR만 포함되고 JDK 및 빌드 중간 산출물은 포함되지 않는다.
- `.dockerignore`는 빌드 컨텍스트 전송량을 줄이고 로컬 설정이 이미지 빌드 입력에 섞이는 것을 방지한다.

BuildKit 캐시 마운트는 최종 이미지 안에 저장되는 캐시가 아니다. 동일한 Docker/BuildKit 빌더를 재사용하는 로컬 환경에서는 효과가 지속된다. CI가 매번 새 빌더를 사용한다면 Gradle 캐시 마운트만으로는 빌드 간 캐시가 이어지지 않을 수 있으므로, BuildKit 원격 캐시 또는 CI 캐시를 별도로 구성한다.

## 빌드 및 캐시 검증

`main-server` 디렉터리에서 동일한 빌드 명령을 연속 실행해 첫 빌드와 재빌드 로그를 비교한다.

```powershell
docker build --progress=plain -t main-server:cache-check .
docker build --progress=plain -t main-server:cache-check .
```

두 번째 빌드에서는 변경이 없는 Docker 레이어와 Gradle 의존성 준비 단계가 캐시되는지 확인한다. 소스 변경이 있는 경우에는 의존성 준비 단계가 캐시되고 `bootJar` 단계만 다시 실행되는지 확인한다. 빌드 시간은 베이스 이미지 다운로드 여부와 빌더의 캐시 상태에 따라 달라지므로, 동일한 조건에서 비교한다.

## 테스트와 배포 전 검증

현재 `.dockerignore`에서 `src/test/`를 제외하고 Dockerfile은 `bootJar`만 실행한다. 따라서 Docker 이미지 빌드가 단위 테스트 성공을 의미하지 않는다.

- 단위 테스트는 CI의 체크아웃된 소스에서 별도로 Gradle 테스트 태스크를 실행한다.
- 이미지 빌드 후에는 그 이미지로 컨테이너를 실행해 시작 여부와 HTTP 헬스 엔드포인트 등 스모크 테스트를 수행한다.
- 검증을 통과한 이미지를 다시 빌드하지 않고 동일한 이미지 태그 또는 digest로 배포한다.

## 후속 개선 우선순위

1. 로컬에서 연속 빌드 및 소스 변경 후 빌드를 측정하고 캐시 로그를 확인한다.
2. CI 빌더가 매 실행마다 새로 만들어지는 경우 원격 BuildKit 캐시를 설정한다.
3. 배포 보안을 강화할 때 최종 이미지에 비특권 실행 사용자를 추가하고, JAR 파일 소유권을 해당 사용자에 맞춘다.
4. 이미지 재현성이 필요한 경우 베이스 이미지를 digest로 고정한다. digest 고정은 자동 업데이트를 막으므로 보안 업데이트 시 정기적으로 갱신하는 절차도 함께 둔다.

`resolute`를 선택하는 이유는 빌드 및 런타임 단계의 Ubuntu 계열을 일치시키기 위해서다. 이 태그 자체가 빌드 속도나 CPU 성능을 보장하지는 않는다. 빌드 시간 개선 여부는 캐시 적중 로그와 동일 조건의 측정값으로 판단한다.
