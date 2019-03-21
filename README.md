# 1. Install docker server
- To install the latest docker release just execute the following curl command.
```
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```
- Check docker version
```
docker -v
```
- Enable docker service
```
systemctl enable docker.service
```
- Enable docker remote
```
- Edit file docker.service
vim /usr/lib/systemd/system/docker.service
- Change line 14 
ExecStart=/usr/bin/dockerd -H unix:///var/run/docker.sock -H tcp://0.0.0.0:2375
```
- Restart docker service
```
systemctl daemon-reload
systemctl restart docker.service
```
---
# 2. Setting project CI/CD
## a. External docker
- Setting gradle
``` gradle
jib {
    from {
        image = 'openjdk:alpine'
    }
    to {
        image = "docker.gnt-global.com/gx/spring-cicd-prod"
        auth {
            username = "DOCKER_USER"
            password = "DOCKER_PASSWORD"
        }
        tags = ["latest", "0.0.1-SNAPSHOT"]
    }
    container {
        useCurrentTimestamp = true
        ports = ['9000']
        mainClass = "com.gnt.demo.springcicd.SpringCicdApplication"
    }
}
```
- Add file .gitlab-ci.yml to project
```yml
image: docker:latest

services:
  - docker:dind
# 2 stages build and deploy
stages:
  - build
  - deploy
# master: Deploy, if push code branch develop
build-develop:
  stage: build
  image: openjdk:8-jdk-alpine
  cache:
    paths:
      - .gradle/wrapper
      - .gradle/caches
      - /cache

  script:
    - ./gradlew -g /cache jib
  only:
    - master
deploy-master:
  stage: deploy
  script:
    - export DOCKER_HOST=$CI_DOCKER_HOST
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker container stop  spring-cicd || true
    - docker container rm spring-cicd || true
    - docker pull docker.gnt-global.com/gx/spring-cicd-prod:latest
    - docker run -d -p 9000:9000 --name spring-cicd  docker.gnt-global.com/gx/spring-cicd-prod:latest
```
- Setting Environment variables on gitlab(settings->CI/CD)
```yml
    key: CI_DOCKER_HOST
    value: tcp://ip server:2375
```
```yml
    key: CI_REGISTRY
    value: docker.gnt-global.com
```
```yml
    key: CI_REGISTRY_USER
    value: USER_NAME
```
```yml
    key: CI_REGISTRY_PASSWORD
    value: PASSWORD_REGISTRY
```

## b. Registy Gitlab
- Setting gradle
```gradle
jib {
    from {
        image = 'openjdk:alpine'
    }
    to {
        image = System.getenv("CI_REGISTRY_IMAGE")+"/prod"
        tags = ["latest", "0.0.1-SNAPSHOT"]
        auth{
            username =System.getenv("CI_REGISTRY_USER")
            password= System.getenv("CI_REGISTRY_PASSWORD")
        }
    }
    container {
        useCurrentTimestamp = true
        ports = ['9000']
        mainClass = "com.gnt.demo.springcicd.SpringCicdApplication"
    }
}

```
- Add file .gitlab-ci.yml to project
```yml
image: docker:latest

services:
  - docker:dind
stages:
  - build
  - deploy

build-master:
  stage: build
  image: openjdk:8-jdk-alpine
  cache:
    paths:
      - .gradle/wrapper
      - .gradle/caches
      - /cache
  script:
    - ./gradlew -g /cache jib
  only:
    - master
deploy-master:
  stage: deploy
  script:
    - export DOCKER_HOST=$CI_DOCKER_HOST
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker container stop  spring-cicd || true
    - docker container rm spring-cicd || true
    - docker pull $CI_REGISTRY_IMAGE/prod:latest
    - docker run -p 9000:9000 --name spring-cicd -d $CI_REGISTRY_IMAGE/prod:latest

```
- Setting Environment variables on gitlab(settings->CI/CD)
```yml
    key: CI_DOCKER_HOST
    value: tcp://ip server:2375
```
