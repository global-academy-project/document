# Prometheus / Grafana Compose 구성

작성일: 2026-11-19

## 목적

Prometheus는 애플리케이션과 컨테이너에서 메트릭을 수집하고, Grafana는 수집된 데이터를 시각화한다.
Grafana는 Prometheus가 없으면 시각화할 메트릭 데이터를 가져올 수 없다.

## 현재 Docker Compose 구성

`docker/docker-compose.yml`의 모니터링 서비스는 다음과 같다.

| 서비스 | 내부 주소 | 호스트 포트 | 용도 |
|---|---|---:|---|
| Prometheus | `http://prometheus:9090` | `9090` | 메트릭 수집 및 저장 |
| Grafana | `http://grafana:3000` | `3000` | 메트릭 시각화 |

두 컨테이너는 Compose 기본 네트워크인 `default` 브리지 네트워크에 연결된다.

## 로컬 실행

```powershell
cd C:\projects\org\main-infra-manage\docker
docker compose up -d prometheus grafana
```

접속 주소:

- Prometheus: `http://localhost:9090`
- Grafana: `http://localhost:3000`

Grafana 최초 로그인 정보는 이미지의 기본값을 사용하며, 일반적으로 사용자 이름은 `admin`이다.
첫 접속 후 비밀번호를 변경해야 한다.

## Grafana 데이터 소스 등록

1. Grafana 접속
2. `Connections` 또는 `Data sources` 메뉴 이동
3. `Prometheus` 선택
4. URL에 `http://prometheus:9090` 입력
5. `Save & test` 실행

호스트 브라우저가 아니라 Grafana 컨테이너 내부에서 Prometheus에 접근하므로, 데이터 소스 URL은 `localhost:9090`이 아니라 서비스 이름인 `prometheus:9090`을 사용한다.

## 다른 환경에서의 사용 비활성화

현재 Prometheus/Grafana는 작성자 로컬 환경에서 먼저 사용하기 위해 Compose에 활성화되어 있다.
다른 개발 환경, 테스트 환경, 운영 환경에서 이 기능을 사용하지 않을 경우 아래와 같이 두 서비스 전체를 주석 처리한다.

```yaml
  # #### Monitoring services
  # ## 다른 환경에서는 아래 두 서비스 전체를 주석 처리한다.
  # prometheus:
  #   image: prom/prometheus:v3.13.5
  #   container_name: prometheus-container
  #   ports:
  #     - "9090:9090"
  #   networks:
  #     - default

  # grafana:
  #   image: grafana/grafana:13.0-slim
  #   container_name: grafana-container
  #   volumes:
  #     - grafana-data:/var/lib/grafana:rw
  #   ports:
  #     - "3000:3000"
  #   networks:
  #     - default
```

다른 환경의 Compose 파일이나 배포 방식에서 모니터링이 필요 없도록 빌드 파이프라인과 실행 스크립트도 함께 확인해야 한다.

## 볼륨 및 설정 주의사항

- Grafana 데이터는 명명 볼륨 `grafana-data`에 저장된다.
- Prometheus 설정을 직접 변경하거나 수집 대상을 추가하려면 별도의 `prometheus.yml`과 볼륨 마운트가 필요하다.
- 현재 Compose 구성만으로 모든 애플리케이션 메트릭을 자동 수집하는 것은 아니다.
- 애플리케이션에 Prometheus 메트릭 엔드포인트와 스크랩 설정이 없는 경우 Grafana에 데이터를 표시할 수 없다.
