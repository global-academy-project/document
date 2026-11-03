# Docker with GitHub Actions workflow

# 1. what is Build and Buildx?

#### Build

```yml
- name: Set up Docker Build
  uses: docker/setup-buildx-action@v4.4.1
```


#### Buildx

```yml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v4.4.1
```

# 2. Builded for create image tag manage