# GitLab CI → Docker build & deploy (demo)

Prosty pipeline GitLab CI, który:
1) Buduje obraz Dockera aplikacji (dowolne źródło, np. `./app`).  
2) Loguje się do registry i **pushuje** obraz.  
3) Wdraża na zdalnym VPS (SSH), restartuje usługę i weryfikuje zdrowie.

## Architektura
- GitLab Runner (shared lub własny)
- Registry (GitLab Container Registry lub Docker Hub)
- VPS z Dockerem + Nginx jako reverse proxy
- Deploy via SSH + `docker compose` (opcjonalnie) lub `docker run`

## Zmienne w GitLab CI (Settings → CI/CD → Variables)
- `REGISTRY_USER`, `REGISTRY_PASSWORD`
- `REGISTRY_IMAGE` (np. `registry.gitlab.com/ORG/REPO/app`)
- `DEPLOY_HOST`, `DEPLOY_USER`
- `DEPLOY_SSH_KEY` (Protected, Masked, type: File lub wklej PEM)
- `DEPLOY_APP_NAME` (np. `demo-app`), `DEPLOY_PORT` (np. `8080`)

## Minimalny `.gitlab-ci.yml`
```yaml
stages: [build, deploy]

variables:
  DOCKER_DRIVER: overlay2

build:
  stage: build
  image: docker:24.0.5
  services: [docker:24.0.5-dind]
  script:
    - echo "$REGISTRY_PASSWORD" | docker login -u "$REGISTRY_USER" --password-stdin
    - docker build -t "$REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA" -t "$REGISTRY_IMAGE:latest" .
    - docker push "$REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"
    - docker push "$REGISTRY_IMAGE:latest"
  rules:
    - if: $CI_COMMIT_BRANCH

deploy:
  stage: deploy
  image: alpine:3.20
  needs: [build]
  before_script:
    - apk add --no-cache openssh-client docker-cli
    - mkdir -p ~/.ssh && chmod 700 ~/.ssh
    - echo "$DEPLOY_SSH_KEY" > ~/.ssh/id_rsa && chmod 600 ~/.ssh/id_rsa
    - ssh-keyscan -H "$DEPLOY_HOST" >> ~/.ssh/known_hosts
  script:
    - |
      ssh -i ~/.ssh/id_rsa "$DEPLOY_USER@$DEPLOY_HOST" '
        docker pull "$REGISTRY_IMAGE:latest" &&         docker stop "$DEPLOY_APP_NAME" || true &&         docker rm "$DEPLOY_APP_NAME" || true &&         docker run -d --name "$DEPLOY_APP_NAME" -p "$DEPLOY_PORT:80" "$REGISTRY_IMAGE:latest"
      '
  environment:
    name: staging
    url: http://$DEPLOY_HOST:$DEPLOY_PORT/
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

> Uwaga: jeśli używasz `docker compose`, zamień `docker run` na `docker compose up -d` i dołącz `compose.yml`.

## Szybki start
- Skopiuj `.gitlab-ci.yml` do repo.
- Ustaw zmienne w CI/CD.
- Dodaj prostą aplikację (np. `app/` + `Dockerfile` w katalogu głównym).
- Zrób `git push` → pipeline powinien zbudować i wdrożyć.

## TODO
- Healthcheck po wdrożeniu (curl /status)
- Artefakty z logami
- Środowisko `production` i tagi
