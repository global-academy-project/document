# 1. Database

> [!NOTE]
> * **DB 영역은 Dockerfile에서 이미지를 직접적인 빌드를 거치지 않고 Docker Compose를 통해 자동으로 Container까지 설정을 진행해준다. 따라서 DB는 별도의 Installation Guide를 작성하지 않음에 따라 향후 SQL Qurey 파일로 CURD를 어떻게 구성되는지 추가할 예정이다.**

```yml
# docker-compose.yaml

<...>
  # PostgreSQL database service
  database:
    image: postgres:17-alpine3.23
    environment:
      # Required env variables for PostgreSQL
      POSTGRES_HOST: {YOUR_POSTGRESQL_HOST}
      POSTGRES_PORT: {BIND_POSTGRESQL_PORT}
      POSTGRES_DB: {YOUR_POSTGRESQL_DB_NAME}
      POSTGRES_USER: {YOUR_POSTGRESQL_USER}
      POSTGRES_PASSWORD: {YOUR_POSTGRESQL_PS}
    # Bind the container port to the host port (50002)
    ports:
      - "POSTGRES_PORT:50002"
<...>
```